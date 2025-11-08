# Příklady rozšíření projektu BAFI

Tento dokument obsahuje konkrétní implementační návrhy pro rozšíření funkcionality projektu BAFI.

## 1. HTTP Server Mode

### Implementace REST API

**Nový soubor: `server.go`**

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "github.com/gorilla/mux"
)

type TransformRequest struct {
    Data         interface{} `json:"data"`
    InputFormat  string      `json:"input_format"`
    Template     string      `json:"template"`
    OutputFormat string      `json:"output_format,omitempty"`
}

type TransformResponse struct {
    Result string `json:"result"`
    Error  string `json:"error,omitempty"`
}

func startServer(port string) error {
    r := mux.NewRouter()

    r.HandleFunc("/transform", handleTransform).Methods("POST")
    r.HandleFunc("/health", handleHealth).Methods("GET")
    r.HandleFunc("/formats", handleFormats).Methods("GET")

    fmt.Printf("Starting BAFI server on port %s\n", port)
    return http.ListenAndServe(":"+port, r)
}

func handleTransform(w http.ResponseWriter, r *http.Request) {
    var req TransformRequest

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
        return
    }

    // Convert request to internal format and process
    // ... implementation

    resp := TransformResponse{
        Result: "transformed data",
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

func handleHealth(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func handleFormats(w http.ResponseWriter, r *http.Request) {
    formats := []string{"json", "bson", "yaml", "csv", "xml", "mt940"}
    json.NewEncoder(w).Encode(map[string]interface{}{
        "input_formats": formats,
        "template_functions": getTemplateFunctionNames(),
    })
}

func writeError(w http.ResponseWriter, code int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(TransformResponse{Error: message})
}
```

**Použití:**
```bash
# Start server
bafi serve -p 8080

# Use API
curl -X POST http://localhost:8080/transform \
  -H "Content-Type: application/json" \
  -d '{
    "data": {"name": "John", "age": 30},
    "input_format": "json",
    "template": "Name: {{.name}}, Age: {{.age}}"
  }'
```

## 2. Watch Mode - Automatické zpracování

**Nový soubor: `watcher.go`**

```go
package main

import (
    "log"
    "path/filepath"
    "time"

    "github.com/fsnotify/fsnotify"
)

type WatchConfig struct {
    InputPath    string
    Template     string
    OutputPath   string
    InputFormat  string
    Debounce     time.Duration
}

func startWatcher(config WatchConfig) error {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        return err
    }
    defer watcher.Close()

    done := make(chan bool)

    // Debounce timer to avoid processing multiple events
    var debounceTimer *time.Timer

    go func() {
        for {
            select {
            case event, ok := <-watcher.Events:
                if !ok {
                    return
                }

                if event.Op&fsnotify.Write == fsnotify.Write {
                    log.Println("Modified file:", event.Name)

                    // Reset debounce timer
                    if debounceTimer != nil {
                        debounceTimer.Stop()
                    }

                    debounceTimer = time.AfterFunc(config.Debounce, func() {
                        processFile(event.Name, config)
                    })
                }

            case err, ok := <-watcher.Errors:
                if !ok {
                    return
                }
                log.Println("Error:", err)
            }
        }
    }()

    inputDir := filepath.Dir(config.InputPath)
    err = watcher.Add(inputDir)
    if err != nil {
        return err
    }

    log.Printf("Watching %s for changes...\n", config.InputPath)
    <-done
    return nil
}

func processFile(filename string, config WatchConfig) {
    log.Printf("Processing %s...\n", filename)

    // Build params
    params := tParams{
        inputFile:    &filename,
        outputFile:   &config.OutputPath,
        textTemplate: &config.Template,
        inputFormat:  &config.InputFormat,
    }

    if err := processTemplate(params); err != nil {
        log.Printf("Error processing: %v\n", err)
        return
    }

    log.Printf("Successfully processed %s -> %s\n", filename, config.OutputPath)
}
```

**Použití:**
```bash
# Watch and auto-process on change
bafi watch -i data.json -t template.tmpl -o output.txt

