# Template System — LLMNative CMS

## Principio

Il sistema è **data-driven**: sono i dati che definiscono le pagine, non il contrario.
Il template è uno scheletro che sa come presentare i dati — non contiene dati.

---

## Nomenclatura: Component

L'unità atomica del template system si chiama **Component**.

Convenzione di disambiguazione (solo per chi sviluppa il CMS):
- **Component** — sempre CMS Component (LiquidJS), se non specificato diversamente
- **React component** — quando si parla esplicitamente della parte `@llmnative/react`

`@llmnative/react` è un vendor opaco. Creator, crafter e AI che usano il CMS conoscono solo i CMS Component.

---

## Template engine: LiquidJS

Scelto per:
- **Sandbox sicuro** — il crafter scrive template senza poter eseguire codice arbitrario
- **Browser + Node.js** — stesso engine per preview in-browser e build engine server-side
- **Pre-compilabile** — i template si compilano una volta sola in AST, poi si eseguono n volte
- **Sintassi leggibile** — `{{ var }}`, `{% if %}`, `{% for %}`, filtri `| upcase`
- **Estendibile** — custom tags per il sistema di composizione component
- **LLM-friendly** — Shopify lo usa massivamente, è nel training data di tutti i major LLM

---

## Due nature di component

### Editorial

Ha il blocco `{% schema %}` con inputs. Il CMS genera automaticamente il form di edit per il creator. I dati vengono salvati per istanza.

### Structural

Non ha `{% schema %}` (o ha inputs vuoti). Puramente decorativo o di layout. Nessuna interazione con il creator, nessun dato salvato. I parametri sono hardcoded dal crafter nel tag di inclusione.

Il builder distingue i due tipi automaticamente: ha `{% schema %}` con inputs → editorial. Non ce l'ha → structural.

Esempi structural: `Divider`, `Spacer`, `AnimatedBackground`, `TwoColumnWrapper`
Esempi editorial: `Hero`, `FeatureGrid`, `Testimonials`, `ArticleCard`

---

## File e registry

```
components/Hero.liquid            ← custom del crafter (ha precedenza)
templates/components/Hero.liquid  ← built-in del CMS
```

- PascalCase, nessun suffisso
- Se stesso nome, il custom del crafter vince sul built-in
- Un component = un file `.liquid`

---

## Struttura del file (ordine fisso)

I 5 blocchi sono sempre nello stesso ordine. Solo il template è obbligatorio.

```liquid
{% comment %} 1. TEMPLATE (obbligatorio) {% endcomment %}
<section class="hero hero--{{ layout }}">
  <h1>{{ title }}</h1>
  {% if subtitle %}<p class="hero__subtitle">{{ subtitle }}</p>{% endif %}
  {% if image %}{{ image | img_tag }}{% endif %}
  {% if cta_text %}
    <a class="hero__cta" href="{{ cta_url }}">{{ cta_text }}</a>
  {% endif %}
</section>

{% comment %} 2. CSS (opzionale) {% endcomment %}
{% css %}
.hero { padding: 4rem 2rem; background: {{ styles.bg_primary }}; }
.hero--centered { text-align: center; }
.hero--left { text-align: left; }
{% endcss %}

{% comment %} 3. JS (opzionale) {% endcomment %}
{% javascript %}
// nessuna regola imposta — il crafter fa come vuole
{% endjavascript %}

{% comment %} 4. SCHEMA_ORG (opzionale) {% endcomment %}
{% schema_org %}
{
  "@context": "https://schema.org",
  "@type": "WPHeader",
  "name": "{{ title }}"
}
{% endschema_org %}

{% comment %} 5. SCHEMA (opzionale — necessario per generare l'edit UI) {% endcomment %}
{% schema %}
{
  "name": "Hero",
  "description": "Sezione hero con titolo, immagine e CTA",
  "category": "layout",
  "inputs": {
    "title":    { "type": "text",   "label": "Titolo",      "required": true },
    "subtitle": { "type": "text",   "label": "Sottotitolo"                   },
    "image":    { "type": "image",  "label": "Immagine"                      },
    "cta_text": { "type": "text",   "label": "Testo CTA"                     },
    "cta_url":  { "type": "url",    "label": "URL CTA"                       },
    "layout":   { "type": "select", "label": "Layout",
                  "options": ["centered", "left", "right"],
                  "default": "centered"                                        }
  }
}
{% endschema %}
```

