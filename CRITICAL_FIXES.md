# Kritické opravy - Implementační příručka

Tento dokument obsahuje konkrétní kód pro opravu kritických problémů nalezených v projektu BAFI.

## 1. Oprava dělení nulou

### functions.go - funkce div()

**Před:**
```go
// div divide
func div(a, b interface{}) int64 { return toInt64(a) / toInt64(b) }
```

**Po:**
```go
// div divide - returns 0 if divisor is zero
func div(a, b interface{}) int64 {
    divisor := toInt64(b)
    if divisor == 0 {
        return 0
    }
    return toInt64(a) / divisor
}
```

### functions.go - funkce mod()

**Před:**
```go
// mod modulo
func mod(a, b interface{}) int64 { return toInt64(a) % toInt64(b) }
```

**Po:**
```go
// mod modulo - returns 0 if divisor is zero
func mod(a, b interface{}) int64 {
    divisor := toInt64(b)
    if divisor == 0 {
        return 0
    }
    return toInt64(a) % divisor
}
```

### functions.go - funkce divf()

**Před:**
```go
// divide float
func divf(a interface{}, v ...interface{}) float64 {
    return execDecimalOp(a, v, func(d1, d2 decimal.Decimal) decimal.Decimal { return d1.Div(d2) })
}
```

**Po:**
```go
// divide float
func divf(a interface{}, v ...interface{}) float64 {
    return execDecimalOp(a, v, func(d1, d2 decimal.Decimal) decimal.Decimal {
        if d2.IsZero() {
            return decimal.Zero
        }
        return d1.Div(d2)
    })
}
```

## 2. Odstranění deprecated rand.Seed

### main.go - funkce init()

**Před:**
```go
func init() {
    rand.Seed(time.Now().UTC().UnixNano())
    if _, err := os.Stat("./lua/functions.lua"); !os.IsNotExist(err) {
        luaData = lua.NewState()
        if err := luaData.DoFile("./lua/functions.lua"); err != nil {
            log.Fatal("loadLuaFunctions", err.Error())
        }
    }
}
```

**Po:**
```go
func init() {
    // rand.Seed is no longer needed in Go 1.20+
    // The default seed is now random and automatically set
    if _, err := os.Stat("./lua/functions.lua"); !os.IsNotExist(err) {
        luaData = lua.NewState()
        if err := luaData.DoFile("./lua/functions.lua"); err != nil {
            log.Fatal("loadLuaFunctions", err.Error())
        }
    }
}
```

### functions.go - funkce randInt()

**Před:**
```go
// randInt returns random integer in defined range {{randInt min max}} e.g. {{randInt 1 10}}
func randInt(min, max int) int {
    if max <= min {
        return min
    }
    return rand.Intn(max-min+1) + min
}
```

**Po (pro Go 1.22+):**
```go
// randInt returns random integer in defined range {{randInt min max}} e.g. {{randInt 1 10}}
func randInt(min, max int) int {
    if max < min {
        return min
    }
    if max == min {
        return min
    }
    // Go 1.22+ has rand.IntN which is automatically seeded
    return rand.IntN(max-min+1) + min
}
```

**Nebo (pro kompatibilitu s Go 1.20-1.21):**
```go
// randInt returns random integer in defined range {{randInt min max}} e.g. {{randInt 1 10}}
func randInt(min, max int) int {
    if max <= min {
        return min
    }
    // rand.Intn is automatically seeded since Go 1.20
    return rand.Intn(max-min+1) + min
}
```

## 3. Oprava race condition s Lua state

### main.go - globální proměnné a init

**Před:**
```go
var (
    luaData *lua.LState
)

func init() {
    rand.Seed(time.Now().UTC().UnixNano())
    if _, err := os.Stat("./lua/functions.lua"); !os.IsNotExist(err) {
        luaData = lua.NewState()
        if err := luaData.DoFile("./lua/functions.lua"); err != nil {
            log.Fatal("loadLuaFunctions", err.Error())
        }
    }
}
```