# With custom debounce (default 500ms)
bafi watch -i data.json -t template.tmpl -o output.txt --debounce 1s
```

## 3. Batch Processing

**Nový soubor: `batch.go`**

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
    "sync"
)

type BatchConfig struct {
    InputPattern  string
    Template      string
    OutputDir     string
    InputFormat   string
    Parallel      int
    Verbose       bool
}

func processBatch(config BatchConfig) error {
    // Find all matching files
    matches, err := filepath.Glob(config.InputPattern)
    if err != nil {
        return fmt.Errorf("glob error: %w", err)
    }

    if len(matches) == 0 {
        return fmt.Errorf("no files matching pattern: %s", config.InputPattern)
    }

    // Create output directory if needed
    if err := os.MkdirAll(config.OutputDir, 0755); err != nil {
        return fmt.Errorf("create output dir: %w", err)
    }

    // Process files in parallel
    semaphore := make(chan struct{}, config.Parallel)
    var wg sync.WaitGroup
    errors := make(chan error, len(matches))

    for _, inputFile := range matches {
        wg.Add(1)
        semaphore <- struct{}{} // Acquire

        go func(input string) {
            defer wg.Done()
            defer func() { <-semaphore }() // Release

            if err := processOneFile(input, config); err != nil {
                errors <- fmt.Errorf("%s: %w", input, err)
            } else if config.Verbose {
                fmt.Printf("✓ Processed: %s\n", input)
            }
        }(inputFile)
    }

    wg.Wait()
    close(errors)

    // Collect errors
    var errs []error
    for err := range errors {
        errs = append(errs, err)
    }

    if len(errs) > 0 {
        fmt.Printf("Completed with %d errors:\n", len(errs))
        for _, err := range errs {
            fmt.Printf("  - %v\n", err)
        }
        return fmt.Errorf("batch processing had errors")
    }

    fmt.Printf("Successfully processed %d files\n", len(matches))
    return nil
}

func processOneFile(inputFile string, config BatchConfig) error {
    baseName := filepath.Base(inputFile)
    ext := filepath.Ext(baseName)
    nameWithoutExt := baseName[:len(baseName)-len(ext)]
    outputFile := filepath.Join(config.OutputDir, nameWithoutExt+".out")

    params := tParams{
        inputFile:    &inputFile,
        outputFile:   &outputFile,
        textTemplate: &config.Template,
        inputFormat:  &config.InputFormat,
    }

    return processTemplate(params)
}
```

**Použití:**
```bash
# Process all JSON files in directory
bafi batch -i "./data/*.json" -t template.tmpl -o ./output/

# With parallelism control
bafi batch -i "./data/*.json" -t template.tmpl -o ./output/ --parallel 4

# Verbose mode
bafi batch -i "./data/*.json" -t template.tmpl -o ./output/ -v
```

## 4. Schema Generation

**Nový soubor: `schema.go`**

```go
package main

import (
    "encoding/json"
    "fmt"
    "reflect"
)

type JSONSchema struct {
    Schema      string                 `json:"$schema"`
    Type        string                 `json:"type"`
    Properties  map[string]interface{} `json:"properties,omitempty"`
    Items       interface{}            `json:"items,omitempty"`
    Required    []string               `json:"required,omitempty"`
    Description string                 `json:"description,omitempty"`
}

func generateSchema(data interface{}) (*JSONSchema, error) {
    schema := &JSONSchema{
        Schema: "http://json-schema.org/draft-07/schema#",
    }

    fillSchemaFromValue(schema, reflect.ValueOf(data))
    return schema, nil
}

func fillSchemaFromValue(schema *JSONSchema, v reflect.Value) {
    switch v.Kind() {
    case reflect.Map:
        schema.Type = "object"
        schema.Properties = make(map[string]interface{})

        for _, key := range v.MapKeys() {
            keyStr := fmt.Sprint(key.Interface())
            val := v.MapIndex(key)

            propSchema := &JSONSchema{}
            fillSchemaFromValue(propSchema, val)
            schema.Properties[keyStr] = propSchema
        }

    case reflect.Slice, reflect.Array:
        schema.Type = "array"
        if v.Len() > 0 {
            itemSchema := &JSONSchema{}
            fillSchemaFromValue(itemSchema, v.Index(0))
            schema.Items = itemSchema
        }

    case reflect.String:
        schema.Type = "string"

    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64,
         reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        schema.Type = "integer"

    case reflect.Float32, reflect.Float64:
        schema.Type = "number"

    case reflect.Bool:
        schema.Type = "boolean"

    default:
        schema.Type = "null"
    }
}

func schemaCommand(inputFile, outputFile, inputFormat string) error {
    // Read and parse input
    data, _, err := getInputData(&inputFile)
    if err != nil {
        return err
    }

    params := tParams{
        inputFormat: &inputFormat,
    }

    mapData, err := mapInputData(data, params)
    if err != nil {
        return err
    }

    // Generate schema
    schema, err := generateSchema(mapData)
    if err != nil {
        return err
    }

    // Write schema
    schemaJSON, err := json.MarshalIndent(schema, "", "  ")
    if err != nil {
        return err
    }

    if outputFile == "" {
        fmt.Println(string(schemaJSON))
    } else {
        if err := os.WriteFile(outputFile, schemaJSON, 0644); err != nil {
            return err
        }
    }

    return nil
}
```

