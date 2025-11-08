# Analýza kódu projektu BAFI

## Přehled projektu

BAFI je univerzální nástroj pro převod dat mezi různými formáty (JSON, BSON, YAML, CSV, XML, MT940) s podporou šablon a Lua skriptů. Projekt je napsaný v Go 1.23 a nabízí flexibilní způsob transformace dat.

---

## 🔴 KRITICKÉ PROBLÉMY

### 1. Dělení nulou (functions.go:108, 111)

**Lokace:** `functions.go:108, 111`

```go
func div(a, b interface{}) int64 { return toInt64(a) / toInt64(b) }
func mod(a, b interface{}) int64 { return toInt64(a) % toInt64(b) }
```

**Problém:** Funkce `div()` a `mod()` neověřují dělení nulou, což způsobí panic aplikace.

**Doporučení:**
```go
func div(a, b interface{}) int64 {
    divisor := toInt64(b)
    if divisor == 0 {
        return 0 // nebo vrátit error
    }
    return toInt64(a) / divisor
}
```

### 2. Dělení nulou ve float operacích (functions.go:139-145)

**Lokace:** `functions.go:139-145`

**Problém:** Funkce `divf()` také neověřuje dělení nulou.

**Doporučení:** Přidat kontrolu nulového dělitele před operací.

### 3. Race condition s globální Lua state (main.go:31-32)

**Lokace:** `main.go:31-32`

```go
var (
    luaData *lua.LState
)
```

**Problém:** Globální proměnná `luaData` není thread-safe. Při paralelním zpracování více requestů může dojít k race conditions.

**Doporučení:**
- Použít sync.Pool pro Lua states
- Nebo vytvořit nový LState pro každý request
- Přidat mutex pro synchronizaci přístupu

---

## 🟠 BEZPEČNOSTNÍ PROBLÉMY

### 4. ChatGPT API klíč v CLI parametrech

**Lokace:** `main.go:70`

**Problém:** API klíč je předáván jako parametr příkazové řádky (-gk), což znamená:
- Je viditelný v historii příkazů
- Je viditelný v process listu (ps aux)
- Může být zalogován v různých systémech

**Doporučení:**
- Číst klíč z environment variable (OPENAI_API_KEY)
- Číst z konfiguračního souboru s příslušnými právy (600)
- Použít secrets management systém

### 5. XML External Entity (XXE) injection

**Lokace:** `main.go:286-289`

**Problém:** XML parser neomezuje external entity, což může vést k XXE útokům.

**Doporučení:** Použít bezpečnější konfiguraci XML parseru s vypnutými external entities.

### 6. Chybějící validace vstupních souborů

**Problém:** Aplikace nečte soubory s ověřením velikosti, což může vést k DoS útokům načtením velmi velkých souborů.

**Doporučení:** Přidat limit na velikost vstupního souboru (např. 100MB).

---

## 🟡 DEPRECATED A ZASTARALÉ FUNKCE

### 7. Deprecated rand.Seed (main.go:48)

**Lokace:** `main.go:48`

```go
func init() {
    rand.Seed(time.Now().UTC().UnixNano())
```

**Problém:** `rand.Seed()` je deprecated od Go 1.20. Go nyní automaticky inicializuje random seed.

**Doporučení:**
```go
func init() {
    // rand.Seed není potřeba od Go 1.20+
    if _, err := os.Stat("./lua/functions.lua"); !os.IsNotExist(err) {
        luaData = lua.NewState()
        if err := luaData.DoFile("./lua/functions.lua"); err != nil {
            log.Fatal("loadLuaFunctions", err.Error())
        }
    }
}
```

Pro funkci `randInt`, pokud je potřeba lepší randomness:
```go
func randInt(min, max int) int {
    if max <= min {
        return min
    }
    return rand.IntN(max-min+1) + min // Go 1.22+
}
```

---

## 🟢 LOGICKÉ CHYBY A EDGE CASES

### 8. CSV s pouze hlavičkou (main.go:274)

**Lokace:** `main.go:274`

```go
mapData = make([]map[string]interface{}, len(lines[1:]))
```

**Problém:** Pokud CSV má pouze hlavičku (1 řádek), `lines[1:]` je prázdné pole, ale to už je částečně ošetřeno kontrolou na řádku 271-273.

**Doporučení:** Přidat jasnější chybovou hlášku:
```go
if len(lines) < 2 {
    return nil, fmt.Errorf("mapCSV: CSV has no data rows (only header)")
}
```

### 9. mustArray vrací nil (functions.go:498-500)

**Lokace:** `functions.go:498-500`

```go
func mustArray(v interface{}) []interface{} {
    if v == nil {
        return nil
    }
```

**Problém:** Vrací nil místo prázdného pole, což může způsobit panic při iteraci v šablonách.

