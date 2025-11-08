# Testy pro kritické opravy

Tento dokument obsahuje testy, které by měly být přidány po implementaci kritických oprav.

## 1. Testy pro dělení nulou

**Přidat do functions_test.go:**

```go
// TestDivByZero tests division by zero handling
func TestDivByZero(t *testing.T) {
    // Test div function
    result := div(10, 0)
    if result != 0 {
        t.Errorf("div(10, 0) expected 0, got: %v", result)
    }

    result = div(10, 5)
    if result != 2 {
        t.Errorf("div(10, 5) expected 2, got: %v", result)
    }
}

// TestModByZero tests modulo by zero handling
func TestModByZero(t *testing.T) {
    result := mod(10, 0)
    if result != 0 {
        t.Errorf("mod(10, 0) expected 0, got: %v", result)
    }

    result = mod(10, 3)
    if result != 1 {
        t.Errorf("mod(10, 3) expected 1, got: %v", result)
    }
}

// TestDivfByZero tests float division by zero handling
func TestDivfByZero(t *testing.T) {
    result := divf(10.5, 0.0)
    if result != 0.0 {
        t.Errorf("divf(10.5, 0.0) expected 0.0, got: %v", result)
    }

    result = divf(10.5, 2.0)
    if result != 5.25 {
        t.Errorf("divf(10.5, 2.0) expected 5.25, got: %v", result)
    }

    // Test with decimal zero
    result = divf("100", "0")
    if result != 0.0 {
        t.Errorf("divf('100', '0') expected 0.0, got: %v", result)
    }
}

// TestDivInTemplate tests division in template
func TestDivInTemplate(t *testing.T) {
    // Should not panic
    err := runt(`{{ div 10 0 }}`, "0")
    if err != nil {
        t.Errorf("template division by zero failed: %v", err.Error())
    }

    err = runt(`{{ div 10 2 }}`, "5")
    if err != nil {
        t.Errorf("template division failed: %v", err.Error())
    }
}

// TestModInTemplate tests modulo in template
func TestModInTemplate(t *testing.T) {
    err := runt(`{{ mod 10 0 }}`, "0")
    if err != nil {
        t.Errorf("template modulo by zero failed: %v", err.Error())
    }

    err = runt(`{{ mod 10 3 }}`, "1")
    if err != nil {
        t.Errorf("template modulo failed: %v", err.Error())
    }
}

// TestDivfInTemplate tests float division in template
func TestDivfInTemplate(t *testing.T) {
    err := runt(`{{ divf 10.5 0 }}`, "0")
    if err != nil {
        t.Errorf("template float division by zero failed: %v", err.Error())
    }
}
```

## 2. Testy pro Lua pool (thread-safety)

**Přidat do functions_test.go:**

```go
// TestLuaConcurrent tests Lua function calls from multiple goroutines
func TestLuaConcurrent(t *testing.T) {
    if luaPool == nil {
        t.Skip("Lua not available")
    }

    const numGoroutines = 100
    const numCalls = 10

    var wg sync.WaitGroup
    errors := make(chan error, numGoroutines*numCalls)

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 0; j < numCalls; j++ {
                result := luaF("sum", "5", "3")
                if result != "8" {
                    errors <- fmt.Errorf("goroutine %d call %d: expected '8', got '%s'", id, j, result)
                }
            }
        }(i)
    }

    wg.Wait()
    close(errors)

    var errorList []error
    for err := range errors {
        errorList = append(errorList, err)
    }

    if len(errorList) > 0 {
        t.Errorf("Concurrent Lua calls had %d errors:", len(errorList))
        for _, err := range errorList {
            t.Errorf("  - %v", err)
        }
    }
}

// TestLuaPoolReuse tests that Lua states are properly reused
func TestLuaPoolReuse(t *testing.T) {
    if luaPool == nil {
        t.Skip("Lua not available")
    }

    // Call function multiple times
    for i := 0; i < 100; i++ {
        result := luaF("sum", "10", "20")
        if result != "30" {
            t.Errorf("iteration %d: expected '30', got '%s'", i, result)
        }
    }
}
```