---

## Scope delle variabili

Il template riceve tre categorie di variabili:

| Variabile | Sorgente | CMS la gestisce? | Edit UI? |
|---|---|---|---|
| `{{ title }}`, `{{ layout }}`, … | `{% schema %}.inputs` | ✅ sì | ✅ sì |
| `{{ page.slug }}`, `{{ page.seo }}` | namespace sistema | ❌ no | ❌ no |
| `{{ site.name }}`, `{{ site.locale }}` | namespace sistema | ❌ no | ❌ no |
| `{{ styles.primary }}`, `{{ styles.font }}` | namespace sistema | ❌ no | ❌ no |

**Regola**: ogni variabile non appartenente a un namespace di sistema è un input locale → il builder genera il campo di edit corrispondente usando `Component.schema[type]()`.

### Namespace di sistema riservati

| Namespace | Contenuto |
|---|---|
| `page` | Metadata della pagina corrente (`slug`, `seo`, `locale`, `url`, …) |
| `site` | Impostazioni globali del sito (`name`, `locale`, `baseUrl`, …) |
| `styles` | Design tokens condivisi (`primary`, `font`, `bg_primary`, …) |

I namespace sono estendibili. Le variabili di sistema si usano liberamente nel template ma non generano mai campi di edit.

### i18n

Il component è locale-agnostico. Riceve sempre stringhe già risolte nel locale corrente — non sa e non deve sapere in che lingua sta operando. Se il sito è multilingua, il builder produce una build separata per ogni locale: ogni pagina viene renderizzata una volta per lingua, con i dati del locale corrente passati come variabili. La gestione della locale è una responsabilità del layer pagina, non del component.

---

## Il blocco `{% schema %}`

```json
{
  "name":        "Nome leggibile del component",
  "description": "Descrizione opzionale",
  "category":    "layout | content | media | navigation | utility",
  "groups": [
    { "id": "content", "label": "Contenuto", "width": "full" },
    { "id": "style",   "label": "Stile",     "width": "1/2"  },
    { "id": "media",   "label": "Media",     "width": "1/2"  }
  ],
  "inputs": {
    "chiave_scalare": {
      "type":     "string | textarea | number | email | password | color | date | time | datetime | week | month | boolean | switch | checkbox | select | autocomplete | checklist | uploadImage | uploadDocument | imageUrl",
      "label":    "Label visibile nel form di edit",
      "group":    "id del gruppo (opzionale)",
      "required": true,
      "default":  "valore di default",
      "options":  ["opzione1", "opzione2"],
      "showIf":   "espressione condizionale",
      "alias":    ["vecchio_nome", "altro_vecchio_nome"]
    },
    "chiave_lista": {
      "type":     "list",
      "label":    "Label visibile nel form di edit",
      "group":    "id del gruppo (opzionale)",
      "min":      0,
      "max":      10,
      "children": {
        "sotto_chiave": {
          "type":     "qualsiasi tipo scalare (non list)",
          "label":    "Label",
          "required": true,
          "default":  "valore di default"
        }
      }
    }
  }
}
```

Gli `inputs` guidano due sistemi:
1. **Edit UI**: il builder usa `Component.schema[type](overrides)` per generare il form del creator
2. **Validazione**: il builder verifica che i dati passati al component rispettino il contratto

### Gruppi di input

`groups` è opzionale. Serve a raggruppare gli inputs in sezioni visive nell'edit UI. Ogni input può dichiarare il proprio gruppo con la proprietà `group` — gli inputs senza `group` finiscono in una sezione generica in cima al form.

`width` controlla la larghezza del gruppo nella griglia dell'edit UI. Valori accettati: `full` (default), `1/2`, `1/3`, `2/3`. Gruppi adiacenti si affiancano finché la riga è piena, poi vanno a capo.

