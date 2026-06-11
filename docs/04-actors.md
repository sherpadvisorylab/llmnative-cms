# Attori del sistema — LLMNative CMS

## Overview

```
                    ┌─────────────┐
                    │   CRAFTER   │  definisce strutture
                    │   (admin)   │  schemi, template, stili
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   CREATOR   │  inserisce contenuti
                    │             │◄── AI Content Assistant
                    └──────┬──────┘◄── SEO Orchestrator
                           │
                    [Build Pipeline]
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │  Web User   │  │LLM Crawlers │  │   Search    │
   │   (umano)   │  │  (AI first) │  │   Engines   │
   └─────────────┘  └─────────────┘  └─────────────┘
```

---

## Crafter (Admin)

### Chi è
Il progettista del sito. Tipicamente uno sviluppatore, un designer tecnico, o un web agency che configura il CMS per il cliente.

### Cosa fa
- Definisce i **content type** (schemi dati)
- Crea e gestisce i **template** (come i dati vengono renderizzati in HTML)
- Configura **stili** (CSS, tema)
- Definisce **workflow** di pubblicazione e quality gate
- Gestisce **utenti** e permessi (chi può fare cosa)
- Configura **integrazioni** (data provider, storage, AI provider)
- Imposta la **strategia SEO globale** del sito

### Permessi
- Accesso completo all'area admin
- Può modificare schemi (operazione distruttiva — richiede conferma)
- Può pubblicare qualsiasi contenuto
- Può bypassare il quality gate (con log esplicito)

### Strumenti dedicati nell'admin
- Schema Editor (visual o JSON)
- Template Editor
- Style Manager
- User Management
- Site Configuration
- Quality Gate Settings

---

## Creator (Content Editor)

### Chi è
Chi produce i contenuti del sito. Può essere un copywriter, un marketing manager, o il cliente finale.

### Cosa fa
- Inserisce e modifica **contenuti** nei content type definiti dal crafter
- Usa l'**AI Content Assistant** (opzionale, parziale, o completo)
- Fa **review** dei contenuti generati dall'AI — ha sempre il pieno controllo
- Sceglie quando **pubblicare** (ora o schedulato)
- Vede il **report del quality gate** e può correggere prima della pubblicazione

### Permessi
- Accesso limitato alla gestione contenuti
- Non può modificare schemi o template
- Non può modificare configurazioni globali
- Può pubblicare solo i propri contenuti (configurabile)

### AI Content Assistant
Il creator è affiancato da due assistenti AI coordinati:

**1. SEO Orchestrator Assistant**
- Opera a livello di sito (non di singola pagina)
- Analizza la strategia di comunicazione del sito
- Recupera keyword intent da Google Trends, Google Ads Keywords
- Produce **linee guida SEO** per ogni pagina/tipo di contenuto
- Passa le linee guida all'AI Content Assistant come contesto

**2. AI Content Assistant**
- Opera a livello di singola pagina
- Riceve le linee guida dal SEO Orchestrator
- Conosce lo schema del content type (sa cosa deve produrre)
- Può operare in tre modalità:
  - **Full auto**: genera tutto il contenuto, il creator fa solo review
  - **Partial**: suggerisce mentre il creator scrive (completamento, miglioramento)
  - **On demand**: il creator chiede aiuto puntualmente
- Il creator può **accettare, modificare, o rifiutare** ogni suggerimento
- Ogni modifica manuale del creator sovrascrive il suggerimento AI

---

## Web User (Utente finale)

### Chi è
Il navigatore che fruisce il sito pubblico.

### Cosa vede
Solo HTML statico. Nessuna interazione con il CMS.

### Note
Questo attore **non è il differenziatore** del sistema. L'esperienza utente è nella media del mercato. Il focus è sulla qualità tecnica dell'output (velocità, accessibilità), non su interattività avanzata.

---

## LLM Crawlers (Primo cliente)

### Chi sono
Sistemi AI (Anthropic Claude, OpenAI GPT, Perplexity, ecc.) che crawlano il web per costruire knowledge base o rispondere a query degli utenti.

### Cosa ricevono
Un canale dati dedicato, strutturato, senza rumore HTML:
- `/llms.txt` — descrizione del sito in formato testo per AI
- `/data/{type}.json` — dati strutturati per tipo di contenuto
- JSON-LD inline in ogni pagina

### Perché sono il primo cliente
Il web si sta spostando verso una fruizione mediata da AI. Essere facilmente comprensibili dai sistemi AI aumenta la visibilità e la citabilità del sito.

---

## Search Engines (Secondo cliente)

### Chi sono
Google, Bing, e altri motori di ricerca.

### Cosa ricevono
- HTML semanticamente corretto
- PageSpeed ≥ soglia configurata (default 90)
- `sitemap.xml` aggiornata
- Meta SEO completi (title, description, OG, Twitter Card)
- Schema.org JSON-LD appropriato per ogni content type
- URL canonici corretti
- Nessun JavaScript client-side che blocca il rendering

### Quality gate
Il sistema **blocca la pubblicazione** se le metriche non superano la soglia. Non si può pubblicare una pagina lenta o con SEO carente per negligenza.
