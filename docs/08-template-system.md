# Template System — LLMNative CMS

## Principio

Il sistema è **data-driven**: sono i dati che definiscono le pagine, non il contrario.
Il template è uno scheletro che sa come presentare i dati — non contiene dati.

## Template engine: LiquidJS

Scelto per:
- **Sandbox sicuro** — il crafter scrive template senza poter eseguire codice arbitrario
- **Browser + Node.js** — stesso engine per preview in-browser e build engine server-side
- **Pre-compilabile** — i template si compilano una volta sola in AST, poi si eseguono n volte
- **Sintassi leggibile** — `{{ var }}`, `{% if %}`, `{% for %}`, filtri `| upcase`
- **Estendibile** — custom tags per il sistema di composizione frame

---

## I due livelli del template system

```
Livello 1 — COMPOSIZIONE (custom, cuore del sistema)
  Come i frame si assemblano in una pagina
  Gestito dal builder tramite custom Liquid tags

Livello 2 — SINTASSI (LiquidJS)
  Come dentro un frame si esprime la presentazione del dato
  {{ title }}, {% if %}, filtri, ecc.
```

---

## Frame

Un frame è il componente atomico del sistema. Ogni frame:
- Sa come presentare un tipo di dato specifico
- È un file `.liquid` con sintassi LiquidJS
- Riceve i dati del campo a cui è mappato
- Non conosce il contesto della pagina — è isolato

```
frames/
  ├── HeadingFrame.liquid
  ├── BodyFrame.liquid
  ├── HeroFrame.liquid
  ├── ImageFrame.liquid
  ├── TagsFrame.liquid
  ├── AuthorFrame.liquid
  └── [custom frames del crafter]
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

Il page template assembla i frame in una pagina. È scritto dal crafter usando il custom tag `{% frame %}`:

```liquid
{% layout "BaseLayout" %}

{% block content %}
  {% frame "HeroFrame"    field: "hero"    %}
  {% frame "HeadingFrame" field: "title", subtitle_field: "subtitle" %}
  {% frame "BodyFrame"    field: "body"    %}
  {% frame "TagsFrame"    field: "tags"    %}
  {% frame "AuthorFrame"  field: "author"  %}
{% endblock %}
```

Il custom tag `{% frame %}` è implementato dal builder — LiquidJS non lo conosce nativamente. Il builder lo intercetta, risolve il frame dal registry, passa i dati del campo, e inietta l'HTML risultante.

---

## Schema → Frame mapping

Il mapping tra campo dello schema e frame di default è configurato a livello di schema definition:

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

Il `defaultFrame` è il frame usato se il crafter non specifica nulla nel page template. Il crafter può sempre sovrascriverlo con un frame diverso — o non usarlo affatto e scrivere HTML diretto.

---

## Build flow

```
1. Builder carica il page template (.liquid)
2. Builder carica i dati della pagina (JSON dal data provider)
3. LiquidJS processa il template
4. Quando incontra {% frame "HeadingFrame" field: "title" %}:
     a. Builder recupera HeadingFrame.liquid dal frame registry
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
  "source": "{% layout ... %}{% frame ... %}...",
  "compiled": "{ ... AST serializzato ... }",
  "frames": {
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