```json
"groups": [
  { "id": "content", "label": "Contenuto"                   },
  { "id": "style",   "label": "Stile",   "width": "1/2"    },
  { "id": "seo",     "label": "SEO",     "width": "1/2"    }
]
```

### Proprietà type-specifiche

Alcuni tipi accettano proprietà aggiuntive opzionali direttamente sull'input, oltre a quelle comuni (`label`, `required`, `default`, `showIf`, `alias`, `group`):

| Tipo | Proprietà aggiuntive |
|---|---|
| `string`, `textarea`, `email`, `password` | `placeholder` |
| `textarea` | `rows`, `maxRows` |
| `number` | `min`, `max`, `step` |
| `date` | `min`, `max` |
| `select`, `checklist`, `autocomplete` | `options` (statico) oppure `optionsSource` (dinamico) |
| `autocomplete` | `minItems`, `maxItems`, `creatable` |
| `uploadImage`, `uploadDocument` | `accept`, `max`, `multiple` |
| `uploadImage` | `previewWidth`, `previewHeight` |

`options` e `optionsSource` sono mutuamente esclusivi — uno dei due è obbligatorio per `select`, `checklist`, `autocomplete`.

**Options statico:**
```json
"category": {
  "type":    "select",
  "label":   "Categoria",
  "options": ["news", "tutorial", "case-study"]
}
```

**Options dinamico da collezione** — carica le opzioni dal data provider a edit time:
```json
"category": {
  "type":  "autocomplete",
  "label": "Categoria",
  "optionsSource": {
    "path":     "categories",
    "fieldMap": { "label": "name", "value": "id" },
    "order":    { "name": "asc" }
  }
}
```

`optionsSource.path` è il percorso della collezione nel data provider. `fieldMap` mappa i campi del record alle proprietà `label` e `value` dell'opzione. `where` e `order` filtrano e ordinano le opzioni caricate.

### Input di tipo `list`

`list` definisce un array di oggetti gestito dal creator nell'edit UI — può aggiungere, rimuovere e riordinare gli items. Ogni item ha la struttura definita in `children`.

```json
"social_links": {
  "type":  "list",
  "label": "Social links",
  "min":   1,
  "max":   8,
  "children": {
    "name": { "type": "string",   "label": "Nome",  "required": true },
    "href": { "type": "string",   "label": "URL",   "required": true },
    "icon": { "type": "imageUrl", "label": "Icona"                   }
  }
}
```

Nel template si itera con `{% for %}` standard LiquidJS:

```liquid
<ul class="social-links">
  {% for item in social_links %}
    <li>
      <a href="{{ item.href }}">
        {% if item.icon %}<img src="{{ item.icon }}" alt="{{ item.name }}">{% endif %}
        {{ item.name }}
      </a>
    </li>
  {% endfor %}
</ul>
```

I `children` accettano qualsiasi tipo scalare. `list` dentro `children` non è permesso — un solo livello di profondità. Ogni child supporta le stesse proprietà degli inputs di primo livello: `required`, `default`, `showIf`, `alias`, `options`/`optionsSource` e le proprietà type-specifiche del suo tipo.

### Build-time data — iterazione su collezioni

Un component può iterare su dati dinamici del sito (es. articoli, prodotti) usando `{% for %}` su variabili del namespace `collections`, rese disponibili dal build engine a render time:

```liquid
{% for post in collections.posts limit: 5 %}
  <article>
    <h2>{{ post.title }}</h2>
    <p>{{ post.excerpt }}</p>
  </article>
{% endfor %}
```

Il component non dichiara dipendenze esplicite — referenzia la variabile e ci itera sopra come con qualsiasi altra variabile di sistema. Il namespace `collections` è definito nel layer dati (vedere `docs/03-data-model.md`).

### Retrocompatibilità dei nomi — `alias`

`alias` è un array di nomi precedenti che il campo accettava. Il builder lo usa come fallback di lettura: se il campo corrente non è presente nei dati della pagina, cerca i nomi nell'array `alias` nell'ordine dichiarato.

```json
"title": { "type": "text", "label": "Titolo", "alias": ["heading", "h1"] }
```

