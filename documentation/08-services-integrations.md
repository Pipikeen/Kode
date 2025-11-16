# Services et Intégrations - Claude, OpenAI, MCP et Logging

> Documentation technique des services et intégrations de Kode : Service Claude (2242 lignes), OpenAI, MCP, adapters, logging avancé et services auxiliaires

---

## Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Service Claude](#2-service-claude)
3. [Service OpenAI](#3-service-openai)
4. [Model Adapter Factory](#4-model-adapter-factory)
5. [Service MCP](#5-service-mcp)
6. [Debug Logger](#6-debug-logger)
7. [Services Auxiliaires](#7-services-auxiliaires)
8. [Intégrations Externes](#8-intégrations-externes)

---

## 1. Vue d'Ensemble

### 1.1 Architecture des Services

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  COUCHE SERVICES                                │
│  ━━━━━━━━━━━━━━━                              │
│                                                 │
│  ┌─────────────────────────────────┐           │
│  │  Model Services (Primary)       │           │
│  │  ────────────────────────────   │           │
│  │  • claude.ts (2242 lignes)     │           │
│  │  • openai.ts (1000+ lignes)    │           │
│  │  • modelAdapterFactory.ts       │           │
│  └─────────────────────────────────┘           │
│            ↓                                    │
│  ┌─────────────────────────────────┐           │
│  │  Integration Services           │           │
│  │  ────────────────────────────   │           │
│  │  • mcpClient.ts                 │           │
│  │  • oauth.ts                     │           │
│  │  • vcr.ts (recording)           │           │
│  └─────────────────────────────────┘           │
│            ↓                                    │
│  ┌─────────────────────────────────┐           │
│  │  Support Services               │           │
│  │  ────────────────────────────   │           │
│  │  • debugLogger.ts               │           │
│  │  • fileFreshness.ts             │           │
│  │  • systemReminder.ts            │           │
│  │  • mentionProcessor.ts          │           │
│  │  • customCommands.ts            │           │
│  └─────────────────────────────────┘           │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 1.2 Fichiers Principaux

| Fichier | Lignes | Responsabilité |
|---------|--------|----------------|
| **claude.ts** | 2242 | Service principal Anthropic Claude |
| **openai.ts** | 1000+ | Service OpenAI et compatibles |
| **modelAdapterFactory.ts** | 70 | Factory d'adaptateurs de modèles |
| **mcpClient.ts** | 400+ | Client Model Context Protocol |
| **debugLogger.ts** | 1200+ | Système de logging avancé |
| **fileFreshness.ts** | 270 | Suivi de fraîcheur des fichiers |
| **systemReminder.ts** | 390 | Génération de rappels système |
| **oauth.ts** | 230 | Authentification OAuth |
| **customCommands.ts** | 620 | Commandes personnalisées |
| **mentionProcessor.ts** | 220 | Traitement des mentions @file |

---

## 2. Service Claude

### 2.1 Vue d'Ensemble

**Fichier** : `src/services/claude.ts` (2242 lignes)

**Responsabilité** : Service principal pour l'API Anthropic Claude

**Capacités** :
- ✅ Support Claude (Messages API)
- ✅ Support Bedrock (AWS)
- ✅ Support Vertex AI (Google Cloud)
- ✅ Streaming avec AsyncGenerator
- ✅ Gestion du contexte et compaction
- ✅ Thinking tokens et extended thinking
- ✅ Prompt caching
- ✅ System prompts avec contexte projet
- ✅ File freshness reminders
- ✅ Rate limiting et retry

### 2.2 Clients Anthropic

```typescript
// Client standard (Anthropic API)
const anthropic = new Anthropic({
  apiKey: getAnthropicApiKey(),
  timeout: API_TIMEOUT_MS,
  httpAgent: USER_AGENT,
  maxRetries: 3
})

// Client Bedrock (AWS)
const bedrock = new AnthropicBedrock({
  awsAccessKey: process.env.AWS_ACCESS_KEY_ID,
  awsSecretKey: process.env.AWS_SECRET_ACCESS_KEY,
  awsRegion: process.env.AWS_REGION
})

// Client Vertex AI (Google Cloud)
const vertex = new AnthropicVertex({
  region: getVertexRegionForModel(modelName),
  projectId: process.env.GOOGLE_CLOUD_PROJECT
})
```

### 2.3 Fonction Principale : query()

**Signature** :
```typescript
export async function* query(
  messages: MessageType[],
  systemPrompt: string,
  context: string,
  canUseTool: CanUseToolFn,
  options: {
    tools?: Tool[]
    verbose?: boolean
    safeMode?: boolean
    maxThinkingTokens?: number
    // ...
  }
): AsyncGenerator<
  MessageType | AssistantMessage | ProgressMessage,
  void,
  unknown
>
```

**Fonctionnement** :
```
┌────────────────────────────────────────────┐
│                                            │
│  1. PRÉPARATION                            │
│     ┌────────────────────────────┐         │
│     │ Normaliser messages         │         │
│     │ Construire system prompt    │         │
│     │ Convertir outils en schemas │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  2. AUTO-COMPACTION (si nécessaire)        │
│     ┌────────────────────────────┐         │
│     │ Vérifier usage contexte     │         │
│     │ Si > 92% → Compacter        │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  3. SÉLECTION CLIENT                       │
│     ┌────────────────────────────┐         │
│     │ USE_BEDROCK → Bedrock       │         │
│     │ USE_VERTEX → Vertex AI      │         │
│     │ Sinon → Anthropic standard  │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  4. REQUÊTE API                            │
│     ┌────────────────────────────┐         │
│     │ stream=true → Streaming     │         │
│     │ Gestion thinking tokens     │         │
│     │ Prompt caching              │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  5. STREAMING RESPONSE                     │
│     ┌────────────────────────────┐         │
│     │ Yield progress updates      │         │
│     │ Yield text blocks           │         │
│     │ Yield thinking blocks       │         │
│     │ Yield tool uses             │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  6. TOOL EXECUTION                         │
│     ┌────────────────────────────┐         │
│     │ Pour chaque tool_use:       │         │
│     │   - Vérifier permissions    │         │
│     │   - Exécuter outil          │         │
│     │   - Yield résultat          │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  7. RÉCURSION (si tool uses)               │
│     ┌────────────────────────────┐         │
│     │ Appeler query() avec results│         │
│     │ → Boucle jusqu'à stop_reason│         │
│     └────────────────────────────┘         │
│                                            │
└────────────────────────────────────────────┘
```

### 2.4 System Prompt Construction

```typescript
async function constructSystemPrompt(
  systemPrompt: string,
  context: string,
  tools: Tool[],
  options: { safeMode?: boolean }
): Promise<string> {
  const parts: string[] = []

  // 1. CLI sysprompt prefix
  parts.push(getCLISyspromptPrefix())

  // 2. Contexte projet (KODE.md, etc.)
  if (context) {
    parts.push('# Project Context\n\n' + context)
  }

  // 3. System prompt principal
  parts.push(systemPrompt)

  // 4. Prompts des outils
  for (const tool of tools) {
    const toolPrompt = await tool.prompt({ safeMode })
    parts.push(toolPrompt)
  }

  // 5. System reminders
  const reminders = await generateSystemReminders()
  parts.push(reminders)

  return parts.join('\n\n')
}
```

### 2.5 Thinking Tokens

**Extended Thinking** : Réflexion approfondie pour Claude

```typescript
export async function getMaxThinkingTokens(
  messages: MessageType[]
): Promise<number> {
  // Vérifier configuration
  if (process.env.MAX_THINKING_TOKENS) {
    return parseInt(process.env.MAX_THINKING_TOKENS)
  }

  const config = getGlobalConfig()
  const modelManager = getModelManager()
  const modelProfile = modelManager.getModel('main')

  // Appliquer logique de reasoning effort
  const reasoningEffort = getReasoningEffort(modelProfile)

  switch (reasoningEffort) {
    case 'low':
      return 2000
    case 'medium':
      return 5000
    case 'high':
      return 10000
    default:
      return 0 // Pas de thinking
  }
}
```

**API Call** :
```typescript
const response = await client.beta.messages.create({
  model: modelName,
  messages: normalizedMessages,
  system: systemPrompt,
  max_tokens: 8192,
  stream: true,
  thinking: {
    type: 'enabled',
    budget_tokens: maxThinkingTokens
  },
  ...
})
```

### 2.6 Prompt Caching

**Principe** : Réutiliser parties du prompt pour réduire coûts

```typescript
// System blocks avec cache_control
const systemBlocks: AnthropicAPISystemBlock[] = [
  {
    type: 'text',
    text: cliSyspromptPrefix,
    cache_control: { type: 'ephemeral' } // Cacheable
  },
  {
    type: 'text',
    text: projectContext,
    cache_control: { type: 'ephemeral' } // Cacheable
  },
  {
    type: 'text',
    text: systemPrompt,
    // Pas de cache_control → Non caché
  }
]
```

**Économies** :
- Cache Read : 10% du prix normal
- Cache Write : 25% de surcoût (première fois)
- Net positif après 2-3 requêtes

---

## 3. Service OpenAI

### 3.1 Vue d'Ensemble

**Fichier** : `src/services/openai.ts` (1000+ lignes)

**Responsabilité** : Service pour OpenAI et tous les providers compatibles

**Providers supportés** :
- OpenAI (GPT-4, GPT-5, etc.)
- Mistral AI
- DeepSeek
- Kimi (Moonshot)
- Qwen (Alibaba)
- ChatGLM (Zhipu)
- Minimax
- Baidu Qianfan
- SiliconFlow
- BigDream
- OpenDev
- xAI (Grok)
- Groq
- Google Gemini
- Ollama (local)
- Azure OpenAI
- Custom providers

### 3.2 Fonctions Principales

#### 3.2.1 getCompletionWithProfile()

**Standard Chat Completions API** :

```typescript
export async function* getCompletionWithProfile(
  modelProfile: ModelProfile,
  messages: MessageType[],
  tools: Tool[],
  systemPrompt: string,
  options: {
    temperature?: number
    maxTokens?: number
    stream?: boolean
  }
): AsyncGenerator<AssistantMessage | ProgressMessage, void, unknown> {
  // 1. Créer client OpenAI
  const client = new OpenAI({
    apiKey: modelProfile.apiKey,
    baseURL: modelProfile.baseURL,
    timeout: API_TIMEOUT_MS
  })

  // 2. Convertir messages au format OpenAI
  const openaiMessages = convertMessagesToOpenAI(messages, systemPrompt)

  // 3. Convertir tools au format OpenAI
  const openaiTools = tools.map(tool => ({
    type: 'function' as const,
    function: {
      name: tool.name,
      description: await tool.description(),
      parameters: zodToJsonSchema(tool.inputSchema)
    }
  }))

  // 4. Créer requête
  const stream = await client.chat.completions.create({
    model: modelProfile.modelName,
    messages: openaiMessages,
    tools: openaiTools,
    temperature: options.temperature,
    max_tokens: options.maxTokens,
    stream: true
  })

  // 5. Streamer réponse
  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta

    if (delta?.content) {
      yield {
        type: 'progress',
        content: delta.content
      }
    }

    if (delta?.tool_calls) {
      // Accumuler tool calls
    }
  }

  // 6. Yield final message
  yield {
    type: 'assistant',
    message: accumulatedMessage
  }
}
```

#### 3.2.2 getGPT5CompletionWithProfile()

**GPT-5 Responses API** (architecture différente) :

```typescript
export async function* getGPT5CompletionWithProfile(
  modelProfile: ModelProfile,
  messages: MessageType[],
  tools: Tool[],
  systemPrompt: string,
  options: {
    reasoningEffort?: 'low' | 'medium' | 'high'
    maxCompletionTokens?: number
  }
): AsyncGenerator<AssistantMessage | ProgressMessage, void, unknown> {
  const client = new OpenAI({
    apiKey: modelProfile.apiKey,
    baseURL: 'https://api.openai.com/v1'
  })

  // GPT-5 utilise Responses API (pas Chat Completions)
  const response = await client.beta.responses.create({
    model: modelProfile.modelName,
    messages: convertMessagesToResponsesAPI(messages, systemPrompt),
    tools: convertToolsToResponsesAPI(tools),
    reasoning_effort: options.reasoningEffort || 'medium',
    max_completion_tokens: options.maxCompletionTokens,
    stream: true
  })

  // Streamer réponse (format Responses API)
  for await (const event of response) {
    if (event.type === 'content.delta') {
      yield {
        type: 'progress',
        content: event.delta
      }
    }

    if (event.type === 'reasoning.delta') {
      // GPT-5 reasoning (similaire à Claude thinking)
      yield {
        type: 'thinking',
        content: event.delta
      }
    }

    if (event.type === 'tool_call.delta') {
      // Tool calls
    }
  }
}
```

### 3.3 Conversion de Messages

**Anthropic → OpenAI** :

```typescript
function convertMessagesToOpenAI(
  messages: MessageType[],
  systemPrompt: string
): OpenAI.ChatCompletionMessageParam[] {
  const result: OpenAI.ChatCompletionMessageParam[] = []

  // System prompt
  result.push({
    role: 'system',
    content: systemPrompt
  })

  // User et Assistant messages
  for (const msg of messages) {
    if (msg.type === 'user') {
      result.push({
        role: 'user',
        content: convertUserContent(msg.message.content)
      })
    } else if (msg.type === 'assistant') {
      result.push({
        role: 'assistant',
        content: convertAssistantContent(msg.message.content),
        tool_calls: extractToolCalls(msg.message.content)
      })
    }
  }

  return result
}
```

---

## 4. Model Adapter Factory

### 4.1 Principe

**Problème** : Différentes architectures d'API (Chat Completions vs Responses API)

**Solution** : Adaptateurs avec interface unifiée

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  CLIENT CODE                                    │
│  ───────────────                               │
│                                                 │
│  const adapter = ModelAdapterFactory            │
│    .createAdapter(modelProfile)                 │
│                                                 │
│  const result = await adapter.execute(...)      │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  FACTORY                                        │
│  ───────────────                               │
│                                                 │
│  determineAPIType() → 'responses_api' | 'chat_completions' │
│                                                 │
│  ┌─────────────────┬────────────────────┐      │
│  │                 │                    │      │
│  │  Responses API  │  Chat Completions  │      │
│  │  (GPT-5)        │  (GPT-4, Claude)   │      │
│  │                 │                    │      │
│  └─────────────────┴────────────────────┘      │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 4.2 Adaptateurs

#### 4.2.1 ResponsesAPIAdapter

**Usage** : GPT-5 et modèles supportant Responses API

**Fichier** : `src/services/adapters/responsesAPI.ts`

**Capacités** :
- Reasoning (thinking)
- Tool calls avec gestion d'état
- Streaming avancé
- Gestion de conversation_id

#### 4.2.2 ChatCompletionsAdapter

**Usage** : GPT-4, Claude (via conversion), autres modèles

**Fichier** : `src/services/adapters/chatCompletions.ts`

**Capacités** :
- Standard tool calls
- Streaming classique
- Compatible tous providers OpenAI-like

### 4.3 Détermination Automatique

```typescript
static determineAPIType(
  modelProfile: ModelProfile,
  capabilities: ModelCapabilities
): 'responses_api' | 'chat_completions' {
  // 1. Vérifier support Responses API
  if (capabilities.apiArchitecture.primary !== 'responses_api') {
    return 'chat_completions'
  }

  // 2. Endpoint officiel ?
  const isOfficialOpenAI = !modelProfile.baseURL ||
    modelProfile.baseURL.includes('api.openai.com')

  // 3. Non-officiel → Utiliser fallback si disponible
  if (!isOfficialOpenAI) {
    if (capabilities.apiArchitecture.fallback === 'chat_completions') {
      return capabilities.apiArchitecture.fallback
    }
  }

  // 4. Officiel et supporté → Responses API
  return 'responses_api'
}
```

---

## 5. Service MCP

Voir [05-systeme-outils.md § 6 Intégration MCP](./05-systeme-outils.md#6-intégration-mcp) pour détails complets.

### 5.1 Résumé

**Fichier** : `src/services/mcpClient.ts` (400+ lignes)

**Responsabilité** : Client pour Model Context Protocol

**Capacités** :
- Gestion serveurs MCP (stdio, SSE)
- Conversion outils MCP → Tool Kode
- Cycle de vie des processus
- Configuration multi-scope (project, global, mcprc)

---

## 6. Debug Logger

### 6.1 Vue d'Ensemble

**Fichier** : `src/utils/debugLogger.ts` (1200+ lignes)

**Responsabilité** : Système de logging avancé multi-niveaux

**Innovation** : Logs structurés, fichiers séparés par type, timestamps précis

### 6.2 Niveaux de Log

```typescript
export enum LogLevel {
  TRACE = 'TRACE',      // Trace détaillée
  DEBUG = 'DEBUG',      // Debug info
  INFO = 'INFO',        // Info générale
  WARN = 'WARN',        // Avertissements
  ERROR = 'ERROR',      // Erreurs
  FLOW = 'FLOW',        // Flux d'exécution
  API = 'API',          // Requêtes API
  STATE = 'STATE',      // Changements d'état
  REMINDER = 'REMINDER' // System reminders
}
```

### 6.3 Fichiers de Log

**Emplacement** : `~/.kode/<project>/debug/`

| Fichier | Contenu |
|---------|---------|
| **detailed.log** | Tous les logs (niveau TRACE+) |
| **flow.log** | Flux d'exécution uniquement |
| **api.log** | Requêtes/réponses API uniquement |
| **state.log** | Changements d'état uniquement |

**Format** :
```
2025-01-16T10:30:45.123Z [INFO] [SESSION_START] {requestId: "abc123", elapsed: 0ms}
  ├─ sessionId: "xyz789"
  ├─ cwd: "/home/user/project"
  └─ modelName: "claude-sonnet-4"
```

### 6.4 API de Logging

```typescript
import { debug } from '@utils/debugLogger'

// Log simple
debug.info('USER_INPUT', { input: userMessage })

// Log avec phase
debug.api('API_REQUEST', {
  model: 'claude-sonnet-4',
  tokens: 1234,
  stream: true
})

// Log d'erreur avec diagnostic
debug.error('API_ERROR', {
  error: error.message,
  statusCode: 429,
  retryAfter: 60
})

// Log de flow
debug.flow('TOOL_EXECUTION_START', {
  toolName: 'BashTool',
  input: { command: 'npm test' }
})
```

### 6.5 Request Context

**Tracking de requêtes** :

```typescript
class RequestContext {
  public readonly id: string
  public readonly startTime: number
  private phases: Map<string, number> = new Map()

  constructor() {
    this.id = randomUUID()
    this.startTime = Date.now()
  }

  markPhase(phaseName: string): void {
    this.phases.set(phaseName, Date.now())
  }

  getElapsed(): number {
    return Date.now() - this.startTime
  }

  getPhaseElapsed(phaseName: string): number {
    const phaseStart = this.phases.get(phaseName)
    if (!phaseStart) return 0
    return Date.now() - phaseStart
  }
}

// Usage
const request = getCurrentRequest()
debug.flow('PHASE_START', { requestId: request.id })
request.markPhase('api_call')
// ...
debug.flow('PHASE_END', {
  requestId: request.id,
  elapsed: request.getPhaseElapsed('api_call')
})
```

### 6.6 Logging LLM Interactions

**Fonction spécialisée** :

```typescript
export function logLLMInteraction(
  direction: 'request' | 'response',
  data: {
    model: string
    messages?: any[]
    tools?: any[]
    response?: any
    usage?: any
    cost?: number
  }
): void {
  if (direction === 'request') {
    debug.api('LLM_REQUEST', {
      model: data.model,
      messageCount: data.messages?.length,
      toolCount: data.tools?.length,
      systemPromptLength: data.messages?.[0]?.content?.length
    })
  } else {
    debug.api('LLM_RESPONSE', {
      model: data.model,
      usage: data.usage,
      cost: data.cost,
      responseLength: JSON.stringify(data.response).length
    })
  }
}
```

---

## 7. Services Auxiliaires

### 7.1 File Freshness Service

**Fichier** : `src/services/fileFreshness.ts` (270 lignes)

**Responsabilité** : Suivi de la fraîcheur des fichiers lus/modifiés

**Fonctionnalités** :

#### 7.1.1 Tracking

```typescript
export function recordFileRead(filePath: string): void {
  const state = getFileFreshnessState()
  state.fileReads.set(filePath, Date.now())
}

export function recordFileEdit(filePath: string): void {
  const state = getFileFreshnessState()
  state.fileEdits.set(filePath, Date.now())
}
```

#### 7.1.2 Stale File Detection

```typescript
export function generateFileModificationReminder(
  filePath: string,
  lastReadTime: number
): string | null {
  const state = getFileFreshnessState()
  const lastEdit = state.fileEdits.get(filePath)

  // Si modifié depuis dernière lecture
  if (lastEdit && lastEdit > lastReadTime) {
    return `⚠️  File ${filePath} has been modified since you last read it. Consider re-reading.`
  }

  return null
}
```

#### 7.1.3 System Reminder Integration

```typescript
// Lors de la construction du system prompt
const reminders: string[] = []

for (const [filePath, lastRead] of readFileTimestamps) {
  const reminder = generateFileModificationReminder(filePath, lastRead)
  if (reminder) {
    reminders.push(reminder)
  }
}

if (reminders.length > 0) {
  systemPrompt += '\n\n<system-reminder>\n' +
    reminders.join('\n') +
    '\n</system-reminder>'
}
```

### 7.2 System Reminder Service

**Fichier** : `src/services/systemReminder.ts` (390 lignes)

**Responsabilité** : Génération de rappels système contextuels

**Types de Reminders** :

#### 7.2.1 File Modification Reminders

```typescript
// Fichier modifié depuis dernière lecture
<system-reminder>
Note: /path/to/file.ts was edited before the last conversation was
summarized, but the contents are too large to include. Use Read tool
if you need to access it.
</system-reminder>
```

#### 7.2.2 Empty File Reminders

```typescript
// Fichier vide après lecture
<system-reminder>
WARNING: /path/to/file.ts exists but has empty contents. This may
indicate a problem with the file or that it was recently cleared.
</system-reminder>
```

#### 7.2.3 Malware Warning Reminders

```typescript
// Lors de la lecture d'un fichier suspect
<system-reminder>
Whenever you read a file, you should consider whether it would be
considered malware. You CAN and SHOULD provide analysis of malware,
what it is doing. But you MUST refuse to improve or augment the code.
</system-reminder>
```

#### 7.2.4 Tool Usage Reminders

```typescript
// Rappel d'utilisation d'outil
<system-reminder>
The TodoWrite tool hasn't been used recently. If you're working on
tasks that would benefit from tracking progress, consider using the
TodoWrite tool to track progress.
</system-reminder>
```

### 7.3 Mention Processor Service

**Fichier** : `src/services/mentionProcessor.ts` (220 lignes)

**Responsabilité** : Traitement des mentions @file dans les prompts

**Exemple** :
```
User input: @src/App.tsx Can you explain this component?

Processed:
```
# src/App.tsx
```tsx
import React from 'react'
// ... file contents
```

Can you explain this component?
```
```

**Fonctionnement** :
```typescript
export async function processMentions(
  input: string
): Promise<{ text: string; files: string[] }> {
  const mentionRegex = /@([^\s]+)/g
  const mentions: string[] = []
  let text = input

  // Extraire mentions
  let match
  while ((match = mentionRegex.exec(input)) !== null) {
    mentions.push(match[1])
  }

  // Lire fichiers et remplacer
  for (const mention of mentions) {
    const filePath = resolve(getCwd(), mention)
    if (existsSync(filePath)) {
      const content = await readFile(filePath)
      text = text.replace(
        `@${mention}`,
        `# ${mention}\n\`\`\`\n${content}\n\`\`\``
      )
    }
  }

  return { text, files: mentions }
}
```

### 7.4 Custom Commands Service

**Fichier** : `src/services/customCommands.ts` (620 lignes)

**Responsabilité** : Chargement et exécution de commandes personnalisées

**Emplacement** : `.claude/commands/` ou `.kode/commands/`

**Format** : Fichiers Markdown avec YAML frontmatter

**Exemple** : `.kode/commands/test.md`
```markdown
---
name: test
description: Run project tests
---

Run the test suite for this project and report results.
```

**Exécution** :
```bash
kode /test
# → Charge le fichier et exécute le prompt
```

### 7.5 OAuth Service

**Fichier** : `src/services/oauth.ts` (230 lignes)

**Responsabilité** : Authentification OAuth pour Anthropic

**Flow** :
```
1. User: kode oauth login
2. → Ouvre browser avec authorization URL
3. User autorise dans le browser
4. → Callback avec code
5. → Exchange code pour access token
6. → Stocke token dans config
7. ✓ Authentifié
```

---

## 8. Intégrations Externes

### 8.1 VCR (Recording)

**Fichier** : `src/services/vcr.ts` (110 lignes)

**Responsabilité** : Enregistrement et replay de requêtes API (testing)

**Usage** :
```typescript
import { withVCR } from '@services/vcr'

const result = await withVCR('test-scenario', async () => {
  return await anthropic.messages.create({ ... })
})

// Première exécution → Enregistre
// Exécutions suivantes → Replay depuis enregistrement
```

**Avantages** :
- Tests déterministes
- Pas de dépendance API pendant tests
- Coûts réduits

### 8.2 Notifier Service

**Fichier** : `src/services/notifier.ts` (23 lignes)

**Responsabilité** : Notifications terminal (iTerm2, bell)

**Modes** :
```typescript
export type NotificationChannel =
  | 'iterm2'              // iTerm2 notifications
  | 'terminal_bell'       // Bell sonore
  | 'iterm2_with_bell'    // Les deux
  | 'notifications_disabled' // Désactivé
```

**Usage** :
```typescript
import { notify } from '@services/notifier'

notify('Task complete', 'Tests passed successfully')
```

### 8.3 Sentry Service

**Fichier** : `src/services/sentry.ts` (3 lignes)

**Responsabilité** : Error tracking (désactivé par défaut)

---

## Annexes

### A. Statistiques des Services

| Service | Lignes | Fonctions | Complexité |
|---------|--------|-----------|------------|
| **claude.ts** | 2242 | 50+ | Très haute |
| **openai.ts** | 1000+ | 30+ | Haute |
| **debugLogger.ts** | 1200+ | 40+ | Haute |
| **mcpClient.ts** | 400+ | 25+ | Moyenne |
| **fileFreshness.ts** | 270 | 15+ | Moyenne |
| **systemReminder.ts** | 390 | 20+ | Moyenne |
| **customCommands.ts** | 620 | 15+ | Moyenne |
| **mentionProcessor.ts** | 220 | 10+ | Basse |
| **oauth.ts** | 230 | 12+ | Moyenne |
| **vcr.ts** | 110 | 5+ | Basse |

### B. Variables d'Environnement

| Variable | Service | Description |
|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | claude.ts | Clé API Anthropic |
| `OPENAI_API_KEY` | openai.ts | Clé API OpenAI |
| `USE_BEDROCK` | claude.ts | Utiliser AWS Bedrock |
| `USE_VERTEX` | claude.ts | Utiliser Google Vertex |
| `DISABLE_PROMPT_CACHING` | claude.ts | Désactiver cache |
| `MAX_THINKING_TOKENS` | claude.ts | Tokens thinking max |
| `API_TIMEOUT_MS` | All | Timeout API (ms) |
| `NODE_ENV` | All | Environment (dev/prod/test) |

### C. Dépendances Principales

| Dépendance | Version | Usage |
|------------|---------|-------|
| `@anthropic-ai/sdk` | 0.x | Client Anthropic |
| `@anthropic-ai/bedrock-sdk` | 0.x | Client Bedrock |
| `@anthropic-ai/vertex-sdk` | 0.x | Client Vertex |
| `openai` | 4.x | Client OpenAI |
| `@modelcontextprotocol/sdk` | 0.x | Client MCP |
| `zod` | 3.x | Validation schemas |
| `lodash-es` | 4.x | Utilitaires |

### D. Patterns Utilisés

| Pattern | Où | Description |
|---------|-----|-------------|
| **AsyncGenerator** | claude.ts, openai.ts | Streaming de réponses |
| **Factory** | modelAdapterFactory.ts | Création d'adaptateurs |
| **Singleton** | debugLogger.ts | Instance unique de logger |
| **Adapter** | adapters/ | Unification d'APIs différentes |
| **Strategy** | messageContextManager.ts | Stratégies de rétention |
| **Observer** | fileFreshness.ts | Tracking de modifications |
| **Memoization** | mcpClient.ts | Cache de résultats |

### E. Exemples d'Utilisation

#### Requête Claude Simple

```typescript
import { query } from '@services/claude'
import { getSystemPrompt } from '@constants/prompts'
import { getContext } from '@context'

const generator = query(
  messages,
  await getSystemPrompt(),
  await getContext(),
  canUseTool,
  {
    tools: getAllTools(),
    verbose: true,
    safeMode: false,
    maxThinkingTokens: 5000
  }
)

for await (const message of generator) {
  if (message.type === 'progress') {
    console.log('Progress:', message.content)
  } else if (message.type === 'assistant') {
    console.log('Assistant:', message.message.content)
  }
}
```

#### Logging Structuré

```typescript
import { debug, getCurrentRequest } from '@utils/debugLogger'

// Début de requête
const request = getCurrentRequest()
debug.flow('REQUEST_START', {
  requestId: request.id,
  userInput: input
})

// Phase API
request.markPhase('api_call')
debug.api('API_REQUEST', {
  model: 'claude-sonnet-4',
  tokens: countTokens(messages)
})

// Réponse
debug.api('API_RESPONSE', {
  elapsed: request.getPhaseElapsed('api_call'),
  tokensUsed: response.usage.total_tokens,
  cost: calculateCost(response.usage)
})

// Fin de requête
debug.flow('REQUEST_END', {
  totalElapsed: request.getElapsed()
})
```

#### Custom Model Adapter

```typescript
import { ModelAdapterFactory } from '@services/modelAdapterFactory'

const modelProfile = {
  name: "My Custom Model",
  modelName: "custom-model-v1",
  provider: "custom-openai",
  baseURL: "https://api.custom.com/v1",
  apiKey: "sk-...",
  maxTokens: 4096,
  contextLength: 32000,
  isActive: true,
  createdAt: Date.now()
}

const adapter = ModelAdapterFactory.createAdapter(modelProfile)

const result = await adapter.execute({
  messages,
  tools,
  systemPrompt,
  stream: true
})
```

---

**Navigation** :
- ← [Configuration](./07-configuration.md)
- → (Fin des documents techniques)
- ↑ [Index](./INDEX.md)
- ⌂ [README](./README.md)

---

**Dernière mise à jour** : 16 janvier 2025
**Version de Kode** : 0.1.0