**Po:**
```go
var (
    luaPool *sync.Pool
    luaFile string
)

func init() {
    luaFile = "./lua/functions.lua"
    if _, err := os.Stat(luaFile); !os.IsNotExist(err) {
        // Initialize pool of Lua states for thread-safe concurrent use
        luaPool = &sync.Pool{
            New: func() interface{} {
                L := lua.NewState()
                if err := L.DoFile(luaFile); err != nil {
                    log.Printf("Warning: Failed to load Lua file: %v", err)
                    return nil
                }
                return L
            },
        }
    }
}
```

### functions.go - funkce luaF()

**Před:**
```go
func luaF(i ...interface{}) string {
    if luaData == nil {
        return "error: ./lua/functions.lua file missing)"
    }
    strData, err := json.Marshal(i[1:])
    if err != nil {
        return fmt.Sprintf("luaInputError: %s\r\n", err.Error())
    }
    if err := luaData.CallByParam(
        lua.P{Fn: luaData.GetGlobal(i[0].(string)), NRet: 1, Protect: true}, lua.LString(string(strData))); err != nil {
        return fmt.Sprintf("luaError: %s\r\n", err.Error())
    }
    if str, ok := luaData.Get(-1).(lua.LString); ok {
        luaData.Pop(1)
        return str.String()
    }
    return "luaError: getResult"
}
```

**Po:**
```go
func luaF(i ...interface{}) string {
    if luaPool == nil {
        return "error: ./lua/functions.lua file missing)"
    }

    // Get Lua state from pool
    L := luaPool.Get().(*lua.LState)
    if L == nil {
        return "error: failed to initialize Lua state"
    }
    defer luaPool.Put(L)

    strData, err := json.Marshal(i[1:])
    if err != nil {
        return fmt.Sprintf("luaInputError: %s\r\n", err.Error())
    }
    if err := L.CallByParam(
        lua.P{Fn: L.GetGlobal(i[0].(string)), NRet: 1, Protect: true}, lua.LString(string(strData))); err != nil {
        return fmt.Sprintf("luaError: %s\r\n", err.Error())
    }
    if str, ok := L.Get(-1).(lua.LString); ok {
        L.Pop(1)
        return str.String()
    }
    return "luaError: getResult"
}
```

### main.go - main()

**Před:**
```go
func main() {
    // ... existing code ...
    if err := processTemplate(params); err != nil {
        log.Fatal(err.Error())
    }
    if luaData != nil {
        luaData.Close()
    }
}
```

**Po:**
```go
func main() {
    // ... existing code ...
    if err := processTemplate(params); err != nil {
        log.Fatal(err.Error())
    }
    // Pool cleanup is handled by GC, but we can clear it explicitly if needed
    if luaPool != nil {
        // Note: sync.Pool doesn't need explicit cleanup
        // Items will be garbage collected when no longer needed
    }
}
```

## 4. Zabezpečení ChatGPT API klíče

### main.go - tParams a flag parsing

**Před:**
```go
chatGPTkey: flag.String("gk", "", "OpenAI API key"),
```

**Po:**
```go
chatGPTkey: flag.String("gk", os.Getenv("OPENAI_API_KEY"), "OpenAI API key (can also be set via OPENAI_API_KEY env var)"),
```

**Doporučené použití:**
```bash
# Místo
bafi -i data.json -t template.tmpl -gk sk-xxxxxxxxxxxxx -gq "What is this?"

# Použít
export OPENAI_API_KEY=sk-xxxxxxxxxxxxx
bafi -i data.json -t template.tmpl -gq "What is this?"
```

## 5. Oprava addSubstring

### functions.go - funkce addSubstring()

**Před:**
```go
func addSubstring(s string, ss string, pos interface{}) string {
    if toInt(pos) >= len(s) || -toInt(pos) >= len(s) {
        return "err:substringOutOfRange"
    }
    switch x := toInt(pos); {
    case x == 0:
        return s
    case x > 0:
        return fmt.Sprintf("%s%s%s", s[:len(s)-x], ss, s[len(s)-x:])
    case x < 0:
        return fmt.Sprintf("%s%s%s", s[:-x], ss, s[-x:])  // CHYBA: neplatná syntax
    default:
        return "inputError"
    }
}
```