Quando il builder renderizza una pagina con dati `{ "heading": "Benvenuto" }`:
```
1. Cerca "title" nei dati → non trovato
2. Controlla alias → trova "heading" → usa quel valore
3. {{ title }} renderizza "Benvenuto"
```

I dati originali non vengono mai modificati. L'alias è un fallback di lettura puro.

**Quando usarlo:**
- Il crafter rinomina un campo nel proprio component — l'alias garantisce che le pagine già salvate continuino a funzionare
- Si importa un component esterno o built-in che usa nomi diversi da quelli già presenti nei dati del sito

#### Rilevamento automatico nell'admin

Quando il crafter salva un component con modifiche allo schema, l'admin confronta il nuovo schema con quello salvato in precedenza e rileva i campi rimossi e aggiunti.

Per ogni campo rimosso, l'admin chiede:

```
Campo "heading" rimosso.
È stato rinominato?

[ Sì → scegli il nuovo nome: title ▾ ]   [ No, eliminato ]
```

Se il crafter conferma la rinomina, l'admin aggiunge automaticamente `alias` al nuovo campo — nessuna scrittura manuale.

#### Evoluzione futura

`alias` è la base per una procedura opzionale di migrazione bulk: scansiona tutte le pagine che usano nomi aliasati, aggiorna i dati al nome corrente, rimuove l'alias una volta che tutte le istanze sono pulite. Questa procedura coinvolge il layer pagina e il builder — non il component — ed è una fase futura opzionale.

### Visibilità condizionale — `showIf`

`showIf` controlla la visibilità di un campo nell'edit UI in base al valore di altri campi dello stesso component. Il campo esiste sempre nei dati — semplicemente non viene mostrato al creator quando la condizione è falsa.

```json
{% schema %}
{
  "name": "Hero",
  "inputs": {
    "title":       { "type": "text",     "label": "Titolo" },
    "subtitle":    { "type": "text",     "label": "Sottotitolo" },
    "show_cta":    { "type": "boolean",  "label": "Mostra CTA",    "default": false },
    "cta_text":    { "type": "text",     "label": "Testo CTA",     "showIf": "show_cta == true" },
    "cta_url":     { "type": "url",      "label": "URL CTA",       "showIf": "show_cta == true" },
    "layout":      { "type": "select",   "label": "Layout",
                     "options": ["centered", "left", "grid"],      "default": "centered" },
    "columns":     { "type": "number",   "label": "Colonne",       "showIf": "layout == 'grid'",
                     "default": 3 }
  }
}
{% endschema %}
```

#### Operatori supportati

| Operatore | Esempio |
|---|---|
| `==` `!=` | `"layout == 'grid'"` · `"title != ''"` |
| `>` `<` `>=` `<=` | `"columns > 1"` |
| `in` | `"layout in ['grid', 'masonry']"` |
| `nin` | `"layout nin ['centered', 'left']"` |
| `like` | `"title like 'hero*'"` · `"slug like '*news*'"` |
| `&&` | `"show_cta == true && layout != 'grid'"` |
| `\|\|` | `"title != '' \|\| subtitle != ''"` |
| `!` | `"!show_cta"` |

`like` supporta il wildcard `*` in qualsiasi posizione: `'hero*'` (inizia con), `'*news*'` (contiene), `'*footer'` (finisce con).

#### Separazione di responsabilità

`showIf` governa la **visibilità nell'edit UI** — non il rendering HTML. Sono due cose distinte e possono avere condizioni diverse:

```liquid
{% comment %} edit UI: cta_url visibile solo se show_cta è true (via showIf in schema) {% endcomment %}
{% comment %} template: rendering condizionale indipendente {% endcomment %}
{% if show_cta and cta_url != "" %}
  <a class="hero__cta" href="{{ cta_url }}">{{ cta_text }}</a>
{% endif %}
```

---

## CSS e JS

