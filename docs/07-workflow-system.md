# Workflow System — LLMNative CMS

## Principio

Il publish pipeline non è hardcoded. È guidato da **workflow configurabili** che definiscono quali agenti usare, con quali parametri, e su quali pagine applicarsi.

Un workflow è un profilo di pubblicazione riutilizzabile. Il crafter può creare più workflow con livelli di permissività diversi e assegnarli a subset di pagine tramite pattern di path.

---

## Struttura di un `PublishWorkflow`

```json
{
  "id": "strict-landing",
  "name": "Strict — Landing Pages",
  "description": "Pipeline completo con soglie alte per le landing page principali",
  "scope": [
    "/landing-pages/**",
    "/home",
    "/contatti"
  ],
  "transform": [
    {
      "agent": "SeoAgent",
      "config": {
        "titleMaxLength": 55,
        "descriptionMaxLength": 150,
        "h1MaxLength": 55,
        "requireAltText": true,
        "autoFixLevel": "conservative",
        "maxRetries": 3
      }
    },
    {
      "agent": "OptimizerAgent",
      "config": {
        "imageQuality": 85,
        "imageFormats": ["webp", "avif"],
        "inlineCriticalCss": true,
        "minifyHtml": true,
        "minifyCss": true,
        "minifyJs": true,
        "maxRetries": 3
      }
    }
  ],
  "validate": {
    "agent": "SentinelAgent",
    "config": {
      "pagespeed": { "min": 95, "blocking": true },
      "seo": { "min": 90, "blocking": true },
      "accessibility": { "min": 85, "blocking": false },
      "maxRetries": 2
    }
  },
  "deploy": {
    "channels": ["prod"],
    "provider": "cloudflare-pages"
  }
}
```

---

## Scope matching con wildcard

Il campo `scope` è un array di pattern. Una pagina viene processata dal workflow il cui scope la intercetta per primo (ordine di priorità = ordine di definizione).

| Pattern | Matches |
|---|---|
| `/home` | Solo `/home` |
| `/blog/**` | Tutto il blog incluse sottocartelle |
| `/landing-pages/*` | Solo il primo livello di `/landing-pages/` |
| `/landing-pages/hero-*` | Tutte le landing che iniziano con `hero-` |
| `/**` | Qualsiasi pagina (wildcard globale) |

### Risoluzione del workflow per una pagina

```
Per ogni pagina da pubblicare:
  1. Scorri i workflow nell'ordine configurato
  2. Il primo workflow il cui scope intercetta il path → viene applicato
  3. Se nessuno intercetta → si applica il workflow "default"
```

---

## Workflow multipli — esempio reale

```json
[
  {
    "id": "strict-landing",
    "name": "Strict — Landing",
    "scope": ["/landing-pages/**", "/home"],
    "...": "soglie alte, agenti aggressivi"
  },
  {
    "id": "standard-blog",
    "name": "Standard — Blog",
    "scope": ["/blog/**"],
    "...": "soglie medie, pipeline bilanciato"
  },
  {
    "id": "fast-news",
    "name": "Fast — News",
    "scope": ["/news/**"],
    "transform": [
      { "agent": "SeoAgent", "config": { "autoFixLevel": "aggressive", "maxRetries": 1 } }
    ],
    "...": "niente OptimizerAgent, Sentinel con soglie basse, pubblica veloce"
  },
  {
    "id": "default",
    "name": "Default",
    "scope": ["/**"],
    "...": "fallback per tutto il resto"
  }
]
```

---

## Il `maxRetries` è condiviso

Il valore `maxRetries` configurato per ogni agente nella sezione `transform` è lo stesso usato:
1. **Dal transform agent internamente** — quanti tentativi fa prima di bloccarsi
2. **Dal SentinelAgent** — quante volte può re-triggerare quell'agente se `validate()` fallisce post-transform

Unica fonte di verità, zero disallineamenti.

---

## Workflow editor (nel CMS admin)

Il crafter configura i workflow tramite un editor visuale nell'area admin:

```
┌─────────────────────────────────────────────┐
│  Workflow: Strict — Landing Pages           │
│                                             │
│  Scope                                      │
│  ┌─────────────────────────────────────┐   │
│  │ /landing-pages/**                   │   │
│  │ /home                        [+ add]│   │
│  └─────────────────────────────────────┘   │
│                                             │
│  Transform Agents  (parallelo)              │
│  ┌──────────────┐  ┌──────────────┐        │
│  │ SeoAgent     │  │ OptimizerAgent│  [+]  │
│  │ [configure]  │  │ [configure]  │        │
│  └──────────────┘  └──────────────┘        │
│                                             │
│  Validate (sequenziale)                     │
│  ┌──────────────────────────────────┐      │
│  │ SentinelAgent  [configure]       │      │
│  └──────────────────────────────────┘      │
│                                             │
│  Deploy                                     │
│  Channels: [prod ✓] [stage] [dev]           │
│  Provider: Cloudflare Pages                 │
└─────────────────────────────────────────────┘
```

---

## Estendibilità

**Aggiungere un agente al workflow** = selezionarlo dal registry degli agenti disponibili e configurarlo. Non richiede codice.

**Aggiungere un nuovo tipo di agente al sistema** = implementare `TransformAgent` e registrarlo. Diventa automaticamente disponibile in tutti i workflow editor.

**Creare un workflow più permissivo** = abbassare le soglie Sentinel, aumentare `maxRetries`, usare `autoFixLevel: "aggressive"`, rimuovere agenti opzionali.

**Creare un workflow zero-tolerance** = soglie Sentinel al 99, `maxRetries: 0` (blocca subito se non è perfetto), `autoFixLevel: "conservative"` (non tocca nulla senza certezza).
