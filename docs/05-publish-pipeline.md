# Publish Pipeline — LLMNative CMS

## Overview

Il publish non è un semplice "salva e vai live". È un **pipeline orchestrato** che garantisce qualità prima di rendere la pagina pubblica.

```
Creator clicca "Publish"
         │
         ▼
  ┌─────────────┐
  │  Scheduler  │  now / scheduled datetime
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │ Build Engine│  JSON → HTML statico
  └──────┬──────┘
         │
         ├──────────────────────┐
         ▼                      ▼
  ┌─────────────┐       ┌─────────────┐
  │  SEO Agent  │       │ Speed Agent │
  └──────┬──────┘       └──────┬──────┘
         │                      │
         └──────────┬───────────┘
                    ▼
           ┌─────────────────┐
           │ Verifier Agent  │
           │ (quality gate)  │
           └────────┬────────┘
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
   [PASS: pubblica]     [FAIL: blocca]
         │                     │
         ▼                     ▼
   Output su CDN/VPS    Report al creator
```

---

## Fasi del pipeline

### Fase 1 — Trigger

Il creator sceglie:
- **Publish Now**: avvia subito il pipeline
- **Schedule**: mette in coda per una data/ora specifica
- **Save Draft**: salva senza avviare nulla

### Fase 2 — Build Engine

Legge dal data provider i JSON dell'entità da pubblicare e applica il template definito dal crafter.

Output:
- `/{slug}/index.html` — pagina HTML principale
- Aggiornamento di `/data/{type}.json`
- Aggiornamento di `/llms.txt`
- Aggiornamento di `/sitemap.xml`

Il build engine **non pubblica ancora** — produce file in una staging area.

### Fase 3 — Agenti in parallelo

Partono in parallelo su i file nella staging area:

#### SEO Agent
- Analizza il contenuto HTML prodotto
- Verifica: title tag, meta description, H1 unico, struttura heading, alt image, link interni
- Verifica coerenza con le keyword target (definite dal SEO Orchestrator)
- Verifica JSON-LD corretto e completo
- Produce un `seo_report.json` con score e issues

#### Speed Agent
- Analizza il bundle HTML/CSS/JS
- Esegue PageSpeed Insights API (o equivalente)
- Verifica: immagini ottimizzate, CSS minificato, JS minimale, lazy loading, cache headers
- Può applicare fix automatici (minificazione, ottimizzazione immagini)
- Produce un `speed_report.json` con score e issues

### Fase 4 — Verifier Agent (Quality Gate)

Riceve i report da SEO Agent e Speed Agent e applica le soglie configurate:

```json
{
  "qualityGate": {
    "pagespeed": { "min": 90, "blocking": true },
    "seo": { "min": 85, "blocking": true },
    "accessibility": { "min": 80, "blocking": false }
  }
}
```

- `blocking: true` — se sotto soglia, la pubblicazione è bloccata
- `blocking: false` — se sotto soglia, la pubblicazione procede con warning

Output: `verification_result.json` con `status: "pass" | "fail"` e lista issues.

### Fase 5 — Publish o Block

**Se PASS:**
- I file dalla staging area vengono copiati nell'output pubblico
- Il `_build.qualityScore` dell'entità viene aggiornato nel data provider
- Il `_meta.status` diventa `published`
- Il `_build.lastPublishedAt` viene aggiornato

**Se FAIL:**
- I file staging vengono eliminati
- Il creator riceve un report dettagliato con:
  - Score ottenuto vs soglia richiesta
  - Lista degli issues specifici da correggere
  - Suggerimenti di fix (dove possibile)
- Il `_meta.status` rimane `draft` o `scheduled`

---

## Configurazione del Quality Gate

Configurata dal crafter per sito, con possibilità di override per content type:

```json
{
  "siteDefaults": {
    "pagespeed": { "min": 90, "blocking": true },
    "seo": { "min": 85, "blocking": true },
    "accessibility": { "min": 80, "blocking": false }
  },
  "overrides": {
    "landing-page": {
      "pagespeed": { "min": 95, "blocking": true }
    }
  }
}
```

Il crafter può anche **bypassare il quality gate** per una pubblicazione specifica, con log obbligatorio del motivo.

---

## Orchestrazione agenti — opzioni tecniche (da decidere)

| Opzione | Pro | Contro |
|---|---|---|
| **Custom workflow** (Node.js) | Zero dipendenze, controllo totale | Da costruire da zero |
| **Inngest** | Event-driven, retry nativi, UI di monitoring | Dipendenza esterna, costi |
| **LangGraph** | Nativamente pensato per agenti AI | Overkill per fasi non-AI |
| **BullMQ** (Redis) | Job queue robusto, scheduling nativo | Richiede Redis infrastruttura |

**Raccomandazione**: iniziare con custom workflow Node.js per MVP, valutare Inngest o BullMQ quando il volume di publish lo richiede.

---

## Estensibilità

Il pipeline è progettato per essere estensibile. Nuovi agenti possono essere aggiunti dal crafter come step opzionali:

- **Accessibility Agent** — verifica WCAG
- **Content Quality Agent** — verifica grammatica, leggibilità, plagiarism
- **Brand Voice Agent** — verifica coerenza con il tono di voce del brand
- **Link Checker Agent** — verifica link non rotti
- **Security Scanner** — verifica header di sicurezza nell'HTML
