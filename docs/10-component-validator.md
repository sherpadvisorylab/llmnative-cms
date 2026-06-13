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
| `SCHEMA_SELECT_MISSING_OPTIONS` | error | `type == "select"` senza `options` né `optionsSource` |
| `SCHEMA_CHECKLIST_MISSING_OPTIONS` | error | `type == "checklist"` senza `options` né `optionsSource` |
| `SCHEMA_AUTOCOMPLETE_MISSING_OPTIONS` | error | `type == "autocomplete"` senza `options` né `optionsSource` |
| `SCHEMA_OPTIONS_CONFLICT` | error | `options` e `optionsSource` dichiarati insieme — sono mutuamente esclusivi |
| `SCHEMA_DEFAULT_TYPE_MISMATCH` | warning | `default` non è compatibile con il `type` dichiarato |
| `SCHEMA_SHOWIF_INVALID_SYNTAX` | error | Espressione `showIf` non parsabile |
| `SCHEMA_SHOWIF_UNKNOWN_FIELD` | error | `showIf` referenzia un campo non dichiarato negli inputs |
| `SCHEMA_GROUP_INVALID_WIDTH` | error | `width` di un gruppo ha un valore non in `full`, `1/2`, `1/3`, `2/3` |
| `SCHEMA_GROUP_UNKNOWN_REF` | error | Un input referenzia un `group` non dichiarato in `groups` |
| `SCHEMA_LIST_MISSING_CHILDREN` | error | Input di tipo `list` senza `children` |
| `SCHEMA_LIST_CHILDREN_INVALID` | error | Un child di `list` ha `type: "list"` — nesting non permesso |
| `SCHEMA_LIST_CHILDREN_NO_TYPE` | error | Un child di `list` non ha `type` |
| `SCHEMA_LIST_CHILD_SHOWIF_UNKNOWN` | error | `showIf` di un child referenzia un campo non dichiarato nei `children` dello stesso `list` |
| `SCHEMA_ALIAS_NOT_ARRAY` | error | `alias` è presente ma non è un array |
| `SCHEMA_ALIAS_NOT_STRING` | error | Un elemento dell'array `alias` non è una stringa |
| `SCHEMA_ALIAS_DUPLICATE` | warning | Un nome alias è già usato come chiave in un altro input (collisione garantita) |
| `SCHEMA_ALIAS_SELF_REFERENCE` | error | Un alias è uguale al nome del campo corrente (inutile e ambiguo) |

Tipi scalari noti: `string`, `textarea`, `number`, `email`, `password`, `color`, `date`, `time`, `datetime`, `week`, `month`, `boolean`, `switch`, `checkbox`, `select`, `autocomplete`, `checklist`, `uploadImage`, `uploadDocument`, `imageUrl`.
Tipo strutturato: `list` (con `children`).

### Blocco template

| Codice | Livello | Condizione |
|---|---|---|
| `TEMPLATE_INVALID_LIQUID` | error | Sintassi LiquidJS non valida |
| `TEMPLATE_UNCLOSED_TAG` | error | Tag Liquid aperto e non chiuso |
| `TEMPLATE_INVALID_COMPONENT_TAG` | error | Tag `{% component %}` con sintassi errata |
| `TEMPLATE_RENDER_TAG_FORBIDDEN` | error | Tag `{% render %}` non permesso — usare `{% component %}` |
| `TEMPLATE_ASSIGN_FORBIDDEN` | error | Tag `{% assign %}` non permesso — usare filtri inline |
| `TEMPLATE_CAPTURE_FORBIDDEN` | error | Tag `{% capture %}` non permesso |
| `TEMPLATE_INCREMENT_FORBIDDEN` | error | Tag `{% increment %}` / `{% decrement %}` non permessi |
| `TEMPLATE_DUPLICATE_COMPONENT_NO_AS` | warning | Lo stesso component editoriale appare più volte senza `as:` esplicito — il builder assegna auto-suffix (`Feature`, `Feature-2`, …) ma la chiave è order-sensitive |
| `TEMPLATE_EDITORIAL_STATIC_PARAM` | error | Tag `{% component %}` su component editoriale con parametri statici diversi da `as:` — i valori degli editorial sono sempre gestiti dal creator |
| `TEMPLATE_STRUCTURAL_AS_PARAM` | error | Tag `{% component %}` su component structural con parametro `as:` — i structural non hanno dati da salvare |
| `TEMPLATE_AS_INVALID_FORMAT` | error | Valore di `as:` contiene caratteri non validi — solo alfanumerici e `_` sono permessi |
| `TEMPLATE_AS_EMPTY` | error | Valore di `as:` è una stringa vuota |

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

Per ogni loop {% for X in Y %} nel template:
  1. Y è un namespace di sistema?                       → ok, ignorato
  2. Y è dichiarato nello schema come type: "list"?     → ok
  3. Y non è dichiarato nello schema                    → CROSS_UNDEFINED_VAR (error)
  4. Y è dichiarato ma non è type: "list"               → CROSS_FOR_ON_NON_LIST (error)
  Per ogni {{ X.prop }} dentro il loop:
  5. prop è un child dichiarato in Y.children           → ok
  6. prop non esiste in Y.children                      → CROSS_UNDEFINED_VAR (error)
  (X è solo un alias di contesto — ignorato dalla validazione)
```

### Codici cross-block

| Codice | Livello | Condizione |
|---|---|---|
| `CROSS_UNDEFINED_VAR` | error | Variabile usata nel template non dichiarata nello schema e non in namespace di sistema |
| `CROSS_ORPHAN_INPUT` | warning | Input dichiarato nello schema ma non usato nel template |
| `CROSS_SHOWIF_SELF_REFERENCE` | error | Un campo ha `showIf` che referenzia se stesso |
| `CROSS_FOR_ON_NON_LIST` | error | `{% for X in Y %}` dove `Y` è dichiarato nello schema ma non è `type: "list"` |
| `CROSS_DUPLICATE_AS` | error | Due tag `{% component %}` nella stessa pagina usano lo stesso valore `as:` — collisione di chiave dati |

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

## Schema diff detection (admin)

Questa logica opera **al salvataggio nell'admin** — non è parte di `validateComponent()`, che è stateless. È un comportamento separato del layer admin che confronta il nuovo schema con quello precedentemente salvato.

### Algoritmo

```
1. Carica lo schema salvato (versione precedente)
2. Confronta con il nuovo schema ricevuto
3. Trova i campi rimossi (in vecchio, non in nuovo) e aggiunti (in nuovo, non in vecchio)
4. Per ogni campo rimosso, mostra un dialog al crafter:

   "Campo 'heading' rimosso. È stato rinominato?"
   [ Sì → seleziona il nuovo nome: _____ ]   [ No, eliminato ]

5. Se confermato rename:
   - aggiunge il vecchio nome nell'array alias del nuovo campo
   - salva il component aggiornato

6. Se confermato delete:
   - il campo viene rimosso senza alias
   - le pagine che usavano quel campo manterranno il dato orfano nei loro record
     (non viene cancellato — il builder lo ignora semplicemente)
```

### Validazione alias post-salvataggio

Dopo che l'admin ha auto-generato gli alias dalla detect+prompt, il validator viene chiamato sul component aggiornato. Le regole `SCHEMA_ALIAS_*` garantiscono la correttezza degli alias prodotti.

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
