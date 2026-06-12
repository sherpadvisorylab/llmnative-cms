# LLMNative CMS — Contesto per Claude

## Cos'è questo progetto

Un CMS **static-output, AI-first, data-driven** che produce HTML puro pubblicabile su CDN/VPS.
Il flusso è unidirezionale: `Admin → Build Pipeline → Static HTML ← Web User / LLM Crawlers / Search Engines`.

Nessun codice backend è esposto al navigatore finale. Solo HTML/CSS/JS minimale.

## Repo correlati

- **Admin UI**: `C:\projects\assets\sherpa\llmnative\react` — framework `@llmnative/react` (v0.1.x, Apache-2.0)
  - Usato come fondazione per l'area admin
  - Provider agnostici: data, storage, auth, ai, seo già integrati
  - Componenti chiave: `GridDB`, `Form`, `Upload`, `MarkdownReader`

## Decisioni architetturali prese

1. **Data model agnostico**: il CMS lavora sempre con oggetti JSON strutturati. Il data provider (`@llmnative/react`) gestisce la persistenza — il CMS non sa né gli importa dove vengono salvati i dati.

2. **Output statico puro**: nessun SSR, nessun framework frontend nell'output pubblico. Solo HTML generato dal build engine.

3. **LLM-first by design**: ogni sito prodotto ha un canale dati strutturato dedicato ai crawler AI (es. `llms.txt`, `/data.json`, JSON-LD) generato automaticamente.

4. **Quality gate obbligatorio**: nessuna pagina viene pubblicata se non supera soglie configurabili (PageSpeed ≥ 90, SEO checks, ecc.).

5. **Pipeline di publish con agenti**: al momento del publish partono agenti specializzati (SEO agent, Speed agent, Verifier agent) orchestrati in workflow.

## Struttura del progetto (pianificata)

```
cms/
├── CLAUDE.md              # questo file
├── docs/                  # documentazione di progetto
│   ├── 01-vision.md
│   ├── 02-architecture.md
│   ├── 03-data-model.md
│   ├── 04-actors.md
│   └── 05-publish-pipeline.md
├── packages/
│   ├── admin/             # area admin (usa @llmnative/react)
│   ├── build-engine/      # trasforma JSON → HTML statico
│   ├── agents/            # agenti SEO, Speed, Verifier
│   └── public-schema/     # contratto dati condiviso tra admin e build engine
└── apps/
    └── studio/            # app admin principale
```

## Attori del sistema

| Attore | Ruolo |
|---|---|
| **Crafter / Admin** | Definisce schemi, template, stili, struttura del sito |
| **Creator** | Inserisce contenuti, assistito da AI content assistant + SEO assistant |
| **Web User** | Fruisce l'output statico — esperienza media, non il differenziatore |
| **LLM Crawlers** | Primo cliente del sistema — canale dati strutturato dedicato |
| **Search Engines** | Secondo cliente — HTML ultra-ottimizzato SEO/speed nativo |

## Cosa costruire (macro roadmap)

- [ ] MVP Admin: schema editor + content editor (su `@llmnative/react`)
- [ ] Build Engine: JSON schema + template → HTML statico
- [ ] Canale LLM-first: generazione automatica `llms.txt` / JSON-LD
- [ ] Quality Gate: PageSpeed check pre-publish
- [ ] Publish Pipeline: orchestrazione agenti (SEO, Speed, Verifier)
- [ ] AI Content Assistant: affiancamento creator nella scrittura
- [ ] SEO Orchestrator Assistant: strategia keyword, Google Trends integration

## Convenzioni di nomenclatura

| Termine | Significato |
|---|---|
| **Component** | CMS Component — unità atomica LiquidJS del template system (default, senza qualificazione) |
| **React component** | Componente `@llmnative/react` — solo quando si deve disambiguare esplicitamente |
| **Block** | NON usare — termine scartato in favore di Component |
| **Frame** | NON usare — termine provvisorio sostituito da Component |

`@llmnative/react` è un vendor opaco. Creator, crafter, developer CMS e AI che usano il CMS conoscono solo i CMS Component. Non devono sapere cosa c'è sotto.

## Decisioni tecniche prese

| Decisione | Scelta |
|---|---|
| Template engine | **LiquidJS** — sandboxed, browser+node, pre-compilabile |
| Unit atomica template | **Component** (file `.liquid` in `components/`) |
| Preview | **BroadcastChannel** in-browser, zero server |
| Admin UI foundation | **`@llmnative/react`** (vendor opaco) |
| Agent transform | **SeoAgent**, **OptimizerAgent** (parallelo) |
| Agent validate | **SentinelAgent** (sequenziale, ultimo) |

## Note per sessioni future

- Il framework `@llmnative/react` ha già provider SEO con Google Trends/Ads Keywords integrati
- Il provider AI supporta OpenAI, Anthropic, Gemini, DeepSeek, Mistral — scegliere in fase di implementazione
- Il data model agnostico usa un envelope JSON standard (vedere `docs/03-data-model.md`)
- Ogni decisione architetturale significativa va documentata in `docs/`
- Per il dettaglio completo vedere i file in `docs/` — CLAUDE.md è il sommario
