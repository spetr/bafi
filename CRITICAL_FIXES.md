# Critical Fixes - Implementation Guide

This document contains ready-to-use code for fixing critical issues found in the BAFI project.

## Quick Reference

| Issue | File | Priority | Estimated Time |
|-------|------|----------|----------------|
| Division by zero | functions.go:108,111,139 | 🔴 HIGH | 15 min |
| Deprecated rand.Seed | main.go:48 | 🟡 MEDIUM | 10 min |
| Race condition (Lua) | main.go:31, functions.go:518 | 🔴 HIGH | 30 min |
| API key exposure | main.go:70 | 🔴 HIGH | 15 min |
| addSubstring bug | functions.go:324-337 | 🟠 MEDIUM | 20 min |
| mustArray returns nil | functions.go:498 | 🟡 MEDIUM | 5 min |
| Empty template check | main.go:320 | 🟡 MEDIUM | 10 min |
| CSV error handling | main.go:274 | 🟢 LOW | 10 min |

**Total Estimated Time: ~2 hours**

---

## 1. Fix Division by Zero

### functions.go - div() function

**Before:**
```go
// div divide
func div(a, b interface{}) int64 { return toInt64(a) / toInt64(b) }
```

**After:**
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

### functions.go - mod() function

**Before:**
```go
// mod modulo
func mod(a, b interface{}) int64 { return toInt64(a) % toInt64(b) }
```

**After:**
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

### functions.go - divf() function

**Before:**
```go
// divide float
func divf(a interface{}, v ...interface{}) float64 {
    return execDecimalOp(a, v, func(d1, d2 decimal.Decimal) decimal.Decimal { return d1.Div(d2) })
}
```

**After:**
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

---

## 2. Remove Deprecated rand.Seed

### main.go - init() function

**Before:**
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

**After:**
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

### functions.go - randInt() function

**For Go 1.22+:**
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

**Or for compatibility with Go 1.20-1.21:**
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

---

## 3. Fix Race Condition with Lua State

### main.go - global variables and init

**Before:**
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

**After:**
```go
import "sync"

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

### functions.go - luaF() function

**Before:**
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

**After:**
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

**Before:**
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

**After:**
```go
func main() {
    // ... existing code ...
    if err := processTemplate(params); err != nil {
        log.Fatal(err.Error())
    }
    // Pool cleanup is handled by GC
    // Items will be garbage collected when no longer needed
}
```

---

## 4. Secure ChatGPT API Key

### main.go - tParams and flag parsing

**Before:**
```go
chatGPTkey: flag.String("gk", "", "OpenAI API key"),
```

**After:**
```go
chatGPTkey: flag.String("gk", os.Getenv("OPENAI_API_KEY"), "OpenAI API key (can also be set via OPENAI_API_KEY env var)"),
```

**Recommended Usage:**
```bash
# Instead of:
bafi -i data.json -t template.tmpl -gk sk-xxxxxxxxxxxxx -gq "What is this?"

# Use:
export OPENAI_API_KEY=sk-xxxxxxxxxxxxx
bafi -i data.json -t template.tmpl -gq "What is this?"
```

**Additional Security (Optional):**

Create a configuration file loader:

```go
import "os"

const DefaultConfigPath = ".bafi/config.yaml"

type Config struct {
    OpenAI struct {
        APIKey string `yaml:"api_key"`
    } `yaml:"openai"`
}

func loadConfig() (*Config, error) {
    home, err := os.UserHomeDir()
    if err != nil {
        return nil, err
    }

    configPath := filepath.Join(home, DefaultConfigPath)
    // Load and parse config file
    // ...
}
```

---

## 5. Fix addSubstring

### functions.go - addSubstring() function

**Before:**
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
        return fmt.Sprintf("%s%s%s", s[:-x], ss, s[-x:])  // ERROR: invalid syntax
    default:
        return "inputError"
    }
}
```

**After:**
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

---

## 6. Fix mustArray

### functions.go - mustArray() function

**Before:**
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

**After:**
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

---

## 7. Fix Empty Template Validation

### main.go - readTemplate() function

**Before:**
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

**After:**
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

---

## 8. Improve CSV Error Handling

### main.go - mapInputData() for CSV

**Before:**
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

**After:**
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

---

## Testing the Fixes

After implementing these fixes, run:

```bash
# Format code
gofmt -w .

# Run tests
go test -v ./...

# Run go vet
go vet ./...

# Build
go build -v

# Test edge cases
echo '{"a": 10, "b": 0}' | ./bafi -f json -t '?{{div .a .b}}'  # Should return 0, not panic

# Test with race detector
go test -race ./...
```

---

## Checklist

After implementing all fixes:

- [ ] Fix division by zero (div, mod, divf)
- [ ] Remove deprecated rand.Seed
- [ ] Implement Lua pool for thread-safety
- [ ] Add support for OPENAI_API_KEY env var
- [ ] Fix addSubstring
- [ ] Fix mustArray
- [ ] Fix readTemplate validation
- [ ] Improve CSV error handling
- [ ] Run all tests
- [ ] Update documentation
- [ ] Code review
- [ ] Create release notes

---

## Implementation Order

Recommended order for implementing fixes:

1. **Division by zero** (highest priority, easiest fix)
2. **mustArray** (simple, quick fix)
3. **readTemplate** (simple validation)
4. **CSV error handling** (small change)
5. **Deprecated rand.Seed** (requires testing)
6. **addSubstring** (requires careful testing)
7. **API key** (requires documentation update)
8. **Lua pool** (most complex, requires thorough testing)

---

## Expected Results

After implementing all fixes:

- ✅ Zero panics in production
- ✅ Thread-safe Lua execution
- ✅ Better security for API keys
- ✅ Improved error messages
- ✅ More robust edge case handling
- ✅ No deprecated warnings

---

## References

- [Go Data Race Detector](https://go.dev/doc/articles/race_detector)
- [Go 1.20 Release Notes](https://go.dev/doc/go1.20)
- [OWASP Secrets Management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Effective Go](https://go.dev/doc/effective_go)
