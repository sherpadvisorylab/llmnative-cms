# Agent Architecture — LLMNative CMS

## Principio

Ogni agente di trasformazione è **autosufficiente**: contiene sia le regole di trasformazione che il validatore di quelle regole. Il sistema è modulare — aggiungere un nuovo agente non richiede modifiche al core del pipeline.

---

## Due categorie di agenti

### TransformAgent
Agisce sulla pagina: la modifica, la corregge, la ottimizza.
- Corre **in parallelo** con gli altri transform agents
- Si auto-valida prima di segnare il lavoro come completato
- Espone il proprio `validate()` al SentinelAgent

### SentinelAgent (uno solo per workflow)
Non modifica la pagina. Aggrega e verifica.
- Corre **da solo**, dopo tutti i transform agents
- Chiama il `validate()` di ogni transform agent nel workflow
- Aggiunge i propri check esclusivi (esterni, API)
- È l'ultima barriera prima della pubblicazione

---

## Interfaccia `TransformAgent`

```typescript
interface ValidationReport {
  passed: boolean
  checks: Array<{
    rule: string           // es. "h1MaxLength"
    passed: boolean
    expected: unknown      // es. 60
    actual: unknown        // es. 72
    message: string        // es. "H1 supera 60 caratteri (72)"
    severity: 'error' | 'warning'
    autoFixed: boolean
  }>
}

interface TransformAgent {
  readonly name: string    // es. "SeoAgent"

  // Applica le regole di trasformazione alla pagina.
  // Ritorna la pagina modificata.
  transform(
    page: PageArtifact,
    config: AgentConfig
  ): Promise<PageArtifact>

  // Verifica che le regole siano rispettate nella pagina.
  // Stesso metodo chiamato internamente e da SentinelAgent.
  validate(
    page: PageArtifact,
    config: AgentConfig
  ): Promise<ValidationReport>
}
```

---

## Ciclo di esecuzione di un TransformAgent

```typescript
async function runTransformAgent(agent, page, config, maxRetries) {
  let current = page
  let attempt = 0

  while (attempt <= maxRetries) {
    current = await agent.transform(current, config)
    const report = await agent.validate(current, config)

    if (report.passed) {
      return { status: 'done', page: current, report }
    }

    attempt++
  }

  return { status: 'blocked', page: current, report }
}
```

L'agente non conosce il concetto di "retry del pipeline" — esegue i propri tentativi internamente. Se esaurisce i retry e non passa la propria validazione, segnala `blocked` al pipeline.

---

## SentinelAgent

```typescript
interface SentinelAgent {
  readonly name: 'SentinelAgent'

  // Esegue tutti i validatori degli agenti del workflow
  // più i check esclusivi del Sentinel.
  run(
    page: PageArtifact,
    workflow: PublishWorkflow,
    retryCount: number
  ): Promise<SentinelReport>
}

interface SentinelReport {
  passed: boolean
  agentReports: Record<string, ValidationReport>  // un report per agente
  exclusiveChecks: ValidationReport               // check esclusivi Sentinel
  retryTargets: string[]                          // nomi agenti da re-triggerare
  // es. ['SeoAgent'] se solo SeoAgent.validate() ha fallito
}
```

### Check esclusivi del SentinelAgent

Check che non appartengono a nessun transform agent specifico:
- **PageSpeed API** — verifica performance reale della pagina
- **Lighthouse** — audit completo (performance, accessibilità, best practices)
- **Link checker** — verifica link non rotti
- **Schema.org validator** — verifica JSON-LD corretto
- Qualsiasi check configurato a livello di workflow

---

## Agenti built-in

### `SeoAgent`

**Responsabilità transform:**
- Ottimizza `<title>` e `<meta description>` rispettando i limiti di caratteri
- Verifica e corregge struttura heading (H1 unico, gerarchia H2/H3)
- Aggiunge `alt` mancanti sulle immagini
- Genera/aggiorna JSON-LD (Schema.org) per il content type
- Verifica canonical URL
- Aggiunge/corregge Open Graph e Twitter Card meta

**Config (esempio):**
```json
{
  "titleMaxLength": 60,
  "descriptionMaxLength": 160,
  "h1MaxLength": 60,
  "requireAltText": true,
  "autoFixLevel": "aggressive | conservative"
}
```

**Regole validate():**
- `title` presente e ≤ `titleMaxLength`
- `description` presente e ≤ `descriptionMaxLength`
- H1 presente, unico, ≤ `h1MaxLength`
- Nessuna immagine senza `alt` (se `requireAltText: true`)
- JSON-LD presente e valido
- Canonical URL presente

---

### `OptimizerAgent`

**Responsabilità transform:**
- Minifica HTML, CSS, JS inline
- Ottimizza immagini (compressione, formato WebP/AVIF)
- Aggiunge `loading="lazy"` sulle immagini below-fold
- Inlinea CSS critici (above-fold)
- Rimuove CSS/JS non utilizzati

**Config (esempio):**
```json
{
  "imageQuality": 85,
  "imageFormats": ["webp", "avif"],
  "inlineCriticalCss": true,
  "minifyHtml": true,
  "minifyCss": true,
  "minifyJs": true
}
```

**Regole validate():**
- Nessuna immagine sopra soglia dimensione (es. > 200KB)
- CSS/JS minificati
- `loading="lazy"` presente sulle immagini configurate

---

## Aggiungere un nuovo TransformAgent

1. Implementare l'interfaccia `TransformAgent` con `transform()` e `validate()`
2. Registrarlo nel registry degli agenti disponibili
3. Aggiungerlo al workflow config nella sezione `transform`

Il pipeline e il SentinelAgent lo raccolgono automaticamente. Zero modifiche al core.

```typescript
// Esempio: nuovo agente per accessibilità
class AccessibilityAgent implements TransformAgent {
  readonly name = 'AccessibilityAgent'

  async transform(page, config) {
    // Aggiunge aria-label mancanti, migliora contrasto, ecc.
  }

  async validate(page, config) {
    // Verifica WCAG AA/AAA secondo config
  }
}
```