**Doporučení:**
```go
func mustArray(v interface{}) []interface{} {
    if v == nil {
        return []interface{}{} // prázdné pole místo nil
    }
```

### 10. addSubstring index out of range (functions.go:324-337)

**Lokace:** `functions.go:324-337`

**Problém:** Funkce má špatnou logiku kontroly indexu. Na řádku 334 používá `s[:-x]`, což není validní Go syntax.

**Doporučení:** Opravit logiku:
```go
func addSubstring(s string, ss string, pos interface{}) string {
    p := toInt(pos)
    sLen := len(s)

    if p == 0 {
        return s
    }

    if p > 0 {
        if p > sLen {
            return "err:substringOutOfRange"
        }
        return s[:sLen-p] + ss + s[sLen-p:]
    }

    // p < 0
    absP := -p
    if absP > sLen {
        return "err:substringOutOfRange"
    }
    return s[:absP] + ss + s[absP:]
}
```

### 11. Chybějící kontrola prázdného template (main.go:320)

**Lokace:** `main.go:320-331`

```go
func readTemplate(textTemplate string) ([]byte, error) {
    var templateFile []byte
    var err error
    if textTemplate[:1] == "?" {
```

**Problém:** Pokud `textTemplate` je prázdný string, dojde k panic při `textTemplate[:1]`.

**Doporučení:**
```go
func readTemplate(textTemplate string) ([]byte, error) {
    if textTemplate == "" {
        return nil, fmt.Errorf("template is empty")
    }
    if textTemplate[:1] == "?" {
```

---

## 💡 NÁVRHY NA VYLEPŠENÍ

### A. Obecná vylepšení kódu

1. **Přidat kontextové timeout**
   - ChatGPT requesty nemají timeout
   - Dlouhodobé operace mohou zabloknout aplikaci
   - Doporučení: použít `context.WithTimeout()`

2. **Lepší error handling**
   - Template funkce vracejí errory jako stringy
   - Doporučení: použít vlastní error typy nebo standardní Go error handling

3. **Přidat logging**
   - Aplikace používá pouze `log.Fatal()`
   - Doporučení: přidat strukturovaný logging (např. zap, logrus)
   - Přidat debug mode s verbose výstupem

4. **Konfigurace pomocí souboru**
   - Všechny parametry jsou CLI
   - Doporučení: přidat podporu konfiguračního souboru (YAML/JSON)

5. **Validace vstupních formátů**
   - Lepší detekce formátu podle obsahu, ne jen podle extension
   - Přidat magic number detection

### B. Vylepšení architektury

1. **Separace concerns**
   - Rozdělit main.go na více modulů:
     - `parser/` - parsování různých formátů
     - `converter/` - konverze mezi formáty
     - `template/` - template engine wrapping
     - `lua/` - Lua integration
     - `chatgpt/` - ChatGPT integration

2. **Interface-based design**
   - Vytvořit interface pro různé parsery
   - Snadnější přidávání nových formátů

3. **Plugin systém**
   - Možnost přidávat vlastní formáty jako pluginy
   - Rozšíření Lua funkcí bez recompilace

### C. Performance optimalizace

1. **Streaming pro velké soubory**
   - Aktuálně se celý soubor načítá do paměti
   - Pro velké CSV/JSON přidat streaming parser

2. **Paralelní zpracování**
   - Při zpracování více souborů (filesTest.yaml) použít goroutines

3. **Memory pooling**
   - Použít sync.Pool pro často alokované objekty
   - Redukce GC pressure

### D. Testování a kvalita

1. **Zvýšit code coverage**
   - Aktuální coverage není 100%
   - Přidat edge case testy

2. **Benchmark testy**
   - Přidat benchmarky pro kritické funkce
   - Měřit performance regrese

3. **Integration testy**
   - Testy end-to-end scenářů
   - Testování s reálnými daty

4. **Fuzz testing**
   - Go 1.18+ native fuzzing
   - Testování s náhodnými vstupy

---

## 🚀 NÁVRHY NA ROZŠÍŘENÍ PROJEKTU

### 1. Nové formáty

- **Parquet** - populární pro big data
- **Avro** - používaný v Kafka ekosystému
- **Protocol Buffers** - Google's serialization format
- **MessagePack** - binary JSON
- **TOML** - konfigurační formát
- **INI** - starší konfigurační formát
- **Excel (XLSX)** - čtení/zápis Excel souborů
- **EDI** - Electronic Data Interchange formáty

### 2. Pokročilé funkce

#### a) Incremental processing
```bash
# Watch mode - automatické zpracování při změně souboru
bafi watch -i input.json -t template.tmpl -o output.txt
```

