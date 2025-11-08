# Test Suite for Critical Fixes

This document contains comprehensive tests for all critical fixes implemented in BAFI.

## Quick Reference

| Test Suite | File | Tests | Estimated Time |
|------------|------|-------|----------------|
| Division by zero | functions_test.go | 6 tests | 20 min |
| Lua concurrency | functions_test.go | 2 tests | 15 min |
| addSubstring | functions_test.go | 9 test cases | 15 min |
| mustArray | functions_test.go | 3 tests | 10 min |
| readTemplate | main_test.go | 2 tests | 10 min |
| CSV edge cases | main_test.go | 3 tests | 10 min |
| Benchmarks | *_bench_test.go | 7 benchmarks | 10 min |
| Fuzz tests | *_fuzz_test.go | 3 fuzz tests | Variable |

---

## 1. Division by Zero Tests

**Add to `functions_test.go`:**

```go
// TestDivByZero tests division by zero handling
func TestDivByZero(t *testing.T) {
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
}

// TestDivInTemplate tests division in template
func TestDivInTemplate(t *testing.T) {
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
}

// TestDivfInTemplate tests float division in template
func TestDivfInTemplate(t *testing.T) {
    err := runt(`{{ divf 10.5 0 }}`, "0")
    if err != nil {
        t.Errorf("template float division by zero failed: %v", err.Error())
    }
}
```

---

## 2. Lua Concurrency Tests

**Add to `functions_test.go`:**

```go
import "sync"

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
        t.Errorf("Concurrent Lua calls had %d errors", len(errorList))
    }
}

// TestLuaPoolReuse tests that Lua states are properly reused
func TestLuaPoolReuse(t *testing.T) {
    if luaPool == nil {
        t.Skip("Lua not available")
    }

    for i := 0; i < 100; i++ {
        result := luaF("sum", "10", "20")
        if result != "30" {
            t.Errorf("iteration %d: expected '30', got '%s'", i, result)
        }
    }
}
```

---

## 3. addSubstring Tests

**Add to `functions_test.go`:**

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
        {"positive from end", "Hello!!!", " World", "3", "Hello World!!!"},
        {"negative from start", "Hello!!!", " World", "-5", "Hello World!!!"},
        {"position zero", "Hello!!!", " World", "0", "Hello!!!"},
        {"out of range positive", "Hello", "X", "10", "err:substringOutOfRange"},
        {"out of range negative", "Hello", "X", "-10", "err:substringOutOfRange"},
        {"empty string", "", "X", "0", ""},
        {"position equals length", "Hello", "X", "5", "XHello"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := addSubstring(tt.s, tt.ss, tt.pos)
            if result != tt.expected {
                t.Errorf("got %q, expected %q", result, tt.expected)
            }
        })
    }
}
```

---

## 4. mustArray Tests

**Add to `functions_test.go`:**

```go
// TestMustArrayNil tests that mustArray returns empty slice for nil
func TestMustArrayNil(t *testing.T) {
    result := mustArray(nil)

    if result == nil {
        t.Errorf("mustArray(nil) should return empty slice, not nil")
    }

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
}
```

---

## 5. readTemplate Tests

**Add to `main_test.go`:**

```go
// TestReadTemplateEmpty tests empty template handling
func TestReadTemplateEmpty(t *testing.T) {
    _, err := readTemplate("")
    if err == nil {
        t.Errorf("readTemplate(\"\") should return error")
    }
    if !strings.Contains(err.Error(), "empty") {
        t.Errorf("error should mention 'empty', got: %v", err.Error())
    }

    _, err = readTemplate("?")
    if err == nil {
        t.Errorf("readTemplate(\"?\") should return error")
    }
}

// TestReadTemplateValid tests valid templates
func TestReadTemplateValid(t *testing.T) {
    result, err := readTemplate("?{{.value}}")
    if err != nil {
        t.Errorf("readTemplate inline failed: %v", err)
    }
    if string(result) != "{{.value}}" {
        t.Errorf("expected '{{.value}}', got: %s", string(result))
    }
}
```

---

## 6. CSV Edge Cases Tests

**Add to `main_test.go`:**

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
        t.Errorf("empty CSV should return 'no rows' error")
    }

    // Only header
    _, err = mapInputData([]byte("name,age"), params)
    if err == nil || !strings.Contains(err.Error(), "no data rows") {
        t.Errorf("CSV with only header should return error")
    }

    // Valid CSV
    result, err := mapInputData([]byte("name,age\nJohn,30"), params)
    if err != nil {
        t.Errorf("valid CSV failed: %v", err)
    }
    mapData := result.([]map[string]interface{})
    if len(mapData) != 1 || mapData[0]["name"] != "John" {
        t.Errorf("unexpected CSV parsing result")
    }
}

func stringPtr(s string) *string {
    return &s
}
```