## 3. Testy pro addSubstring

**Přidat do functions_test.go:**

```go
// TestAddSubstringFixed tests the fixed version of addSubstring
func TestAddSubstringFixed(t *testing.T) {
    tests := []struct {
        name     string
        s        string
        ss       string
        pos      interface{}
        expected string
    }{
        {
            name:     "positive position from end",
            s:        "Hello!!!",
            ss:       " World",
            pos:      "3",
            expected: "Hello World!!!",
        },
        {
            name:     "negative position from start",
            s:        "Hello!!!",
            ss:       " World",
            pos:      "-5",
            expected: "Hello World!!!",
        },
        {
            name:     "position zero no change",
            s:        "Hello!!!",
            ss:       " World",
            pos:      "0",
            expected: "Hello!!!",
        },
        {
            name:     "position out of range positive",
            s:        "Hello",
            ss:       "X",
            pos:      "10",
            expected: "err:substringOutOfRange",
        },
        {
            name:     "position out of range negative",
            s:        "Hello",
            ss:       "X",
            pos:      "-10",
            expected: "err:substringOutOfRange",
        },
        {
            name:     "insert at beginning",
            s:        "World",
            ss:       "Hello ",
            pos:      "-0",
            expected: "World",
        },
        {
            name:     "insert at end",
            s:        "Hello",
            ss:       " World",
            pos:      "0",
            expected: "Hello",
        },
        {
            name:     "empty string",
            s:        "",
            ss:       "X",
            pos:      "0",
            expected: "",
        },
        {
            name:     "position equals length",
            s:        "Hello",
            ss:       "X",
            pos:      "5",
            expected: "XHello",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := addSubstring(tt.s, tt.ss, tt.pos)
            if result != tt.expected {
                t.Errorf("addSubstring(%q, %q, %v) = %q, expected %q",
                    tt.s, tt.ss, tt.pos, result, tt.expected)
            }
        })
    }
}
```

## 4. Testy pro mustArray

**Přidat do functions_test.go:**

```go
// TestMustArrayNil tests that mustArray returns empty slice for nil
func TestMustArrayNil(t *testing.T) {
    result := mustArray(nil)

    // Check it's not nil
    if result == nil {
        t.Errorf("mustArray(nil) should return empty slice, not nil")
    }

    // Check it's empty
    if len(result) != 0 {
        t.Errorf("mustArray(nil) should return empty slice, got length %d", len(result))
    }

    // Check we can safely range over it
    for range result {
        t.Errorf("mustArray(nil) should be empty")
    }
}

// TestMustArrayInTemplate tests mustArray in template
func TestMustArrayInTemplate(t *testing.T) {
    // Test with nil value
    tpl := `{{range mustArray .value}}item{{end}}done`
    data := map[string]interface{}{"value": nil}

    err := runtv(tpl, "done", data)
    if err != nil {
        t.Errorf("mustArray with nil in template failed: %v", err)
    }

    // Test with actual array
    tpl2 := `{{range mustArray .value}}{{.}}{{end}}`
    data2 := map[string]interface{}{"value": []interface{}{"a", "b"}}

    err = runtv(tpl2, "ab", data2)
    if err != nil {
        t.Errorf("mustArray with array in template failed: %v", err)
    }

    // Test with single value
    tpl3 := `{{range mustArray .value}}{{.}}{{end}}`
    data3 := map[string]interface{}{"value": "hello"}

    err = runtv(tpl3, "hello", data3)
    if err != nil {
        t.Errorf("mustArray with single value in template failed: %v", err)
    }
}
```

## 5. Testy pro readTemplate

**Přidat do main_test.go:**