- Il builder concatena i blocchi `{% css %}` e `{% javascript %}` di tutti i component usati in una pagina in un unico bundle per pagina
- Se lo stesso component appare più volte, CSS e JS vengono inclusi **una sola volta** — le istanze condividono lo stesso bundle
- Il blocco `{% schema_org %}` è un template LiquidJS — riceve le stesse variabili del template principale. Il builder lo inietta come `<script type="application/ld+json">` nell'`<head>` — **una volta per istanza**

### `{% css %}` — template LiquidJS con namespace di sistema

Il blocco `{% css %}` è processato come template LiquidJS. Riceve **solo i namespace di sistema** (`styles`, `site`, `page`) — utile per usare design token nel CSS:

```liquid
{% css %}
.hero {
  background: {{ styles.bg_primary }};
  font-family: {{ styles.font_primary }};
  color: {{ styles.text_primary }};
}
{% endcss %}
```

Gli input del creator non sono disponibili nel CSS — sarebbero incoerenti (il CSS è condiviso tra tutte le istanze) e aprirebbero a CSS injection.

### `{% javascript %}` — statico

Il blocco `{% javascript %}` è **statico** — non è processato come LiquidJS. Il codice viene incluso as-is nel bundle. Nessuna variabile disponibile.

### Isolamento JS tra component

Ogni blocco `{% javascript %}` è autonomo. Il CMS non fornisce nessun bus di comunicazione tra component — se il crafter ha bisogno che due component si parlino a runtime (es. un `FilterBar` che aggiorna un `ProductGrid`), usa `CustomEvent` nativi del browser. Il CMS concatena i bundle, il runtime è responsabilità del crafter.

### Deduplicazione CSS e JS

Se lo stesso component appare più volte nella stessa pagina (es. due istanze di `FeatureCard`), il builder include i blocchi `{% css %}` e `{% javascript %}` di quel component **una sola volta** nel bundle della pagina. Le istanze multiple condividono lo stesso CSS/JS — solo i dati differiscono.

---

## Tag `{% component %}`

Il custom tag usato per includere un component in un page template o dentro un altro component.

### Sintassi

I parametri accettati dipendono dal tipo di component:

```liquid
{% comment %} Editorial: solo as: {% endcomment %}
{% component "NomeComponent" %}
{% component "NomeComponent" as: "chiave" %}

{% comment %} Structural: solo parametri statici, nessun as: {% endcomment %}
{% component "NomeComponent" param: valore %}
```

**Editorial** non accetta parametri statici nel tag — i valori sono sempre gestiti dal creator tramite l'edit UI. Il crafter che vuole varianti diverse crea component distinti (`HeroDark.liquid`, `HeroLight.liquid`) o usa uno schema con un campo `variant`.

**Structural** non accetta `as:` — non ha dati da salvare, i parametri del tag sono il suo unico input.

### Parametro `as` (editorial)

`as` definisce la chiave sotto cui i dati dell'istanza vengono salvati e recuperati.

- **Omesso su occorrenza singola**: chiave = nome del component (`"Hero"`)
- **Omesso su occorrenze multiple**: il builder assegna auto-suffix progressivo (`"FeatureCard"`, `"FeatureCard_2"`, …) e il validator emette un warning — la chiave è order-sensitive
- **Esplicito**: chiave stabile — preferito quando lo stesso component appare più volte

**Formato del valore `as:`**: solo caratteri alfanumerici e underscore (`_`). No spazi, no trattini, non vuoto. Esempi validi: `"top"`, `"card_1"`, `"main_hero"`. Il valore diventa una chiave nel data store della pagina.

```liquid
{% comment %} singola occorrenza → chiave "Hero" {% endcomment %}
{% component "Hero" %}

{% comment %} as esplicito → chiavi stabili {% endcomment %}
{% component "FeatureCard" as: "top" %}
{% component "FeatureCard" as: "bottom" %}

{% comment %} nested editorial → chiave sotto il parent {% endcomment %}
{% component "AuthorAvatar" as: "author" %}

{% comment %} structural → parametri statici, nessun dato salvato {% endcomment %}
{% component "Divider" style: "dashed" %}
{% component "Spacer" height: 40 %}
```

### Struttura dati risultante

