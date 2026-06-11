# Data Model — LLMNative CMS

## Principio

Il CMS è **data-driven**: sono i dati che definiscono le pagine, non il contrario.
Il layer di persistenza è agnostico — il sistema lavora sempre con oggetti JSON.

## L'envelope standard

Ogni entità gestita dal CMS usa questo envelope:

```json
{
  "_type": "article",
  "_schema": "v1",
  "_id": "uuid-or-slug",
  "_meta": {
    "slug": "my-article-slug",
    "locale": "it",
    "status": "draft | published | scheduled",
    "publishAt": null,
    "createdAt": "ISO8601",
    "updatedAt": "ISO8601",
    "createdBy": "user-id",
    "updatedBy": "user-id"
  },
  "_seo": {
    "title": "...",
    "description": "...",
    "keywords": ["..."],
    "ogImage": "url",
    "canonicalUrl": "url",
    "noIndex": false,
    "schemaOrg": {}
  },
  "_build": {
    "template": "template-id",
    "lastBuiltAt": "ISO8601",
    "lastPublishedAt": "ISO8601",
    "qualityScore": {
      "pagespeed": 95,
      "seo": 98,
      "accessibility": 92
    }
  },
  "fields": {
    // contenuto definito dal crafter nello schema
  }
}
```

### Campi riservati (prefisso `_`)
I campi con prefisso `_` sono gestiti dal sistema e non modificabili direttamente dal creator.

### `fields`
Contiene il contenuto effettivo della pagina. La struttura è definita dal crafter tramite lo schema del content type.

---

## Schema definition (definito dai crafter)

Un crafter definisce i content type tramite uno schema:

```json
{
  "id": "article",
  "version": "v1",
  "label": "Articolo",
  "icon": "file-text",
  "template": "article-default",
  "fields": [
    {
      "key": "title",
      "type": "text",
      "label": "Titolo",
      "required": true,
      "ai": { "assist": true, "seoTarget": "h1" }
    },
    {
      "key": "body",
      "type": "markdown",
      "label": "Contenuto",
      "required": true,
      "ai": { "assist": true }
    },
    {
      "key": "coverImage",
      "type": "image",
      "label": "Immagine copertina",
      "required": false
    },
    {
      "key": "tags",
      "type": "tags",
      "label": "Tag",
      "required": false
    }
  ]
}
```

### Tipi di campo supportati
| Tipo | Descrizione |
|---|---|
| `text` | Testo semplice, singola riga |
| `textarea` | Testo lungo, multi-riga |
| `markdown` | Contenuto rich text in Markdown |
| `richtext` | Contenuto WYSIWYG |
| `number` | Numerico |
| `boolean` | Toggle on/off |
| `date` | Data |
| `datetime` | Data e ora |
| `image` | Immagine con upload |
| `file` | File generico |
| `tags` | Lista di tag |
| `select` | Selezione da lista fissa |
| `relation` | Relazione ad altro content type |
| `repeat` | Array di oggetti (campo ripetibile) |
| `object` | Oggetto annidato |

---

## Canale LLM-first

Ogni sito prodotto espone automaticamente:

### `/llms.txt`
```
# Site: Nome Sito
# Description: descrizione del sito
# Language: it
# Last-Updated: ISO8601

## Content Types
- article: /data/articles.json
- product: /data/products.json

## Site Map
- /: Home
- /blog: Blog
...
```

### `/data/{type}.json`
Array di tutti i record pubblicati di quel tipo, senza HTML, solo dati strutturati:
```json
{
  "type": "article",
  "count": 42,
  "updatedAt": "ISO8601",
  "items": [
    {
      "id": "...",
      "slug": "...",
      "title": "...",
      "description": "...",
      "tags": ["..."],
      "url": "/blog/my-article",
      "publishedAt": "ISO8601"
    }
  ]
}
```

### JSON-LD inline (per pagina)
Ogni pagina HTML include `<script type="application/ld+json">` con lo schema Schema.org appropriato per il content type.

---

## Regole del data model

1. I dati non hanno mai dipendenze dal template — il template è separato
2. Il `_build.template` è un riferimento, non un embed
3. I campi `_seo` sono sempre presenti e popolati (dal SEO agent se non manualmente)
4. Lo `slug` è immutabile dopo la prima pubblicazione (per preservare URL)
5. Il `status` ha sempre una transizione definita: `draft → published`, `draft → scheduled → published`