**Použití:**
```bash
# Generate JSON Schema
bafi schema generate -i data.json -o schema.json

# Validate against schema
bafi schema validate -i data.json -s schema.json
```

## 5. Vylepšené Template funkce

**Přidat do functions.go:**

```go
// hash - compute hash of string
func hash(algorithm string, input string) string {
    var h hash.Hash

    switch strings.ToLower(algorithm) {
    case "md5":
        h = md5.New()
    case "sha1":
        h = sha1.New()
    case "sha256":
        h = sha256.New()
    case "sha512":
        h = sha512.New()
    default:
        return "err: unknown hash algorithm"
    }

    h.Write([]byte(input))
    return hex.EncodeToString(h.Sum(nil))
}

// jsonPath - extract value using JSONPath
func jsonPath(path string, data interface{}) interface{} {
    // Implementation using github.com/tidwall/gjson
    jsonData, err := json.Marshal(data)
    if err != nil {
        return nil
    }

    result := gjson.GetBytes(jsonData, path)
    return result.Value()
}

// filter - filter array by condition
func filter(arr interface{}, key string, op string, value interface{}) []interface{} {
    // Implementation for filtering arrays
    // e.g. {{filter .items "age" ">" 18}}
    // ... implementation
    return nil
}

// unique - get unique values from array
func unique(arr interface{}) []interface{} {
    seen := make(map[interface{}]bool)
    result := []interface{}{}

    v := reflect.ValueOf(arr)
    if v.Kind() != reflect.Slice && v.Kind() != reflect.Array {
        return result
    }

    for i := 0; i < v.Len(); i++ {
        item := v.Index(i).Interface()
        if !seen[item] {
            seen[item] = true
            result = append(result, item)
        }
    }

    return result
}

// sort - sort array
func sortArray(arr interface{}, key string) []interface{} {
    // Implementation for sorting
    // ... implementation
    return nil
}

// groupBy - group array by key
func groupBy(arr interface{}, key string) map[string][]interface{} {
    // Implementation for grouping
    // ... implementation
    return nil
}

// encrypt/decrypt - AES encryption
func encrypt(plaintext, key string) string {
    // Implementation using AES
    return ""
}

func decrypt(ciphertext, key string) string {
    // Implementation using AES
    return ""
}

// httpGet - make HTTP GET request
func httpGet(url string) string {
    resp, err := http.Get(url)
    if err != nil {
        return fmt.Sprintf("err: %s", err.Error())
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return fmt.Sprintf("err: %s", err.Error())
    }

    return string(body)
}
```

**Použití v template:**
```go
// Hash
{{hash "sha256" .password}}

// JSONPath
{{jsonPath "users.0.name" .}}

// Filter
{{range filter .users "age" ">" 18}}
  {{.name}} is adult
{{end}}

// Unique
{{unique .tags}}

// Sort
{{range sort .items "price"}}
  {{.name}}: ${{.price}}
{{end}}

// HTTP
{{httpGet "https://api.example.com/data"}}
```

## 6. Plugin System

**Nový soubor: `plugin.go`**