```go
// TestReadTemplateEmpty tests empty template handling
func TestReadTemplateEmpty(t *testing.T) {
    // Empty string
    _, err := readTemplate("")
    if err == nil {
        t.Errorf("readTemplate(\"\") should return error")
    }
    if !strings.Contains(err.Error(), "empty") {
        t.Errorf("error should mention 'empty', got: %v", err.Error())
    }

    // Just question mark
    _, err = readTemplate("?")
    if err == nil {
        t.Errorf("readTemplate(\"?\") should return error")
    }
    if !strings.Contains(err.Error(), "empty") {
        t.Errorf("error should mention 'empty', got: %v", err.Error())
    }
}

// TestReadTemplateValid tests valid templates
func TestReadTemplateValid(t *testing.T) {
    // Inline template
    result, err := readTemplate("?{{.value}}")
    if err != nil {
        t.Errorf("readTemplate inline failed: %v", err)
    }
    if string(result) != "{{.value}}" {
        t.Errorf("expected '{{.value}}', got: %s", string(result))
    }

    // File template (using existing template file)
    result, err = readTemplate("template.tmpl")
    if err != nil {
        t.Errorf("readTemplate file failed: %v", err)
    }
    if len(result) == 0 {
        t.Errorf("template file should not be empty")
    }
}
```

## 6. Testy pro CSV edge cases

**Přidat do main_test.go:**

```go
// TestMapCSVEdgeCases tests CSV parsing edge cases
func TestMapCSVEdgeCases(t *testing.T) {
    params := tParams{
        inputFormat:    stringPtr("csv"),
        inputDelimiter: stringPtr(","),
    }

    // Empty CSV
    _, err := mapInputData([]byte(""), params)
    if err == nil || !strings.Contains(err.Error(), "no rows") {
        t.Errorf("empty CSV should return 'no rows' error, got: %v", err)
    }

    // Only header
    _, err = mapInputData([]byte("name,age"), params)
    if err == nil || !strings.Contains(err.Error(), "no data rows") {
        t.Errorf("CSV with only header should return 'no data rows' error, got: %v", err)
    }

    // Header + one row (valid)
    result, err := mapInputData([]byte("name,age\nJohn,30"), params)
    if err != nil {
        t.Errorf("valid CSV failed: %v", err)
    }
    mapData := result.([]map[string]interface{})
    if len(mapData) != 1 {
        t.Errorf("expected 1 row, got %d", len(mapData))
    }
    if mapData[0]["name"] != "John" {
        t.Errorf("expected name 'John', got: %v", mapData[0]["name"])
    }
}

// Helper function
func stringPtr(s string) *string {
    return &s
}
```

## 7. Integration testy pro API klíče

**Přidat do main_test.go:**

```go
// TestChatGPTKeyFromEnv tests OpenAI key from environment variable
func TestChatGPTKeyFromEnv(t *testing.T) {
    // Save original env
    originalKey := os.Getenv("OPENAI_API_KEY")
    defer os.Setenv("OPENAI_API_KEY", originalKey)

    // Set test key
    testKey := "sk-test-key-123"
    os.Setenv("OPENAI_API_KEY", testKey)

    // Parse flags (would need to refactor main to make this testable)
    // For now this is more of a documentation of expected behavior

    // The key should be picked up from environment if not provided in CLI
    // params.chatGPTkey should default to os.Getenv("OPENAI_API_KEY")
}
```

## 8. Benchmark testy

**Nový soubor: functions_bench_test.go**

