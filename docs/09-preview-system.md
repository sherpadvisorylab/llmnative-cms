# Preview System — LLMNative CMS

## Principio

La preview è **completamente in-browser, in memoria, senza server**.
Funziona tramite comunicazione diretta tra la tab admin e una tab preview dedicata, usando la BroadcastChannel API nativa del browser.

Il creator vede le modifiche in tempo reale senza refresh, su una pagina navigabile che rispecchia fedelmente l'output finale.

---

## Tre modalità di preview

### Instant
Attiva mentre il creator digita. Debounce di ~300ms dopo l'ultimo keystroke.
Solo il campo in modifica viene aggiornato nel DOM della preview — non l'intera pagina.
```
Creator scrive "Titolo Art..." → 300ms → preview aggiorna h1
Creator continua "icolo..." → 300ms → preview aggiorna h1
```

### Draft
Attiva quando il creator clicca "Salva bozza". Aggiorna l'intera pagina con lo snapshot completo di tutti i campi salvati.

### Full
Attiva quando il creator clicca "Anteprima completa". Trigggera il pipeline agenti in background (senza deploy). La preview mostra la pagina con un overlay dei quality score:
```
SEO: 92 ✓ | PageSpeed: 87 ⚠ | H1 troppo lungo (68 car > 60)
```
È l'unica modalità che richiede elaborazione server-side.

---

## Architettura

```
Data Provider
      │
      │ al caricamento dell'admin
      ▼
Admin Tab (memoria)
  ├── compiled templates (AST LiquidJS)
  ├── pages data (JSON)
  └── site structure (slug → template mapping)
      │
      │ window.open() + BroadcastChannel "init"
      ▼
Preview Tab (memoria)
  ├── site store (autonomo)
  ├── LiquidJS (libreria caricata una volta)
  └── morphdom (DOM diffing)
```

La preview tab **non accede mai al data provider** — riceve tutto dall'admin tab che ha già il dato in memoria.

---

## Flusso di inizializzazione

```
1. Creator clicca "Apri Preview"
2. Admin tab: window.open('/preview') → si apre nuova tab
3. Preview tab: carica LiquidJS + morphdom + preview-runtime.js
4. Preview tab: postMessage({ type: "ready" })
5. Admin tab: riceve "ready"
6. Admin tab: postMessage({ type: "init", payload })
7. Preview tab: popola site store, renderizza pagina corrente
```

### Payload "init"
```json
{
  "type": "init",
  "currentSlug": "/blog/articolo-1",
  "templates": {
    "article-default": { "...AST LiquidJS serializzato..." },
    "landing-default": { "...AST LiquidJS serializzato..." }
  },
  "pages": {
    "/blog/articolo-1": { "template": "article-default", "data": { "title": "...", "body": "..." } },
    "/blog/articolo-2": { "template": "article-default", "data": { "..." } },
    "/home":            { "template": "landing-default", "data": { "..." } }
  },
  "assets": {
    "css": "...bundle CSS del sito...",
    "js":  "...bundle JS minimale del sito..."
  }
}
```

---

## Tipi di messaggi BroadcastChannel

Il canale trasporta **solo JSON** — mai codice JS grezzo, mai `eval()`.

### content-update
Inviato ad ogni modifica del creator (instant) o al salvataggio bozza (draft).
```json
{
  "type": "content-update",
  "slug": "/blog/articolo-1",
  "data": {
    "title": "Titolo aggiornato",
    "body": "Testo modificato..."
  }
}
```
Preview riceve → `templateFn(nuoviDati)` → `morphdom(document.body, html)`

### template-update
Inviato quando il crafter modifica e salva un template mentre la preview è aperta.
Il data provider notifica l'admin tab (real-time subscription), che aggiorna la memoria e propaga alla preview.
```json
{
  "type": "template-update",
  "templateId": "article-default",
  "compiled": { "...nuovo AST LiquidJS serializzato..." }
}
```
Preview riceve → aggiorna template nel site store → ri-renderizza pagina corrente.

### navigate
Inviato quando l'admin vuole portare la preview su una pagina specifica.
```json
{
  "type": "navigate",
  "slug": "/blog/altro-articolo"
}
```

---

## Rendering in preview tab

```javascript
// Site store in memoria
const store = {
  templates: {},   // { templateId: compiledTemplateFn }
  pages: {},       // { slug: { template, data } }
  currentSlug: ''
}

// Rendering di una pagina
function render(slug) {
  const page = store.pages[slug]
  const templateFn = store.templates[page.template]
  const html = templateFn(page.data)
  morphdom(document.documentElement, html)
}

// BroadcastChannel
channel.onmessage = ({ data }) => {
  switch (data.type) {
    case 'init':
      initStore(data.payload)
      render(data.payload.currentSlug)
      break

    case 'content-update':
      store.pages[data.slug].data = data.data
      if (data.slug === store.currentSlug) render(data.slug)
      break

    case 'template-update':
      store.templates[data.templateId] = liquid.template(data.compiled)
      render(store.currentSlug)
      break

    case 'navigate':
      store.currentSlug = data.slug
      render(data.slug)
      break
  }
}
```

---

## Navigazione nella preview tab

La preview tab intercetta i click sui link interni e li risolve dal site store — senza navigazione reale del browser:

```javascript
document.addEventListener('click', (e) => {
  const link = e.target.closest('a')
  if (!link) return

  const url = new URL(link.href)
  if (url.origin !== location.origin) return  // link esterni: comportamento normale

  e.preventDefault()
  const slug = url.pathname

  if (store.pages[slug]) {
    store.currentSlug = slug
    render(slug)
    history.pushState({}, '', slug)  // aggiorna URL nella barra del browser
  } else {
    // pagina non ancora in store → mostra placeholder
    showPlaceholder(slug)
  }
})
```

Il creator può navigare tra tutte le pagine del sito nella preview tab — ogni click è istantaneo perché tutti i dati sono già in memoria.

---

## Template compilati: dove vivono

```
Crafter salva template
    └── Builder compila: source → AST LiquidJS
    └── Salva nel data provider:
          {
            "id": "article-default",
            "source": "{% layout ... %}...",
            "compiled": { ...AST... },
            "compiledAt": "ISO8601"
          }

Creator apre admin
    └── Admin carica dal data provider i compiled di tutti i template
    └── Li tiene in memoria (mai ri-compila, usa il compiled salvato)

Creator apre preview
    └── Admin invia i compiled via BroadcastChannel "init"
    └── Preview tab ricostruisce le funzioni con liquid.template(compiled)
    └── Da quel momento esegue solo, senza mai ricompilare
```

---

## Cosa la preview NON fa

- Non accede al data provider direttamente
- Non esegue gli agenti (SeoAgent, OptimizerAgent, SentinelAgent)
- Non ottimizza immagini o asset
- Non esegue codice JS arbitrario ricevuto via canale
- Non produce output pubblicabile

È esclusivamente uno strumento di visualizzazione fedele — quello che vedi è la struttura e il contenuto finale, non la qualità tecnica ottimizzata.
