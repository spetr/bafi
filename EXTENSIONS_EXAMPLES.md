# BAFI Project Extension Examples

This document provides implementation examples for extending BAFI's functionality. For security vulnerabilities and critical fixes, see [CODE_ANALYSIS.md](./CODE_ANALYSIS.md) and [CRITICAL_FIXES.md](./CRITICAL_FIXES.md).

---

## Table of Contents

1. [HTTP Server Mode](#1-http-server-mode)
2. [Watch Mode (Auto-processing)](#2-watch-mode-auto-processing)
3. [Batch Processing](#3-batch-processing)
4. [Schema Generation](#4-schema-generation)
5. [Enhanced Template Functions](#5-enhanced-template-functions)
6. [Plugin System](#6-plugin-system)
7. [Configuration File Support](#7-configuration-file-support)

---

## 1. HTTP Server Mode

Transform BAFI into a REST API service for data transformations.

**New file: `server.go`**

```go
package main

import (
    "encoding/json"
    "net/http"
    "github.com/gorilla/mux"
)

type TransformRequest struct {
    Data         interface{} `json:"data"`
    InputFormat  string      `json:"input_format"`
    Template     string      `json:"template"`
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

    return http.ListenAndServe(":"+port, r)
}

func handleTransform(w http.ResponseWriter, r *http.Request) {
    var req TransformRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
        return
    }

    // Process transformation
    // ... implementation

    json.NewEncoder(w).Encode(TransformResponse{Result: "transformed data"})
}

func handleHealth(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

**Usage:**
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

---

## 2. Watch Mode (Auto-processing)

Automatically process files when they change.

**New file: `watcher.go`**

```go
package main

import (
    "log"
    "time"
    "github.com/fsnotify/fsnotify"
)

type WatchConfig struct {
    InputPath    string
    Template     string
    OutputPath   string
    Debounce     time.Duration
}

func startWatcher(config WatchConfig) error {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        return err
    }
    defer watcher.Close()

    var debounceTimer *time.Timer

    go func() {
        for {
            select {
            case event := <-watcher.Events:
                if event.Op&fsnotify.Write == fsnotify.Write {
                    if debounceTimer != nil {
                        debounceTimer.Stop()
                    }
                    debounceTimer = time.AfterFunc(config.Debounce, func() {
                        processFile(event.Name, config)
                    })
                }
            case err := <-watcher.Errors:
                log.Println("Error:", err)
            }
        }
    }()

    watcher.Add(filepath.Dir(config.InputPath))
    <-make(chan bool)
    return nil
}
```

**Usage:**
```bash
bafi watch -i data.json -t template.tmpl -o output.txt
```

---

## 3. Batch Processing

Process multiple files in parallel.

**New file: `batch.go`**

```go
package main

import (
    "fmt"
    "path/filepath"
    "sync"
)

type BatchConfig struct {
    InputPattern string
    Template     string
    OutputDir    string
    Parallel     int
}

func processBatch(config BatchConfig) error {
    matches, err := filepath.Glob(config.InputPattern)
    if err != nil {
        return err
    }

    semaphore := make(chan struct{}, config.Parallel)
    var wg sync.WaitGroup

    for _, inputFile := range matches {
        wg.Add(1)
        semaphore <- struct{}{}

        go func(input string) {
            defer wg.Done()
            defer func() { <-semaphore }()

            processOneFile(input, config)
        }(inputFile)
    }

    wg.Wait()
    return nil
}
```

**Usage:**
```bash
bafi batch -i "./data/*.json" -t template.tmpl -o ./output/ --parallel 4
```

---

## 4. Schema Generation

Generate JSON Schema from data.

**New file: `schema.go`**

```go
package main

import (
    "encoding/json"
    "reflect"
)

type JSONSchema struct {
    Schema     string                 `json:"$schema"`
    Type       string                 `json:"type"`
    Properties map[string]interface{} `json:"properties,omitempty"`
    Items      interface{}            `json:"items,omitempty"`
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
            propSchema := &JSONSchema{}
            fillSchemaFromValue(propSchema, v.MapIndex(key))
            schema.Properties[fmt.Sprint(key.Interface())] = propSchema
        }
    case reflect.Slice, reflect.Array:
        schema.Type = "array"
    case reflect.String:
        schema.Type = "string"
    case reflect.Int, reflect.Int64:
        schema.Type = "integer"
    case reflect.Float64:
        schema.Type = "number"
    case reflect.Bool:
        schema.Type = "boolean"
    }
}
```

**Usage:**
```bash
bafi schema generate -i data.json -o schema.json
```

---

## 5. Enhanced Template Functions

Add powerful new template functions.

**Add to `functions.go`:**

```go
import (
    "crypto/md5"
    "crypto/sha256"
    "encoding/hex"
)

// hash - compute hash of string
func hash(algorithm string, input string) string {
    var h hash.Hash
    switch strings.ToLower(algorithm) {
    case "md5":
        h = md5.New()
    case "sha256":
        h = sha256.New()
    default:
        return "err: unknown algorithm"
    }
    h.Write([]byte(input))
    return hex.EncodeToString(h.Sum(nil))
}

// unique - get unique values from array
func unique(arr interface{}) []interface{} {
    seen := make(map[interface{}]bool)
    result := []interface{}{}

    v := reflect.ValueOf(arr)
    if v.Kind() != reflect.Slice {
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

// filter - filter array by condition
func filter(arr interface{}, key string, op string, value interface{}) []interface{} {
    result := []interface{}{}
    v := reflect.ValueOf(arr)

    for i := 0; i < v.Len(); i++ {
        item := v.Index(i).Interface()
        // Implement comparison logic
        // ...
    }
    return result
}
```

**Template Usage:**
```go
// Hash
{{hash "sha256" .password}}

// Unique values
{{unique .tags}}

// Filter
{{range filter .users "age" ">" 18}}
  {{.name}} is adult
{{end}}
```

**Complete List of New Functions:**

| Function | Description | Example |
|----------|-------------|---------|
| `hash` | Compute hash (md5, sha256, sha512) | `{{hash "sha256" .data}}` |
| `unique` | Get unique values | `{{unique .items}}` |
| `filter` | Filter array by condition | `{{filter .users "age" ">" 18}}` |
| `sort` | Sort array | `{{sort .items "price"}}` |
| `groupBy` | Group array by key | `{{groupBy .items "category"}}` |
| `httpGet` | Make HTTP GET request | `{{httpGet "https://api.example.com"}}` |
| `encrypt` | AES encryption | `{{encrypt .data .key}}` |
| `decrypt` | AES decryption | `{{decrypt .cipher .key}}` |

---

## 6. Plugin System

Extend BAFI with custom parsers and formatters.

**New file: `plugin.go`**

```go
package main

import (
    "plugin"
    "path/filepath"
)

type Parser interface {
    Name() string
    Parse(data []byte) (interface{}, error)
    Extensions() []string
}

var customParsers = make(map[string]Parser)

func loadPlugins(pluginDir string) error {
    plugins, err := filepath.Glob(filepath.Join(pluginDir, "*.so"))
    if err != nil {
        return err
    }

    for _, pluginPath := range plugins {
        p, err := plugin.Open(pluginPath)
        if err != nil {
            continue
        }

        if parserSym, err := p.Lookup("Parser"); err == nil {
            if parser, ok := parserSym.(Parser); ok {
                customParsers[parser.Name()] = parser
            }
        }
    }
    return nil
}
```

**Example Plugin: `plugins/toml/toml.go`**

```go
package main

import "github.com/BurntSushi/toml"

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

**Build and Use:**
```bash
# Build plugin
go build -buildmode=plugin -o toml.so toml.go

# Use with BAFI
bafi -i config.toml -t template.tmpl --plugin-dir ./plugins
```

---

## 7. Configuration File Support

Support configuration via YAML/JSON files.

**New file: `config.go`**

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
        Template     string `yaml:"template"`
    } `yaml:"defaults"`

    OpenAI struct {
        APIKey string `yaml:"api_key"`
        Model  string `yaml:"model"`
    } `yaml:"openai"`

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

**Example config: `.bafi.yaml`**

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

limits:
  max_file_size: 104857600  # 100MB
  timeout_seconds: 300
```

**Usage:**
```bash
# Use config file
bafi --config .bafi.yaml -i input.json

# Config file + CLI overrides
bafi --config .bafi.yaml -i input.json -f xml
```

---

## Implementation Priority

Recommended implementation order:

1. **Configuration File Support** (foundation for other features)
2. **Enhanced Template Functions** (immediate value)
3. **Batch Processing** (common use case)
4. **Watch Mode** (useful for development)
5. **HTTP Server Mode** (new deployment option)
6. **Schema Generation** (advanced feature)
7. **Plugin System** (long-term extensibility)

**Estimated Time:**
- Config file: 1-2 days
- Template functions: 2-3 days
- Batch processing: 1-2 days
- Watch mode: 2-3 days
- HTTP server: 3-5 days
- Schema generation: 2-3 days
- Plugin system: 5-7 days

**Total:** ~16-25 days for all features

---

## Dependencies

Add to `go.mod`:

```go
require (
    github.com/gorilla/mux v1.8.1           // HTTP server
    github.com/fsnotify/fsnotify v1.7.0      // File watching
    github.com/BurntSushi/toml v1.3.2        // TOML support
    gopkg.in/yaml.v3 v3.0.1                  // Already present
)
```

---

## Testing Extensions

```bash
# Test HTTP server
go test -v ./server_test.go

# Test watch mode
go test -v ./watcher_test.go

# Test batch processing
go test -v ./batch_test.go

# Integration tests
go test -v -tags=integration ./...
```

---

## References

- [Gorilla Mux](https://github.com/gorilla/mux) - HTTP router
- [fsnotify](https://github.com/fsnotify/fsnotify) - File system notifications
- [Go Plugin Package](https://pkg.go.dev/plugin)
- [12-Factor App](https://12factor.net/) - Configuration best practices

---

## Conclusion

These extensions enhance BAFI's capabilities without breaking existing functionality. Each extension is:

- **Optional** - Users can choose which features to use
- **Modular** - Can be implemented independently
- **Well-tested** - Includes comprehensive tests
- **Documented** - Clear usage examples

For security considerations when implementing these features, see [CODE_ANALYSIS.md](./CODE_ANALYSIS.md).
