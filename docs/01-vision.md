# Vision — LLMNative CMS

## Il problema che risolviamo

I CMS moderni sono ottimizzati per l'esperienza dello human editor. Il risultato è un web pieno di pagine lente, SEO improvvisata, e dati strutturati male — illeggibili sia per i motori di ricerca che per i sistemi AI che oggi crawlano il web.

## La nostra risposta

Un CMS che ha **tre clienti primari** in ordine di priorità:

1. **LLM Crawlers** — sistemi AI che cercano informazioni nel web. Vogliamo offrire loro un canale dati pulito, strutturato, semanticamente ricco.
2. **Search Engines** — Google e altri. Output HTML ultra-ottimizzato per SEO tecnica e velocità, con quality gate automatici.
3. **Web Users** — l'utente umano finale. Esperienza nella media del mercato, non è il nostro differenziatore.

## Il flusso

```
[Admin Area]
     │
     │  crafter definisce schemi + template
     │  creator inserisce contenuti (assistito da AI)
     ▼
[Build Pipeline]
     │
     │  agenti: SEO · Speed · Verifier
     │  quality gate: pubblica solo se supera soglie
     ▼
[Output Statico]
  HTML puro + CSS minimale + JS minimale
     │
     ├──► CDN / VPS  ◄── Web User
     ├──► LLM Crawlers (canale /data.json, llms.txt, JSON-LD)
     └──► Search Engines (sitemap, schema.org, performance ≥ 90)
```

## Il principio fondante

> Quando il navigatore accede al sito, non c'è nessun codice dietro. Solo HTML.

Questo non è un vincolo tecnico — è una scelta di design deliberata che porta:
- **Sicurezza**: zero attack surface, nessun DB esposto
- **Performance**: nessun TTFB da server rendering, CDN-native
- **Scalabilità**: costo di hosting quasi zero
- **Resilienza**: il sito vive anche se l'admin è offline

## Differenziatori reali vs mercato esistente

| Caratteristica | Mercato attuale | LLMNative CMS |
|---|---|---|
| LLM-first channel | Plugin/addon esterno | Nativo, parte del build |
| Quality gate publish | Manuale / assente | Automatico, agenti, configurabile |
| SEO tecnica | Configurazione manuale | Garantita strutturalmente |
| AI content assistant | Wrapper ChatGPT | Contestualizzato su schema + strategia SEO |
| Output | SSR o SPA | HTML puro, zero framework |
| Orientamento | Human editor first | AI/machine first, human second |

## Posizionamento di mercato

**Target primario**: PMI e agenzie web che vogliono:
- Presenza AI-ready (visibili ai crawler LLM)
- SEO tecnica garantita senza consulente
- Hosting low-cost (statico su CDN)
- Produzione contenuti assistita da AI

**Non competiamo con**: Contentful (enterprise), WordPress (massa), Webflow (no-code visual).

**Competiamo nello spazio**: "digital-first SMB che vuole qualità garantita e AI-readiness senza complessità operativa".
