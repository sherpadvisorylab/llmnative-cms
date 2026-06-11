# Architettura — LLMNative CMS

## Macro architettura

Il sistema è diviso in due aree completamente segregate:

```
┌─────────────────────────────────────────────────────┐
│                    ADMIN AREA                        │
│                                                      │
│  Studio (React App)                                  │
│  └── @llmnative/react                               │
│       ├── Data Provider (Firebase/Supabase/custom)  │
│       ├── Auth Provider                             │
│       ├── AI Provider (content + SEO assistant)     │
│       └── SEO Provider (Google Trends/Ads)          │
└──────────────────────┬──────────────────────────────┘
                       │
                  JSON Data
                       │
┌──────────────────────▼──────────────────────────────┐
│                 BUILD PIPELINE                       │
│                                                      │
│  Build Engine                                        │
│  ├── legge JSON dal data provider                    │
│  ├── applica template (definiti dai crafter)         │
│  └── produce HTML statico                            │
│                                                      │
│  Agent Workflow (al publish)                         │
│  ├── SEO Agent                                       │
│  ├── Speed Agent                                     │
│  ├── Verifier Agent                                  │
│  └── Quality Gate (blocca se sotto soglia)           │
└──────────────────────┬──────────────────────────────┘
                       │
                  Static Files
                       │
┌──────────────────────▼──────────────────────────────┐
│                  OUTPUT PUBBLICO                     │
│                                                      │
│  /index.html, /page/*.html                           │
│  /assets/*.css, /assets/*.js (minimale)             │
│  /llms.txt                                           │
│  /data.json                                          │
│  /sitemap.xml                                        │
│  /.well-known/schema.json                            │
└─────────────────────────────────────────────────────┘
```

## Principi architetturali

### 1. Segregazione totale admin/public
L'area admin e l'output pubblico non condividono codice a runtime. Il build engine è il ponte one-way.

### 2. Data provider agnostico
Il CMS lavora con oggetti JSON. Il data provider è un dettaglio di implementazione iniettato a runtime via `@llmnative/react`. Il build engine legge sempre lo stesso contratto JSON, indipendentemente da dove i dati sono persistiti.

### 3. Schema-driven UI
Nell'admin, i crafter definiscono schemi di dati. L'UI per i creator emerge automaticamente dallo schema — non si costruisce a mano ogni form.

### 4. Build on demand + quality gate
Il publish non è automatico al salvataggio. Il creator/crafter sceglie quando pubblicare (ora o schedulato). Il publish trigggera il pipeline agenti. Se il quality gate fallisce, la pubblicazione è bloccata con report dettagliato.

### 5. LLM-first output
Ogni build genera automaticamente, oltre all'HTML:
- `llms.txt` — descrizione del sito per AI crawlers (standard emergente)
- `/data.json` — dump strutturato dei contenuti
- JSON-LD inline nelle pagine (Schema.org)
- Open Graph + Twitter Card meta

## Package structure

```
cms/
├── packages/
│   ├── public-schema/     # tipi TypeScript condivisi (contratto dati)
│   ├── admin/             # configurazione studio admin
│   ├── build-engine/      # core: JSON → HTML
│   └── agents/            # agenti specializzati
│       ├── seo-agent/
│       ├── speed-agent/
│       └── verifier-agent/
└── apps/
    └── studio/            # app admin Vite+React
```

## Stack tecnologico

| Layer | Tecnologia | Motivazione |
|---|---|---|
| Admin UI | `@llmnative/react` + Vite + React 18 | Già integra provider agnostici, AI, SEO |
| Build Engine | Node.js | Processo standalone, nessuna dipendenza frontend |
| Template Engine | TBD (da decidere) | Da valutare: Nunjucks, Handlebars, o custom |
| Agent Orchestration | TBD | Da valutare: LangGraph, custom workflow, Inngest |
| Output Hosting | CDN / VPS | Statico — qualsiasi provider |
| AI Provider | Anthropic Claude (default) | Già supportato in `@llmnative/react` |

## Decisioni aperte (da risolvere)

- [ ] Template engine per il build engine
- [ ] Orchestratore degli agenti (LangGraph vs custom vs Inngest)
- [ ] Formato persistenza schema crafter (JSON nel data provider o file system)
- [ ] Strategia deploy output statico (GitHub Actions? CI/CD integrato?)
