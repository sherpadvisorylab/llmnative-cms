# Component Validator — LLMNative CMS

## Principio

Il validator è una **funzione pura** che accetta la stringa sorgente di un file `.liquid` e restituisce l'elenco degli errori e warning trovati. Array vuoto = component valido.

Nessun side effect, nessun accesso al filesystem, nessuna dipendenza da runtime. Stringa dentro, risultato fuori. Chiamabile da qualsiasi contesto.

---

## API

```typescript
validateComponent(source: string): ValidationResult

interface ValidationResult {
  valid:    boolean   // false se ci sono errori (non warning)
  errors:   Issue[]   // bloccanti — impediscono il salvataggio e il build
  warnings: Issue[]   // advisory — il component funziona ma ha potenziali problemi
}

interface Issue {
  block:   'template' | 'css' | 'js' | 'schema_org' | 'schema' | 'cross'
  line?:   number
  code:    string     // es. "SCHEMA_INVALID_JSON", "CROSS_UNDEFINED_VAR"
  message: string     // testo leggibile
}
```

---

## Regole di validazione

### Blocco `{% schema %}`

| Codice | Livello | Condizione |
|---|---|---|
| `SCHEMA_INVALID_JSON` | error | Il blocco non è JSON valido |
| `SCHEMA_MISSING_NAME` | error | Manca il campo `name` |
| `SCHEMA_INPUT_MISSING_TYPE` | error | Un input non ha `type` |
| `SCHEMA_INPUT_MISSING_LABEL` | error | Un input non ha `label` |
| `SCHEMA_INPUT_UNKNOWN_TYPE` | error | `type` non è un valore noto |
| `SCHEMA_SELECT_MISSING_OPTIONS` | error | `type == "select"` ma mancano le `options` |
| `SCHEMA_DEFAULT_TYPE_MISMATCH` | warning | `default` non è compatibile con il `type` dichiarato |
| `SCHEMA_SHOWIF_INVALID_SYNTAX` | error | Espressione `showIf` non parsabile |
| `SCHEMA_SHOWIF_UNKNOWN_FIELD` | error | `showIf` referenzia un campo non dichiarato negli inputs |

Tipi noti: `text`, `textarea`, `number`, `boolean`, `select`, `image`, `url`, `color`, `date`.

### Blocco template

| Codice | Livello | Condizione |
|---|---|---|
| `TEMPLATE_INVALID_LIQUID` | error | Sintassi LiquidJS non valida |
| `TEMPLATE_UNCLOSED_TAG` | error | Tag Liquid aperto e non chiuso |
| `TEMPLATE_INVALID_COMPONENT_TAG` | error | Tag `{% component %}` con sintassi errata |

### Blocco `{% schema_org %}`

| Codice | Livello | Condizione |
|---|---|---|
| `SCHEMA_ORG_INVALID_LIQUID` | error | Sintassi LiquidJS non valida |
| `SCHEMA_ORG_INVALID_JSON_STATIC` | warning | La parte statica (senza variabili) non è JSON valido |

### Blocchi `{% css %}` e `{% javascript %}`

| Codice | Livello | Condizione |
|---|---|---|
| `CSS_INVALID_SYNTAX` | warning | Sintassi CSS non valida |
| `JS_INVALID_SYNTAX` | warning | Sintassi JS non valida |

CSS e JS sono warning, non errori — il crafter ha libertà totale su questi blocchi.

---

## Validazione cross-block

La più importante: confronta le variabili usate nel template con quelle dichiarate nello schema.

### Algoritmo

```
Per ogni variabile {{ x }} o {{ x.y }} trovata nel template:
  1. È un namespace di sistema (page, site, styles, …)?  → ok, ignorata
  2. È dichiarata negli inputs dello schema?             → ok
  3. Nessuna delle due                                   → CROSS_UNDEFINED_VAR (error)

Per ogni input dichiarato nello schema:
  1. È usato nel template?                              → ok
  2. Non è usato                                        → CROSS_ORPHAN_INPUT (warning)
```

### Codici cross-block

| Codice | Livello | Condizione |
|---|---|---|
| `CROSS_UNDEFINED_VAR` | error | Variabile usata nel template non dichiarata nello schema e non in namespace di sistema |
| `CROSS_ORPHAN_INPUT` | warning | Input dichiarato nello schema ma non usato nel template |
| `CROSS_SHOWIF_SELF_REFERENCE` | error | Un campo ha `showIf` che referenzia se stesso |

### Esempio

```liquid
{{-- template --}}
<h1>{{ heading }}</h1>       ← CROSS_UNDEFINED_VAR: "heading" non in schema
<p>{{ subtitle }}</p>        ← ok, "subtitle" è in schema

{% schema %}
{
  "name": "Hero",
  "inputs": {
    "title":    { "type": "text", "label": "Titolo" },   ← CROSS_ORPHAN_INPUT: "title" non usato
    "subtitle": { "type": "text", "label": "Sottotitolo" }
  }
}
{% endschema %}
```

---

## Struttura del package

```
packages/build-engine/src/validator/
  ├── index.ts                ← export: validateComponent()
  ├── ComponentValidator.ts   ← orchestrazione delle regole
  ├── parsers/
  │   ├── parseBlocks.ts      ← split del file nei 5 blocchi
  │   ├── parseSchema.ts      ← parse + validate JSON schema
  │   ├── parseTemplate.ts    ← estrae variabili usate nel template
  │   └── parseShowIf.ts      ← validate espressioni showIf
  └── rules/
      ├── schema.rules.ts     ← regole blocco schema
      ├── template.rules.ts   ← regole blocco template
      ├── css.rules.ts        ← regole blocco css
      ├── js.rules.ts         ← regole blocco js
      ├── schema_org.rules.ts ← regole blocco schema_org
      └── cross.rules.ts      ← regole cross-block
```

Il validator è **isomorfico**: stesso codice in browser (admin) e Node.js (build pipeline, CLI).

---

## Punti di chiamata

### Admin UI — al salvataggio

```typescript
const result = validateComponent(source)
if (!result.valid) {
  showErrors(result.errors)
  return  // blocca il salvataggio
}
if (result.warnings.length > 0) {
  showWarnings(result.warnings)  // mostra ma non blocca
}
saveComponent(source)
```

### Admin UI — live (opzionale)

Il validator può girare in background mentre il crafter digita, con debounce di ~500ms. Evidenzia gli errori inline nell'editor senza bloccare la digitazione.

### Build pipeline

```typescript
for (const file of componentFiles) {
  const source = readFile(file)
  const result = validateComponent(source)
  if (!result.valid) {
    buildLog.error(`${file}: ${result.errors.map(e => e.message).join(', ')}`)
    throw new BuildError(`Component validation failed: ${file}`)
  }
}
```

### CLI

```bash
llmnative lint components/          # valida tutti i component del sito
llmnative lint components/Hero.liquid  # valida un singolo file
```

### LLM / AI tools

Un LLM che genera component può validare il proprio output prima di restituirlo, riducendo iterazioni di correzione.

```typescript
// usabile direttamente da qualsiasi tool AI integrato nel CMS
const result = validateComponent(generatedSource)
if (!result.valid) {
  // l'AI corregge e riprova
}
```