#### b) HTTP server mode
```bash
# REST API pro transformace
bafi serve -p 8080
curl -X POST http://localhost:8080/transform \
  -H "Content-Type: application/json" \
  -d @input.json
```

#### c) Batch processing
```bash
# Zpracování celého adresáře
bafi batch -i ./data/*.json -t template.tmpl -o ./output/
```

#### d) Data validace
```bash
# Validace podle JSON Schema / XML Schema
bafi validate -i data.json -s schema.json
```

### 3. Integrace s dalšími službami

- **Database export/import** - PostgreSQL, MySQL, MongoDB
- **Cloud storage** - S3, GCS, Azure Blob
- **Message queues** - Kafka, RabbitMQ, NATS
- **Webhooks** - odesílání výsledků na HTTP endpoint

### 4. Vylepšení ChatGPT integrace

- **Streaming odpovědí** - real-time výstup
- **Custom prompts** - vlastní prompt templates
- **Token counting** - odhad nákladů před requestem
- **Caching** - cache častých queries
- **Multiple AI providers** - Claude, Gemini, Llama

### 5. Template rozšíření

- **Template inheritance** - podpora pro base templates
- **Macro systém** - opakovaně použitelné template bloky
- **Custom filters** - uživatelské filtry v šablonách
- **Template debugging** - lepší error messages

### 6. CLI vylepšení

```bash
# Interactive mode
bafi interactive

# Diff mode - porovnání výstupů
bafi diff -i1 file1.json -i2 file2.json

# Merge mode - sloučení více zdrojů
bafi merge -i file1.json,file2.yaml -o merged.json

# Schema generation
bafi schema generate -i data.json -o schema.json
```

### 7. GUI aplikace

- Desktop aplikace (Electron nebo native Go GUI)
- Web UI pro transformace
- Visual template editor
- Live preview transformací

### 8. Bezpečnostní features

- **Encryption** - šifrování vstupních/výstupních souborů
- **Signature verification** - ověření integrity dat
- **Audit log** - logování všech operací
- **Access control** - řízení přístupu k funkcím

### 9. Monitoring a observability

- **Prometheus metrics** - exportování metrik
- **Health check endpoint** - pro kubernetes/docker
- **Distributed tracing** - OpenTelemetry integrace
- **Performance profiling** - built-in pprof server

### 10. Developer experience

- **VS Code extension** - syntax highlighting pro templates
- **Playground** - webová aplikace pro testování
- **Library mode** - použití jako Go library
```go
import "github.com/mmalcek/bafi/pkg/converter"

result, err := converter.Convert(data, converter.Options{
    InputFormat: "json",
    OutputFormat: "xml",
    Template: template,
})
```

---

## 📊 PRIORITY IMPLEMENTACE

### HIGH PRIORITY (kritické opravy)
1. ✅ Opravit dělení nulou
2. ✅ Odstranit deprecated rand.Seed
3. ✅ Opravit race condition s Lua state
4. ✅ Zabezpečit ChatGPT API klíč
5. ✅ Opravit addSubstring bug

### MEDIUM PRIORITY (důležitá vylepšení)
1. Přidat konfigurace přes environment variables
2. Lepší error handling a logging
3. Zvýšit test coverage
4. Přidat validaci velikosti vstupních souborů
5. Implementovat timeout pro dlouhé operace

### LOW PRIORITY (nice-to-have)
1. HTTP server mode
2. Nové formáty (Parquet, Avro)
3. GUI aplikace
4. Cloud storage integrace
5. Monitoring a metrics

---

## 🔧 DOPORUČENÉ NÁSTROJE PRO DEVELOPMENT

1. **Linting a static analysis**
   ```bash
   go install honnef.co/go/tools/cmd/staticcheck@latest
   go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
   ```

2. **Security scanning**
   ```bash
   go install github.com/securego/gosec/v2/cmd/gosec@latest
   ```

3. **Dependency checking**
   ```bash
   go install golang.org/x/vuln/cmd/govulncheck@latest
   ```

4. **Code coverage**
   ```bash
   go test -coverprofile=coverage.out ./...
   go tool cover -html=coverage.out
   ```

---

## 📝 ZÁVĚR

BAFI je solidní nástroj s dobrou funkcionalitou. Hlavní oblasti pro zlepšení jsou:

1. **Bezpečnost** - oprava kritických bugů (dělení nulou, race conditions)
2. **Bezpečnost dat** - lepší handling API klíčů a validace vstupů
3. **Modernizace** - odstranění deprecated funkcí
4. **Rozšiřitelnost** - lepší architektura pro přidávání nových formátů
5. **Developer experience** - lepší dokumentace, tooling, testing

Projekt má velký potenciál a s implementací navržených vylepšení by se mohl stát ještě užitečnějším nástrojem pro data transformace.