---

## 7. Benchmark Tests

**New file: `functions_bench_test.go`**

```go
package main

import "testing"

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

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            luaF("sum", "5", "3")
        }
    })
}

func BenchmarkToJSON(b *testing.B) {
    testData := map[string]interface{}{
        "name": "John",
        "age":  30,
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

---

## 8. Fuzz Tests (Go 1.18+)

**New file: `functions_fuzz_test.go`**

```go
package main

import "testing"

func FuzzDiv(f *testing.F) {
    // Seed corpus
    f.Add(int64(10), int64(2))
    f.Add(int64(100), int64(0))
    f.Add(int64(-50), int64(5))

    f.Fuzz(func(t *testing.T, a, b int64) {
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("div panicked with a=%d, b=%d: %v", a, b, r)
            }
        }()

        result := div(a, b)

        if b == 0 && result != 0 {
            t.Errorf("div(%d, 0) = %d, expected 0", a, result)
        }
    })
}

func FuzzAddSubstring(f *testing.F) {
    f.Add("Hello", "World", 3)
    f.Add("", "X", 0)
    f.Add("Test", "", 1)

    f.Fuzz(func(t *testing.T, s, ss string, pos int) {
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("addSubstring panicked: %v", r)
            }
        }()

        _ = addSubstring(s, ss, pos)
    })
}

func FuzzDateFormat(f *testing.F) {
    f.Add("2021-03-15", "2006-01-02", "02/01/2006")

    f.Fuzz(func(t *testing.T, date, inputFmt, outputFmt string) {
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("dateFormat panicked: %v", r)
            }
        }()

        _ = dateFormat(date, inputFmt, outputFmt)
    })
}
```

---

## Running Tests

### Standard Tests
```bash
# Run all tests
go test -v ./...

# Run specific test
go test -v -run TestDivByZero

# Run with coverage
go test -v -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Run with race detector
go test -race ./...
```

### Benchmark Tests
```bash
# Run all benchmarks
go test -bench=. -benchmem

# Run specific benchmark
go test -bench=BenchmarkDiv -benchmem

# Compare benchmarks
go test -bench=. -benchmem > old.txt
# Make changes
go test -bench=. -benchmem > new.txt
benchcmp old.txt new.txt
```

### Fuzz Tests
```bash
# Run fuzz tests (Go 1.18+)
go test -fuzz=FuzzDiv -fuzztime=30s
go test -fuzz=FuzzAddSubstring -fuzztime=30s
go test -fuzz=FuzzDateFormat -fuzztime=30s

# Run all fuzz tests
go test -fuzz=. -fuzztime=1m
```

---

## Test Coverage Goals

Target coverage by file:

| File | Current | Target | Priority |
|------|---------|--------|----------|
| functions.go | ~75% | >= 85% | HIGH |
| main.go | ~70% | >= 80% | HIGH |
| functions_test.go | 100% | 100% | - |
| main_test.go | 100% | 100% | - |

---

## Continuous Integration

**GitHub Actions: `.github/workflows/test.yml`**

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v4
        with:
          go-version: '1.23'

      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.out

      - name: Run benchmarks
        run: go test -bench=. -benchmem

      - name: Run fuzz tests
        run: go test -fuzz=. -fuzztime=10s
```

---

## Test Checklist

After implementing all fixes and tests:

- [ ] All unit tests pass
- [ ] Test coverage >= 80%
- [ ] No benchmarks show regression
- [ ] Fuzz tests find no issues
- [ ] Race detector reports no problems
- [ ] Integration tests pass
- [ ] Manual edge case testing complete
- [ ] Documentation updated

---

## Expected Results

After all tests pass:

- ✅ Zero panics in all tests
- ✅ 100% pass rate
- ✅ High code coverage (>= 80%)
- ✅ No race conditions detected
- ✅ Consistent benchmark performance
- ✅ Fuzz tests stable

---

## References

- [Go Testing Package](https://pkg.go.dev/testing)
- [Table-Driven Tests](https://go.dev/wiki/TableDrivenTests)
- [Go Fuzzing](https://go.dev/doc/fuzz/)
- [Benchmarking](https://pkg.go.dev/testing#hdr-Benchmarks)
- [Race Detector](https://go.dev/doc/articles/race_detector)