```go
package main

import (
    "fmt"
    "plugin"
    "path/filepath"
)

type Parser interface {
    Name() string
    Parse(data []byte) (interface{}, error)
    Extensions() []string
}

type Formatter interface {
    Name() string
    Format(data interface{}) ([]byte, error)
}

var (
    customParsers    = make(map[string]Parser)
    customFormatters = make(map[string]Formatter)
)

func loadPlugins(pluginDir string) error {
    plugins, err := filepath.Glob(filepath.Join(pluginDir, "*.so"))
    if err != nil {
        return err
    }

    for _, pluginPath := range plugins {
        p, err := plugin.Open(pluginPath)
        if err != nil {
            fmt.Printf("Warning: failed to load plugin %s: %v\n", pluginPath, err)
            continue
        }

        // Try to load parser
        if parserSym, err := p.Lookup("Parser"); err == nil {
            if parser, ok := parserSym.(Parser); ok {
                customParsers[parser.Name()] = parser
                fmt.Printf("Loaded parser plugin: %s\n", parser.Name())
            }
        }

        // Try to load formatter
        if formatterSym, err := p.Lookup("Formatter"); err == nil {
            if formatter, ok := formatterSym.(Formatter); ok {
                customFormatters[formatter.Name()] = formatter
                fmt.Printf("Loaded formatter plugin: %s\n", formatter.Name())
            }
        }
    }

    return nil
}
```

**Example plugin: `plugins/toml/toml.go`**

```go
package main

import (
    "github.com/BurntSushi/toml"
)

type TOMLParser struct{}

func (p *TOMLParser) Name() string {
    return "toml"
}

func (p *TOMLParser) Extensions() []string {
    return []string{".toml"}
}

func (p *TOMLParser) Parse(data []byte) (interface{}, error) {
    var result interface{}
    if err := toml.Unmarshal(data, &result); err != nil {
        return nil, err
    }
    return result, nil
}

var Parser TOMLParser
```

**Build plugin:**
```bash
go build -buildmode=plugin -o toml.so toml.go
```

## 7. Konfigurace pomocí souboru

**Nový soubor: `config.go`**

```go
package main

import (
    "os"
    "gopkg.in/yaml.v3"
)

type Config struct {
    Server struct {
        Port    int  `yaml:"port"`
        Enabled bool `yaml:"enabled"`
    } `yaml:"server"`

    Defaults struct {
        InputFormat  string `yaml:"input_format"`
        OutputFormat string `yaml:"output_format"`
        Template     string `yaml:"template"`
    } `yaml:"defaults"`

    OpenAI struct {
        APIKey string `yaml:"api_key"`
        Model  string `yaml:"model"`
    } `yaml:"openai"`

    Lua struct {
        FunctionsPath string `yaml:"functions_path"`
    } `yaml:"lua"`

    Limits struct {
        MaxFileSize int64 `yaml:"max_file_size"`
        Timeout     int   `yaml:"timeout_seconds"`
    } `yaml:"limits"`
}

func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, err
    }

    return &config, nil
}
```

**Example config file: `.bafi.yaml`**

```yaml
server:
  port: 8080
  enabled: false

defaults:
  input_format: json
  template: "{{toJSON .}}"

openai:
  api_key: ${OPENAI_API_KEY}
  model: gpt-4

lua:
  functions_path: ./lua/functions.lua

limits:
  max_file_size: 104857600  # 100MB
  timeout_seconds: 300
```

## Závěr

Tyto rozšíření mohou být implementována postupně podle priorit. Každé rozšíření je navrženo tak, aby:

1. Neporušilo zpětnou kompatibilitu
2. Bylo volitelné (uživatelé stávající funkcionality nejsou ovlivněni)
3. Mělo jasné use case
4. Bylo dobře otestované

Doporučené pořadí implementace:
1. Konfigurace pomocí souboru (základ pro další features)
2. Vylepšené template funkce (okamžitá hodnota pro uživatele)
3. Batch processing (častý use case)
4. Watch mode (užitečné pro development)
5. HTTP server mode (opens new use cases)
6. Schema generation (advanced feature)
7. Plugin system (pro long-term extensibility)
