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
{{-- 1. TEMPLATE (obbligatorio) --}}
<section class="hero hero--{{ layout }}">
  <h1>{{ title }}</h1>
  {% if subtitle %}<p class="hero__subtitle">{{ subtitle }}</p>{% endif %}
  {% if image %}{{ image | img_tag }}{% endif %}
  {% if cta_text %}
    <a class="hero__cta" href="{{ cta_url }}">{{ cta_text }}</a>
  {% endif %}
</section>

{{-- 2. CSS (opzionale) --}}
{% css %}
.hero { padding: 4rem 2rem; background: {{ styles.bg_primary }}; }
.hero--centered { text-align: center; }
.hero--left { text-align: left; }
{% endcss %}

{{-- 3. JS (opzionale) --}}
{% javascript %}
// nessuna regola imposta — il crafter fa come vuole
{% endjavascript %}

{{-- 4. SCHEMA_ORG (opzionale) --}}
{% schema_org %}
{
  "@context": "https://schema.org",
  "@type": "WPHeader",
  "name": "{{ title }}"
}
{% endschema_org %}

{{-- 5. SCHEMA (opzionale — necessario per generare l'edit UI) --}}
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

---

## Il blocco `{% schema %}`

```json
{
  "name":        "Nome leggibile del component",
  "description": "Descrizione opzionale",
  "category":    "layout | content | media | navigation | utility",
  "inputs": {
    "chiave": {
      "type":     "text | textarea | number | boolean | select | image | url | color | date",
      "label":    "Label visibile nel form di edit",
      "required": true,
      "default":  "valore di default",
      "options":  ["opzione1", "opzione2"],
      "showIf":   "espressione condizionale"
    }
  }
}
```

Gli `inputs` guidano due sistemi:
1. **Edit UI**: il builder usa `Component.schema[type](overrides)` per generare il form del creator
2. **Validazione**: il builder verifica che i dati passati al component rispettino il contratto

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
| `&&` | `"show_cta == true && layout != 'grid'"` |
| `\|\|` | `"title != '' \|\| subtitle != ''"` |
| `!` | `"!show_cta"` |

#### Separazione di responsabilità

`showIf` governa la **visibilità nell'edit UI** — non il rendering HTML. Sono due cose distinte e possono avere condizioni diverse:

```liquid
{{-- edit UI: cta_url visibile solo se show_cta è true (via showIf in schema) --}}
{{-- template: rendering condizionale indipendente --}}
{% if show_cta and cta_url != "" %}
  <a class="hero__cta" href="{{ cta_url }}">{{ cta_text }}</a>
{% endif %}
```

---

## CSS e JS

- Embed as-is nel bundle della pagina — nessuna regola di scoping imposta
- Il crafter è responsabile della convenzione che sceglie (BEM, utility classes, ecc.)
- Il builder concatena i blocchi `{% css %}` e `{% javascript %}` di tutti i component usati in una pagina in un unico bundle per pagina
- Il blocco `{% schema_org %}` è un template LiquidJS — riceve le stesse variabili del template principale. Il builder lo inietta come `<script type="application/ld+json">` nell'`<head>`

---

## Tag `{% component %}`

Il custom tag usato per includere un component in un page template o dentro un altro component.

### Sintassi

```liquid
{% component "NomeComponent" %}
{% component "NomeComponent" as: "chiave" %}
{% component "NomeComponent" param: valore %}
```

### Parametro `as`

`as` definisce la chiave sotto cui i dati dell'istanza vengono salvati e recuperati.

- **Omesso**: default al nome del component (`"Hero"`)
- **Esplicito**: necessario quando lo stesso component appare più volte nella stessa pagina

```liquid
{{-- as implicito → chiave "Hero" --}}
{% component "Hero" %}

{{-- as esplicito → necessario per istanze multiple --}}
{% component "FeatureCard" as: "top" %}
{% component "FeatureCard" as: "bottom" %}

{{-- nested editorial → chiave sotto il parent --}}
{% component "AuthorAvatar" as: "author" %}

{{-- structural → parametri diretti, nessun dato salvato --}}
{% component "Divider" style: "dashed" %}
```

### Struttura dati risultante

```json
{
  "Hero":   { "title": "Benvenuto", "layout": "centered" },
  "top":    { "title": "Feature A", "columns": 3 },
  "bottom": { "title": "Feature B", "columns": 2 },
  "card-1": {
    "title": "Articolo",
    "author": { "name": "Mario Rossi", "photo": "...", "role": "Editor" }
  }
}
```

---

## Nesting

I component possono includere altri component a profondità infinita.

### Structural dentro editorial
```liquid
{{-- Hero.liquid --}}
<section class="hero">
  <h1>{{ title }}</h1>
  {% component "Divider" style: "light" %}
</section>
```

### Editorial dentro editorial
```liquid
{{-- ArticleCard.liquid --}}
<article>
  <h2>{{ title }}</h2>
  <p>{{ excerpt }}</p>
  {% component "AuthorAvatar" as: "author" %}
</article>
```

L'edit UI del parent mostra i propri campi + una sezione dedicata per i campi del component figlio.

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
{{ body | markdown }}           {{-- markdown → HTML --}}
{{ date | cms_date: "it" }}     {{-- formattazione data localizzata --}}
{{ image | img_tag }}           {{-- <img> con src, alt, lazy loading --}}
{{ image | srcset: [400,800] }} {{-- attributo srcset per immagini responsive --}}
{{ title | slugify }}           {{-- "Titolo Articolo" → "titolo-articolo" --}}
{{ price | currency: "EUR" }}   {{-- 12.5 → "€ 12,50" --}}
```