**Po:**
```go
func addSubstring(s string, ss string, pos interface{}) string {
    p := toInt(pos)
    sLen := len(s)

    // Position 0 means no insertion
    if p == 0 {
        return s
    }

    // Positive position: count from end
    if p > 0 {
        if p > sLen {
            return "err:substringOutOfRange"
        }
        return s[:sLen-p] + ss + s[sLen-p:]
    }

    // Negative position: count from start
    absP := -p
    if absP > sLen {
        return "err:substringOutOfRange"
    }
    return s[:absP] + ss + s[absP:]
}
```

## 6. Oprava mustArray

### functions.go - funkce mustArray()

**Před:**
```go
func mustArray(v interface{}) []interface{} {
    if v == nil {
        return nil
    }
    if a, ok := v.([]interface{}); ok {
        return a
    }
    return []interface{}{v}
}
```

**Po:**
```go
func mustArray(v interface{}) []interface{} {
    if v == nil {
        return []interface{}{} // return empty slice instead of nil
    }
    if a, ok := v.([]interface{}); ok {
        return a
    }
    return []interface{}{v}
}
```

## 7. Oprava prázdného template stringu

### main.go - funkce readTemplate()

**Před:**
```go
func readTemplate(textTemplate string) ([]byte, error) {
    var templateFile []byte
    var err error
    if textTemplate[:1] == "?" {
        templateFile = []byte(textTemplate[1:])
    } else {
        templateFile, err = os.ReadFile(textTemplate)
        if err != nil {
            return nil, fmt.Errorf("readFile: %s", err.Error())
        }
    }
    return templateFile, nil
}
```

**Po:**
```go
func readTemplate(textTemplate string) ([]byte, error) {
    if textTemplate == "" {
        return nil, fmt.Errorf("template string is empty")
    }

    var templateFile []byte
    var err error
    if textTemplate[0] == '?' {
        if len(textTemplate) == 1 {
            return nil, fmt.Errorf("inline template is empty")
        }
        templateFile = []byte(textTemplate[1:])
    } else {
        templateFile, err = os.ReadFile(textTemplate)
        if err != nil {
            return nil, fmt.Errorf("readFile: %s", err.Error())
        }
    }
    return templateFile, nil
}
```

## 8. Vylepšení CSV error handling

### main.go - mapInputData() pro CSV

**Před:**
```go
case "csv":
    var mapData []map[string]interface{}
    r := csv.NewReader(bytes.NewReader(data))
    r.Comma = prepareDelimiter(*params.inputDelimiter)
    lines, err := r.ReadAll()
    if err != nil {
        return nil, fmt.Errorf("mapCSV: %s", err.Error())
    }
    if len(lines) == 0 {
        return nil, fmt.Errorf("mapCSV: CSV has no rows")
    }
    mapData = make([]map[string]interface{}, len(lines[1:]))
```

**Po:**
```go
case "csv":
    var mapData []map[string]interface{}
    r := csv.NewReader(bytes.NewReader(data))
    r.Comma = prepareDelimiter(*params.inputDelimiter)
    lines, err := r.ReadAll()
    if err != nil {
        return nil, fmt.Errorf("mapCSV: %s", err.Error())
    }
    if len(lines) == 0 {
        return nil, fmt.Errorf("mapCSV: CSV has no rows")
    }
    if len(lines) < 2 {
        return nil, fmt.Errorf("mapCSV: CSV has no data rows (only header present)")
    }
    mapData = make([]map[string]interface{}, len(lines[1:]))
```

## Testování oprav

Po implementaci těchto oprav spusťte:

```bash
# Format kódu
gofmt -w .

# Spustit testy
go test -v ./...

# Spustit go vet
go vet ./...

# Build
go build -v

# Test s edge cases
echo '{"a": 10, "b": 0}' | ./bafi -f json -t '?{{div .a .b}}'  # Mělo by vrátit 0, ne panic
```

## Kontrolní seznam

- [ ] Opravit dělení nulou (div, mod, divf)
- [ ] Odstranit deprecated rand.Seed
- [ ] Implementovat Lua pool pro thread-safety
- [ ] Přidat support pro OPENAI_API_KEY env var
- [ ] Opravit addSubstring
- [ ] Opravit mustArray
- [ ] Opravit readTemplate validation
- [ ] Vylepšit CSV error handling
- [ ] Spustit všechny testy
- [ ] Aktualizovat dokumentaci