```go
package main

import (
    "testing"
)

func BenchmarkDiv(b *testing.B) {
    for i := 0; i < b.N; i++ {
        div(100, 7)
    }
}

func BenchmarkDivf(b *testing.B) {
    for i := 0; i < b.N; i++ {
        divf(100.5, 7.2)
    }
}

func BenchmarkLuaF(b *testing.B) {
    if luaPool == nil {
        b.Skip("Lua not available")
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        luaF("sum", "5", "3")
    }
}

func BenchmarkLuaFParallel(b *testing.B) {
    if luaPool == nil {
        b.Skip("Lua not available")
    }

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            luaF("sum", "5", "3")
        }
    })
}

func BenchmarkToJSON(b *testing.B) {
    testData := map[string]interface{}{
        "name":  "John",
        "age":   30,
        "city":  "New York",
        "items": []interface{}{1, 2, 3, 4, 5},
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        toJSON(testData)
    }
}

func BenchmarkMustArray(b *testing.B) {
    for i := 0; i < b.N; i++ {
        mustArray([]interface{}{1, 2, 3})
    }
}

func BenchmarkAddSubstring(b *testing.B) {
    for i := 0; i < b.N; i++ {
        addSubstring("Hello World!", "XXX", 5)
    }
}
```

## 9. Fuzz testy (Go 1.18+)

**Nový soubor: functions_fuzz_test.go**

```go
package main

import (
    "testing"
)

func FuzzDiv(f *testing.F) {
    // Seed corpus
    f.Add(int64(10), int64(2))
    f.Add(int64(100), int64(0))
    f.Add(int64(-50), int64(5))

    f.Fuzz(func(t *testing.T, a, b int64) {
        // Should never panic
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("div panicked with a=%d, b=%d: %v", a, b, r)
            }
        }()

        result := div(a, b)

        // If b is zero, result should be 0
        if b == 0 && result != 0 {
            t.Errorf("div(%d, 0) = %d, expected 0", a, result)
        }
    })
}

func FuzzAddSubstring(f *testing.F) {
    // Seed corpus
    f.Add("Hello", "World", 3)
    f.Add("", "X", 0)
    f.Add("Test", "", 1)

    f.Fuzz(func(t *testing.T, s, ss string, pos int) {
        // Should never panic
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("addSubstring panicked with s=%q, ss=%q, pos=%d: %v", s, ss, pos, r)
            }
        }()

        result := addSubstring(s, ss, pos)

        // Result should be a string
        if result == "" && s != "" && ss != "" {
            // Only valid if position was out of range
            if !strings.Contains(result, "err:") {
                t.Errorf("unexpected empty result")
            }
        }
    })
}

func FuzzDateFormat(f *testing.F) {
    f.Add("2021-03-15", "2006-01-02", "02/01/2006")
    f.Add("invalid", "2006-01-02", "02/01/2006")

    f.Fuzz(func(t *testing.T, date, inputFmt, outputFmt string) {
        // Should never panic
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("dateFormat panicked: %v", r)
            }
        }()

        _ = dateFormat(date, inputFmt, outputFmt)
    })
}
```

## Spuštění testů

```bash
# Run all tests
go test -v ./...

# Run with coverage
go test -v -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Run specific test
go test -v -run TestDivByZero

# Run benchmarks
go test -bench=. -benchmem

# Run fuzz tests (Go 1.18+)
go test -fuzz=FuzzDiv -fuzztime=30s
go test -fuzz=FuzzAddSubstring -fuzztime=30s

# Run with race detector
go test -race ./...
```

## Kontrolní seznam testování

Po implementaci oprav:

- [ ] Všechny unit testy procházejí
- [ ] Test coverage je >= 80%
- [ ] Benchmarky neukázaly regresi
- [ ] Fuzz testy našly žádné nové problémy
- [ ] Race detector nenašel žádné problémy
- [ ] Integration testy procházejí
- [ ] Manuální testování edge cases
- [ ] Dokumentace je aktualizována

## Očekávané výsledky

Po implementaci všech oprav by měly testy ukázat:

1. **Zero panics** - žádná funkce by neměla panikovat
2. **100% pass rate** - všechny testy procházejí
3. **High coverage** - >= 80% code coverage
4. **No race conditions** - go test -race nehlásí problémy
5. **Good performance** - benchmarky ukazují konzistentní výkon
