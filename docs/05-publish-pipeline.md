# Publish Pipeline — LLMNative CMS

## Overview

Il publish è un pipeline orchestrato a stato che garantisce qualità prima di rendere le pagine pubbliche. È basato su una **job queue** persistita nel data provider, dove ogni riga rappresenta una singola pagina da pubblicare.

---

## Entità della queue

### `PublishJob`
Rappresenta una singola operazione di publish avviata dall'utente.

```json
{
  "id": "job-uuid",
  "triggeredBy": "user-id",
  "triggeredAt": "ISO8601",
  "workflowId": "strict-landing",
  "status": "running | completed | partial",
  "stats": {
    "total": 10,
    "validated": 9,
    "published": 9,
    "blocked": 1
  }
}
```

### `PublishTask`
Una riga = una pagina. Ogni task traccia il proprio stato e il progresso per agente.

```json
{
  "id": "task-uuid",
  "jobId": "job-uuid",
  "pageId": "page-ref",
  "slug": "/blog/my-article",
  "workflowId": "strict-landing",
  "status": "QUEUED | TRANSFORMING | TRANSFORM_REVIEW | TRANSFORM_DONE | VALIDATING | VALIDATED | VALIDATION_BLOCKED | DEPLOYING | DEPLOYED | DEPLOY_FAILED",
  "retryCount": 0,
  "agents": {
    "SeoAgent": {
      "status": "pending | running | done | blocked",
      "retries": 0,
      "report": {}
    },
    "OptimizerAgent": {
      "status": "pending | running | done | blocked",
      "retries": 0,
      "report": {}
    },
    "SentinelAgent": {
      "status": "pending | running | done | blocked",
      "retries": 0,
      "report": {}
    }
  },
  "reviewItems": [],
  "createdAt": "ISO8601",
  "updatedAt": "ISO8601"
}
```

---

## State machine del `PublishTask`

```
QUEUED
  └─► TRANSFORMING
        [SeoAgent.transform() ∥ OptimizerAgent.transform() ∥ ...]
        Ogni agente si auto-valida internamente prima di segnare done.
        │
        ├─ tutti gli agenti → done ──────────────► TRANSFORM_DONE
        └─ uno o più agenti → blocked ───────────► TRANSFORM_REVIEW
                                                    (attende azione umana)

TRANSFORM_DONE
  └─► VALIDATING
        [SentinelAgent.run()]
        Chiama validate() di ogni transform agent + check esclusivi.
        │
        ├─ tutto ok ─────────────────────────────► VALIDATED
        ├─ fail + retryCount < maxRetries ───────► re-trigger agenti target
        │                                           └─► TRANSFORMING (agenti specifici)
        └─ fail + retryCount >= maxRetries ──────► VALIDATION_BLOCKED
                                                    (attende azione umana)

VALIDATED
  └─► (raccolto dal PublisherAgent insieme agli altri VALIDATED)
        └─► DEPLOYING
              ├─ ok ──► DEPLOYED
              └─ fail ─► DEPLOY_FAILED → (review umano)
```

---

## Ciclo interno di ogni TransformAgent

Prima di segnare il proprio task come `done`, ogni transform agent esegue il proprio validatore interno:

```
transform(page, config)
  └─► validate(page, config)
        ├─ pass ──► segna done, emette report
        └─ fail ──► retry transform (fino a config.maxRetries)
                      └─ ancora fail dopo maxRetries ──► segna blocked, popola reviewItems
```

Il `maxRetries` è definito una volta nel workflow config e condiviso tra il ciclo interno dell'agente e i retry del SentinelAgent.

---

## SentinelAgent

Non contiene regole proprie hardcoded. Funziona come **orchestratore di validatori**:

```
SentinelAgent.run(page, workflow)
  ├─► Per ogni transform agent nel workflow:
  │     agent.validate(page, config)  ← stessa funzione usata dall'agente internamente
  │
  ├─► SentinelAgent.exclusiveChecks(page)
  │     (PageSpeed API, lighthouse, check esterni, ecc.)
  │
  └─► Aggrega tutti i report
        ├─ tutti pass ──► emette VALIDATED
        └─ qualcuno fail:
             ├─ identifica gli agenti responsabili del fail
             ├─ se retryCount < maxRetries ──► re-trigger agenti target
             └─ se retryCount >= maxRetries ──► emette VALIDATION_BLOCKED
```

Il retry target è deterministico: se `SeoAgent.validate()` fallisce → si ri-triggera `SeoAgent.transform()`. Mapping 1:1.

---

## PublisherAgent

Stateless per design. Non ha memoria dei job precedenti.

```
PublisherAgent.run(job)
  ├─► Raccoglie tutti i PublishTask con status VALIDATED
  ├─► Ignora TRANSFORM_REVIEW, VALIDATION_BLOCKED, DEPLOY_FAILED
  ├─► Pacchettizza i file HTML/CSS/JS delle pagine validate
  ├─► Pubblica sui canali configurati nel workflow (dev | stage | prod)
  │     └─ Supporta pubblicazione multi-canale simultanea
  ├─► Aggiorna status → DEPLOYED (per pagina pubblicata con successo)
  └─► Aggiorna status → DEPLOY_FAILED (per pagina fallita, con motivo)
```

**Policy**: pubblica ciò che è pronto, ignora il resto. Le pagine bloccate aspettano il prossimo publish esplicito dell'utente, che riparte da zero senza memoria pregressa.

---

## Progress e visibilità

L'utente vede in tempo reale:

```
PublishJob #42 — 10 pagine
├── /home                    ✓ DEPLOYED
├── /blog/article-1          ✓ DEPLOYED
├── /blog/article-2          ⟳ VALIDATING  [Sentinel retry 1/3]
│                                └─ SeoAgent: h1 troppo lungo (72 car > 60 max)
├── /landing/hero            ✗ TRANSFORM_REVIEW
│                                └─ OptimizerAgent: immagine non ottimizzabile (formato raw)
└── ...
```

Ogni task mostra: stato corrente, agente in esecuzione, dettaglio del problema se bloccato.