```json
{
  "Hero":        { "title": "Benvenuto", "layout": "centered" },
  "top":         { "title": "Feature A", "columns": 3 },
  "bottom":      { "title": "Feature B", "columns": 2 },
  "ArticleCard": {
    "title": "Articolo",
    "author": { "name": "Mario Rossi", "photo": "...", "role": "Editor" }
  }
}
```

### Risoluzione dei valori

Al momento del render il builder risolve ogni variabile in questo ordine:

| Priorità | Sorgente | Note |
|---|---|---|
| 1 | Valore salvato dal creator | Scelta esplicita — vince sempre |
| 2 | `default` nello schema | Fallback dichiarato nel component |
| 3 | Stringa vuota / zero del tipo | Nessun valore disponibile |

---

## Nesting

I component possono includere altri component a profondità infinita.

### Structural dentro editorial
```liquid
{% comment %} Hero.liquid {% endcomment %}
<section class="hero">
  <h1>{{ title }}</h1>
  {% component "Divider" style: "light" %}
</section>
```

### Editorial dentro editorial
```liquid
{% comment %} ArticleCard.liquid {% endcomment %}
<article>
  <h2>{{ title }}</h2>
  <p>{{ excerpt }}</p>
  {% component "AuthorAvatar" as: "author" %}
</article>
```

L'edit UI del parent mostra i propri campi + una sezione dedicata per i campi del component figlio.

### Component in branch condizionali

Il crafter può includere component dentro blocchi `{% if %}`:

```liquid
{% if page.locale == 'en' %}
  {% component "HeroBig" %}
{% else %}
  {% component "HeroCompact" %}
{% endif %}
```

L'admin mostra sempre i campi di tutti i component dichiarati nel template, indipendentemente dalle condizioni. Il creator compila i dati di entrambi; il builder a render time usa solo quello che la condizione seleziona. I dati dell'altro restano salvati ma non vengono renderizzati.

### Cycle detection

Il builder mantiene la **render chain** corrente. Se un component è già presente nella catena, il render viene bloccato.

```
Hero → AuthorAvatar → SocialLinks          ✅ OK
Hero → AuthorAvatar → Hero                 ❌ BLOCCATO
ArticleCard → AuthorAvatar → ArticleCard   ❌ BLOCCATO
```

---

## Uso nel page template

Il crafter definisce la struttura della pagina. Il creator popola i dati di ogni istanza.

```liquid
{% layout "BaseLayout" %}

{% block content %}
  {% component "Hero" %}
  {% component "FeatureGrid" %}
  {% component "FeatureCard" as: "top" %}
  {% component "FeatureCard" as: "bottom" %}
  {% component "Divider" %}
  {% component "Testimonials" %}
{% endblock %}
```

---

## Component built-in

Il CMS fornisce una libreria di component pronti all'uso in `templates/components/`.
Il crafter può sovrascriverli creando un file con lo stesso nome in `components/`.

```
templates/components/
  ├── Hero.liquid
  ├── Heading.liquid
  ├── Body.liquid
  ├── FeatureGrid.liquid
  ├── ArticleCard.liquid
  ├── Testimonials.liquid
  ├── AuthorAvatar.liquid
  ├── TagList.liquid
  ├── Divider.liquid
  ├── Spacer.liquid
  └── ImageGallery.liquid
```

---

## Export e import di component

**Il formato del file `.liquid` è il formato di scambio.** Non esiste un formato di export separato.

Un component creato in `components/Hero.liquid` è già pronto per essere:
- condiviso con altri crafter così com'è
- importato in qualsiasi altro sito copiando il file in `components/`
- contribuito alla libreria built-in del CMS

### Contribuire alla libreria built-in

Qualsiasi component collocato in `templates/components/` del repository CMS diventa automaticamente disponibile a tutte le installazioni del CMS. È il meccanismo con cui la libreria built-in cresce nel tempo.

```
Crafter crea Hero.liquid nel suo sito
    └── lo copia in templates/components/ del repo CMS
    └── da quel momento tutti i siti che installano il CMS hanno Hero disponibile
```

### Priorità di risoluzione

Il builder risolve il component nel seguente ordine:

