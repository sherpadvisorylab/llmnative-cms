# Template System — LLMNative CMS

## Principio

Il sistema è **data-driven**: sono i dati che definiscono le pagine, non il contrario.
Il template è uno scheletro che sa come presentare i dati — non contiene dati.

## Nomenclatura: Component

L'unità atomica del template system si chiama **Component** (non "component", non "block", non "partial").

**Convenzione di disambiguazione** (solo per chi sviluppa il CMS):
- **Component** — sempre CMS Component (LiquidJS), se non specificato diversamente
- **React component** — quando si parla esplicitamente della parte `@llmnative/react`

`@llmnative/react` è un vendor opaco. Creator, crafter, developer CMS, e AI che usano il CMS conoscono solo i CMS Component. Non devono sapere cosa c'è sotto.

## Template engine: LiquidJS

Scelto per:
- **Sandbox sicuro** — il crafter scrive template senza poter eseguire codice arbitrario
- **Browser + Node.js** — stesso engine per preview in-browser e build engine server-side
- **Pre-compilabile** — i template si compilano una volta sola in AST, poi si eseguono n volte
- **Sintassi leggibile** — `{{ var }}`, `{% if %}`, `{% for %}`, filtri `| upcase`
- **Estendibile** — custom tags per il sistema di composizione component

---

## I due livelli del template system

```
Livello 1 — COMPOSIZIONE (custom, cuore del sistema)
  Come i component si assemblano in una pagina
  Gestito dal builder tramite custom Liquid tags

Livello 2 — SINTASSI (LiquidJS)
  Come dentro un component si esprime la presentazione del dato
  {{ title }}, {% if %}, filtri, ecc.
```

---

## Component

Un component è il componente atomico del sistema. Ogni component:
- Sa come presentare un tipo di dato specifico
- È un file `.liquid` con sintassi LiquidJS
- Riceve i dati del campo a cui è mappato
- Non conosce il contesto della pagina — è isolato

```
components/
  ├── HeadingFrame.liquid
  ├── BodyFrame.liquid
  ├── HeroFrame.liquid
  ├── ImageFrame.liquid
  ├── TagsFrame.liquid
  ├── AuthorFrame.liquid
  └── [custom components del crafter]
```

### Esempio: HeadingFrame.liquid
```liquid
<header class="page-header {{ cssClass }}">
  <h1>{{ title }}</h1>
  {% if subtitle %}
    <p class="subtitle">{{ subtitle }}</p>
  {% endif %}
</header>
```

### Esempio: TagsFrame.liquid
```liquid
{% if tags.size > 0 %}
  <ul class="tags">
    {% for tag in tags %}
      <li class="tag">
        <a href="/tag/{{ tag | slugify }}">{{ tag }}</a>
      </li>
    {% endfor %}
  </ul>
{% endif %}
```

---

## Page Template

Il page template assembla i component in una pagina. È scritto dal crafter usando il custom tag `{% component %}`:

```liquid
{% layout "BaseLayout" %}

{% block content %}
  {% component "HeroFrame"    field: "hero"    %}
  {% component "HeadingFrame" field: "title", subtitle_field: "subtitle" %}
  {% component "BodyFrame"    field: "body"    %}
  {% component "TagsFrame"    field: "tags"    %}
  {% component "AuthorFrame"  field: "author"  %}
{% endblock %}
```

Il custom tag `{% component %}` è implementato dal builder — LiquidJS non lo conosce nativamente. Il builder lo intercetta, risolve il component dal registry, passa i dati del campo, e inietta l'HTML risultante.

---

## Schema → Frame mapping

Il mapping tra campo dello schema e component di default è configurato a livello di schema definition:

```json
{
  "id": "article",
  "fields": [
    { "key": "title",    "type": "text",     "defaultFrame": "HeadingFrame" },
    { "key": "hero",     "type": "image",    "defaultFrame": "HeroFrame"    },
    { "key": "body",     "type": "markdown", "defaultFrame": "BodyFrame"    },
    { "key": "tags",     "type": "tags",     "defaultFrame": "TagsFrame"    },
    { "key": "author",   "type": "relation", "defaultFrame": "AuthorFrame"  }
  ]
}
```

Il `defaultFrame` è il component usato se il crafter non specifica nulla nel page template. Il crafter può sempre sovrascriverlo con un component diverso — o non usarlo affatto e scrivere HTML diretto.

---

## Build flow

```
1. Builder carica il page template (.liquid)
2. Builder carica i dati della pagina (JSON dal data provider)
3. LiquidJS processa il template
4. Quando incontra {% component "HeadingFrame" field: "title" %}:
     a. Builder recupera HeadingFrame.liquid dal component registry
     b. Estrae il valore di `title` dai dati della pagina
     c. Esegue HeadingFrame.liquid con { title: "..." }
     d. Inietta l'HTML risultante nella posizione del tag
5. Continua fino a produrre l'HTML completo della pagina
```

---

## Pre-compilazione per la preview

I template compilati vengono salvati nel data provider insieme al template sorgente:

```json
{
  "id": "article-default",
  "source": "{% layout ... %}{% component ... %}...",
  "compiled": "{ ... AST serializzato ... }",
  "components": {
    "HeadingFrame": { "source": "...", "compiled": "..." },
    "BodyFrame":    { "source": "...", "compiled": "..." }
  },
  "compiledAt": "ISO8601"
}
```

La preview tab riceve i `compiled` — mai le stringhe sorgente. Esegue senza ricompilare.

---

## Layout system

I page template possono estendere layout base — lo stesso meccanismo di Liquid nativo:

```liquid
<!-- layouts/BaseLayout.liquid -->
<!DOCTYPE html>
<html lang="{{ locale }}">
<head>
  <title>{{ seo.title }}</title>
  <meta name="description" content="{{ seo.description }}">
  {% render "JsonLd", schema: jsonld %}
  {{ css_bundle }}
</head>
<body>
  {% render "Header", nav: navigation %}
  <main>
    {% block content %}{% endblock %}
  </main>
  {% render "Footer" %}
  {{ js_bundle }}
</body>
</html>
```

---

## Filtri custom registrati dal sistema

Il builder registra in LiquidJS filtri utili al CMS:

```liquid
{{ body | markdown }}          <!-- markdown → HTML -->
{{ date | cms_date: "it" }}    <!-- formattazione data localizzata -->
{{ image | srcset: [400,800] }}<!-- genera attributo srcset per immagini responsive -->
{{ title | slugify }}          <!-- "Titolo Articolo" → "titolo-articolo" -->
{{ price | currency: "EUR" }}  <!-- 12.5 → "€ 12,50" -->
```