```
1. components/Hero.liquid          ← custom del sito (massima priorità)
2. templates/components/Hero.liquid ← built-in del CMS
```

Il custom vince sempre. Il crafter può quindi sovrascrivere qualsiasi component built-in senza toccare il codice del CMS.

---

## Filtri custom registrati dal builder

```liquid
{{ body | markdown }}           {% comment %} markdown → HTML {% endcomment %}
{{ date | cms_date: "it" }}     {% comment %} formattazione data localizzata {% endcomment %}
{{ image | img_tag }}           {% comment %} <img> con src, alt, lazy loading {% endcomment %}
{{ image | srcset: [400,800] }} {% comment %} attributo srcset per immagini responsive {% endcomment %}
{{ title | slugify }}           {% comment %} "Titolo Articolo" → "titolo-articolo" {% endcomment %}
{{ price | currency: "EUR" }}   {% comment %} 12.5 → "€ 12,50" {% endcomment %}
```

---

## Error handling nel render

Il comportamento varia in base al contesto.

### Build pipeline (produzione)

| Condizione | Comportamento |
|---|---|
| Component non trovato | Errore bloccante — la build fallisce |
| Campo `required` vuoto | Errore bloccante — la build fallisce |
| Campo non-required vuoto | Stringa vuota, build continua |
| Ciclo di nesting rilevato | Errore bloccante |

Nessuna pagina con errori viene pubblicata.

### Preview in-browser (admin)

| Condizione | Comportamento |
|---|---|
| Component non trovato | Placeholder visibile: `[ Component "Hero" non trovato ]` |
| Campo `required` vuoto | Placeholder inline: `[ titolo mancante ]` + campo evidenziato nell'edit UI |
| Campo non-required vuoto | Stringa vuota — il template gestisce il caso con `{% if %}` |
| Ciclo di nesting rilevato | Placeholder: `[ Ciclo rilevato: Hero → AuthorAvatar → Hero ]` |

L'admin non crasha mai — il crafter deve poter lavorare anche con dati incompleti.

---

## Preview in-browser

La preview gira interamente nel browser — nessun server coinvolto.

Il meccanismo usa `BroadcastChannel`: l'editor invia sorgente del component + dati correnti del creator attraverso un canale nominato. Un iframe in ascolto sullo stesso canale riceve il messaggio, compila il template con LiquidJS (che gira in-browser) e aggiorna il rendering in tempo reale.

```
Editor (modifica sorgente o dati)
    └── BroadcastChannel.postMessage({ source, data })
            └── iframe preview
                    └── LiquidJS.render(source, data)
                    └── aggiorna il DOM
```

La preview riflette esattamente ciò che il build engine produrrebbe — stesso engine, stesso template, stessi dati — con la sola differenza che in preview gli errori mostrano placeholder invece di bloccare.

### Marcatori DOM per il click-to-edit

In modalità preview il renderer avvolge ogni component in un wrapper con attributi `data-cms-*`, che permettono all'admin di mappare il click del creator al component e alla chiave dati corrispondente:

```html
<div data-cms-component="Hero" data-cms-key="main">
  <!-- output HTML del component, invariato -->
  <section class="hero">...</section>
</div>
```

I marcatori sono **iniettati solo in preview** — la build di produzione produce HTML pulito, senza nessun attributo o wrapper CMS. L'output statico finale non ha tracce del sistema.

---

## Tag LiquidJS vietati

Per mantenere i template dichiarativi e la validazione cross-block semplice, alcuni tag LiquidJS sono esplicitamente vietati nei component:

| Tag | Motivo |
|---|---|
| `{% assign %}` | Crea stato locale nel template — i valori derivati si ottengono con filtri inline |
| `{% capture %}` | Stessa ragione di `{% assign %}` |
| `{% increment %}` / `{% decrement %}` | Stato mutabile — non appartiene a un template dichiarativo |
| `{% render %}` | Bypassa il sistema component — usare `{% component %}` |

Tutti gli altri tag LiquidJS standard sono permessi: `{% if %}`, `{% for %}`, `{% unless %}`, `{% case %}`, `{% break %}`, `{% continue %}`, `{% comment %}`.
