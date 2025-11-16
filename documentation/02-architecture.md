# Architecture Système de Kode

> Documentation exhaustive de l'architecture technique de Kode

[← Retour à l'index](./README.md)

---

## Table des Matières

1. [Vue d'Ensemble](#vue-densemble)
2. [Organisation des Répertoires](#organisation-des-répertoires)
3. [Les Trois Couches Principales](#les-trois-couches-principales)
4. [Flux de Données et Interactions](#flux-de-données-et-interactions)
5. [Points d'Entrée et Cycle de Vie](#points-dentrée-et-cycle-de-vie)
6. [Interactions Entre Modules](#interactions-entre-modules)
7. [Patterns Architecturaux](#patterns-architecturaux)
8. [Points Clés d'Architecture](#points-clés-darchitecture)

---

## 1. Vue d'Ensemble

### Principe Fondamental

Kode implémente une **architecture à trois couches** basée sur une boucle récursive sophistiquée :

```
LLM → Tool Execution → LLM → Tool Execution → ...
```

Cette boucle permet à l'IA d'exécuter des séquences complexes de tâches en utilisant des outils spécialisés, avec un système de permissions granulaire et une gestion intelligente du contexte conversationnel.

### Architecture Globale

```
┌─────────────────────────────────────────────────────────────┐
│                    POINT D'ENTRÉE CLI                        │
│  src/entrypoints/cli.tsx                                     │
│  - Parse arguments (Commander.js)                            │
│  - Setup config & agents                                     │
│  - Initialize MCP clients                                    │
└────────────┬────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│              COUCHE 1: USER INTERACTION (REPL)               │
│  src/screens/REPL.tsx                                        │
│                                                              │
│  ┌──────────────┐     ┌──────────────┐    ┌──────────────┐ │
│  │ PromptInput  │────▶│  onQuery()   │───▶│  Messages    │ │
│  │ (Ink)        │     │  Handler     │    │  State       │ │
│  └──────────────┘     └──────┬───────┘    └──────────────┘ │
│                              │                              │
└──────────────────────────────┼──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│           COUCHE 2: ORCHESTRATION (Query Loop)               │
│  src/query.ts                                                │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ query() AsyncGenerator:                                │ │
│  │                                                        │ │
│  │ 1. Auto-compact messages                              │ │
│  │    ↓                                                  │ │
│  │ 2. Format system prompt + context                    │ │
│  │    ↓                                                  │ │
│  │ 3. Call LLM (queryLLM)                               │ │
│  │    ↓                                                  │ │
│  │ 4. Extract tool_use blocks                           │ │
│  │    ↓                                                  │ │
│  │ 5. Execute tools (concurrent/serial) ────────┐       │ │
│  │    ↓                                         │       │ │
│  │ 6. Recursive query with results             │       │ │
│  └──────────────────────────────────────────────┼───────┘ │
│                                                 │         │
└─────────────────────────────────────────────────┼─────────┘
                                                  │
                                                  ▼
┌─────────────────────────────────────────────────────────────┐
│          COUCHE 3: TOOL EXECUTION (21+ Tools)                │
│  src/tools/*/                                                │
│                                                              │
│  ┌─────────────────┐   ┌─────────────────┐   ┌───────────┐ │
│  │   BashTool      │   │  FileEditTool   │   │ TaskTool  │ │
│  │   (shell cmd)   │   │  (edit files)   │   │ (agents)  │ │
│  └────────┬────────┘   └────────┬────────┘   └──────┬────┘ │
│           │                     │                    │      │
│           ▼                     ▼                    ▼      │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Permissions Check (hasPermissionsToUseTool)         │ │
│  └────────┬──────────────────────────────────────────────┘ │
│           │                                                 │
│           ▼                                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Tool.call() → AsyncGenerator<Result | Progress>     │ │
│  └────────┬──────────────────────────────────────────────┘ │
│           │                                                 │
└───────────┼─────────────────────────────────────────────────┘
            │
            ▼ (yield results back to query)
┌─────────────────────────────────────────────────────────────┐
│                    SERVICES LAYER                            │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Claude API   │  │  OpenAI API  │  │  MCP Servers     │  │
│  │ (claude.ts)  │  │ (openai.ts)  │  │  (mcpClient.ts)  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ModelAdapterFactory (unification layer)              │ │
│  │  - Anthropic Adapter                                  │ │
│  │  - OpenAI Adapter                                     │ │
│  │  - GPT-5 Responses API Adapter                        │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Organisation des Répertoires

### Structure Complète

```
src/
├── entrypoints/          # Points d'entrée de l'application
│   ├── cli.tsx          # Point d'entrée principal du CLI
│   └── mcp.ts           # Serveur Model Context Protocol
│
├── screens/             # Composants UI principaux (COUCHE 1)
│   ├── REPL.tsx         # Interface REPL interactive (810 lignes)
│   ├── Doctor.tsx       # Diagnostic système
│   ├── LogList.tsx      # Gestionnaire de logs
│   └── ResumeConversation.tsx  # Reprise de conversation
│
├── tools/               # Outils exécutables (COUCHE 3)
│   ├── TaskTool/        # Orchestrateur d'agents (COUCHE 2)
│   │   ├── TaskTool.tsx
│   │   └── prompt.ts
│   ├── BashTool/        # Exécution shell
│   ├── FileReadTool/    # Lecture fichiers
│   ├── FileEditTool/    # Édition fichiers
│   ├── FileWriteTool/   # Écriture fichiers
│   ├── MultiEditTool/   # Éditions multiples
│   ├── GrepTool/        # Recherche contenu
│   ├── GlobTool/        # Recherche patterns
│   ├── NotebookReadTool/  # Notebooks Jupyter
│   ├── NotebookEditTool/
│   ├── MemoryReadTool/  # Mémoire persistante
│   ├── MemoryWriteTool/
│   ├── WebSearchTool/   # Recherche web
│   ├── URLFetcherTool/  # Fetch URLs
│   ├── ThinkTool/       # Métacognition
│   ├── TodoWriteTool/   # Gestion tâches
│   ├── AskExpertModelTool/  # Consultation multi-modèle
│   ├── ArchitectTool/   # Génération architecturale
│   ├── LSTool/          # Liste répertoires
│   └── MCPTool/         # Template MCP
│
├── services/            # Services externes et intégrations
│   ├── claude.ts        # Service Anthropic (2242 lignes)
│   ├── openai.ts        # Service OpenAI (1287 lignes)
│   ├── mcpClient.ts     # Client MCP (562 lignes)
│   ├── modelAdapterFactory.ts  # Factory d'adaptateurs
│   ├── responseStateManager.ts # État GPT-5
│   ├── systemReminder.ts       # Reminders contextuels
│   ├── vcr.ts           # Test recording/replay
│   └── adapters/        # Adaptateurs de modèles
│       ├── base.ts
│       ├── responsesAPI.ts
│       └── chatCompletions.ts
│
├── utils/               # Utilitaires système
│   ├── model.ts         # ModelManager
│   ├── config.ts        # Configuration hiérarchique
│   ├── agentLoader.ts   # Chargement dynamique agents
│   ├── messageContextManager.ts  # Gestion contexte
│   ├── messages.ts      # Normalisation messages
│   ├── debugLogger.ts   # Logging avancé (1233 lignes)
│   ├── tokens.ts        # Comptage tokens
│   ├── autoCompactCore.ts  # Auto-compaction
│   ├── sessionState.ts  # État session
│   └── permissions/     # Permissions filesystem
│       └── filesystem.ts
│
├── components/          # Composants UI réutilisables (Ink)
│   ├── Message.tsx      # Affichage messages
│   ├── PromptInput.tsx  # Input utilisateur (347 lignes)
│   ├── permissions/     # Composants permissions
│   │   ├── PermissionRequest.tsx
│   │   └── ModeIndicator.tsx
│   ├── binary-feedback/ # A/B testing
│   │   └── BinaryFeedback.tsx
│   └── [50+ autres composants]
│
├── types/               # Définitions TypeScript
│   ├── conversation.ts  # Types messages
│   ├── modelCapabilities.ts  # Capacités modèles
│   ├── RequestContext.ts
│   └── [autres types]
│
├── constants/           # Constantes et configurations
│   ├── prompts.ts       # System prompts
│   ├── models.ts        # Configurations modèles
│   ├── modelCapabilities.ts  # Capacités par modèle
│   └── product.ts       # Métadonnées produit
│
├── commands/            # Commandes slash
│   ├── help.tsx
│   ├── model.tsx
│   ├── config.tsx
│   ├── agents.tsx
│   └── [autres commandes]
│
├── hooks/               # React hooks personnalisés
├── context/             # Contexte React
├── Tool.ts              # Interface de base pour outils
├── query.ts             # Orchestration requêtes LLM (COUCHE 2)
├── permissions.ts       # Système de permissions
└── context.ts           # Gestion contexte projet
```

### Métriques du Projet

- **Total** : 272 fichiers TypeScript
- **Outils natifs** : 21
- **Composants UI** : 50+
- **Commandes** : 15+
- **Services principaux** : 7

---

## 3. Les Trois Couches Principales

### 🎨 Couche 1 : User Interaction Layer

**Fichier** : `src/screens/REPL.tsx` (810 lignes)

#### Responsabilités

- Interface terminal interactive avec Ink (React pour CLI)
- Gestion de l'état de la conversation
- Rendu des messages et composants UI
- Gestion des inputs utilisateur (3 modes)
- Orchestration des dialogues de permissions
- Affichage des coûts et statistiques

#### Composants Clés

```typescript
export function REPL({
  commands, tools, safeMode, verbose,
  initialMessages, mcpClients
}: Props) {
  // État principal
  const [messages, setMessages] = useState<MessageType[]>()
  const [isLoading, setIsLoading] = useState(false)
  const [inputMode, setInputMode] = useState<'prompt' | 'bash' | 'koding'>('prompt')
  const [toolUseConfirm, setToolUseConfirm] = useState<ToolUseConfirm>()

  // Boucle principale : onQuery → query() → setMessages
  async function onQuery(newMessages: MessageType[]) {
    setMessages(old => [...old, ...newMessages])

    for await (const msg of query(...)) {
      setMessages(old => [...old, msg])
    }
  }

  // Rendu optimisé Static/Transient
  return (
    <>
      <Static items={messagesJSX.filter(_ => _.type === 'static')}>
        {/* Messages finalisés */}
      </Static>
      {messagesJSX.filter(_ => _.type === 'transient').map(_ => _.jsx)}
      {/* Messages actifs */}
    </>
  )
}
```

#### Patterns Architecturaux

- **State Management** : Hooks React pour état local
- **Event-Driven** : Callback system pour interactions
- **Streaming UI** : Mise à jour progressive des messages
- **Component Composition** : Static vs Transient rendering

---

### ⚙️ Couche 2 : Orchestration Layer

**Fichiers** : `src/query.ts` + `src/tools/TaskTool/`

#### Responsabilités

##### `query.ts` : Boucle Récursive d'Exécution

```typescript
async function* query(
  messages: Message[],
  systemPrompt: string[],
  context: Record<string, string>,
  canUseTool: CanUseToolFn,
  toolUseContext: ExtendedToolUseContext
): AsyncGenerator<Message> {
  // 1. Auto-compaction si nécessaire
  const { messages: processedMessages, wasCompacted } =
    await checkAutoCompact(messages, toolUseContext)

  if (wasCompacted) {
    yield createAssistantMessage('Context auto-compacted...')
  }

  // 2. Construction du system prompt enrichi
  const { systemPrompt: fullSystemPrompt } =
    formatSystemPromptWithContext(systemPrompt, context, agentId)

  // 3. Appel LLM
  const assistantMessage = await queryLLM(
    processedMessages,
    fullSystemPrompt,
    maxThinkingTokens,
    tools,
    abortController.signal,
    { model, safeMode, toolUseContext }
  )

  yield assistantMessage

  // 4. Extraction des tool_use
  const toolUseMessages = assistantMessage.message.content.filter(
    _ => _.type === 'tool_use'
  )

  if (toolUseMessages.length === 0) {
    return  // Fin de la récursion
  }

  // 5. Exécution des outils
  const toolResults = []

  if (canRunConcurrently(toolUseMessages)) {
    // Concurrent
    for await (const msg of runToolsConcurrently(...)) {
      toolResults.push(msg)
      yield msg
    }
  } else {
    // Sequential
    for await (const msg of runToolsSerially(...)) {
      toolResults.push(msg)
      yield msg
    }
  }

  // 6. Récursion avec résultats
  yield* await query(
    [...processedMessages, assistantMessage, ...toolResults],
    systemPrompt,
    context,
    canUseTool,
    toolUseContext
  )
}
```

##### `TaskTool` : Délégation aux Agents

```typescript
// Charge dynamiquement la configuration de l'agent
const agentConfig = await getAgentByType('general-purpose')

// Priorité de chargement :
// 1. Built-in agents
// 2. ~/.claude/agents/
// 3. ~/.kode/agents/
// 4. ./.claude/agents/ (projet)
// 5. ./.kode/agents/ (projet) ← Priorité maximale

// Applique le filtrage d'outils
if (agentConfig.tools !== '*') {
  tools = tools.filter(t => agentConfig.tools.includes(t.name))
}

// Lance une sous-requête avec le contexte de l'agent
for await (const message of query(
  [createUserMessage(effectivePrompt)],
  taskPrompt,
  context,
  canUseTool,
  { ...toolUseContext, agentId: taskId, options: { tools, model: agentConfig.model_name } }
)) {
  yield { type: 'progress', content: message }
}
```

---

### 🔧 Couche 3 : Tool Execution Layer

**Répertoire** : `src/tools/*/`

#### Interface Unifiée

Tous les outils implémentent l'interface `Tool` définie dans `src/Tool.ts` :

```typescript
export interface Tool<TInput, TOutput> {
  // === MÉTADONNÉES ===
  name: string
  description?: () => Promise<string>  // ASYNC!
  userFacingName?: () => string

  // === SCHÉMA & VALIDATION ===
  inputSchema: z.ZodObject<any>
  inputJSONSchema?: Record<string, unknown>

  validateInput?: (
    input: z.infer<TInput>,
    context?: ToolUseContext
  ) => Promise<ValidationResult>

  // === PROMPTS ===
  prompt: (options?: { safeMode?: boolean }) => Promise<string>

  // === SÉCURITÉ & CAPACITÉS ===
  isEnabled: () => Promise<boolean>
  isReadOnly: () => boolean
  isConcurrencySafe: () => boolean
  needsPermissions: (input?: z.infer<TInput>) => boolean

  // === EXÉCUTION (AsyncGenerator pour streaming) ===
  call: (
    input: z.infer<TInput>,
    context: ToolUseContext
  ) => AsyncGenerator<
    | { type: 'result'; data: TOutput; resultForAssistant?: string }
    | { type: 'progress'; content: any; normalizedMessages?: any[] },
    void, unknown
  >

  // === AFFICHAGE ===
  renderResultForAssistant: (output: TOutput) => string | any[]
  renderToolUseMessage: (
    input: z.infer<TInput>,
    options: { verbose: boolean }
  ) => string
  renderToolResultMessage?: (output: TOutput) => React.ReactElement
}
```

#### Outils Disponibles (21)

| Catégorie | Outils | Description |
|-----------|--------|-------------|
| **Fichiers** | FileRead, FileWrite, FileEdit, MultiEdit, NotebookRead, NotebookEdit, LS, Glob | Manipulation de fichiers |
| **Recherche** | Grep | Recherche regex (ripgrep) |
| **Exécution** | Bash | Commandes shell |
| **Web** | WebSearch, URLFetcher | Recherche et fetch web |
| **Orchestration** | Task, Architect, AskExpertModel | Agents et consultation |
| **Mémoire** | MemoryRead, MemoryWrite | Persistance |
| **Utilitaires** | Think, TodoWrite | Métacognition et tâches |

#### Exemple : BashTool

```typescript
export const BashTool = {
  name: 'Bash',

  async description() {
    return DESCRIPTION
  },

  inputSchema: z.strictObject({
    command: z.string().describe('Shell command to execute'),
    timeout: z.number().optional().describe('Timeout in milliseconds')
  }),

  isEnabled: async () => true,
  isReadOnly: () => false,
  isConcurrencySafe: () => false,
  needsPermissions: (input) => !SAFE_COMMANDS.has(input?.command),

  async *call({ command, timeout = 120000 }, context) {
    // Vérifier annulation
    if (context.abortController.signal.aborted) {
      yield { type: 'result', data: { output: 'Cancelled', exitCode: 1 } }
      return
    }

    // Exécuter
    const result = await exec(command, { timeout, signal: context.abortController.signal })

    // Retourner résultat
    yield {
      type: 'result',
      data: {
        output: result.stdout,
        exitCode: result.exitCode
      },
      resultForAssistant: `Exit code: ${result.exitCode}\n${result.stdout}`
    }
  },

  renderToolUseMessage({ command }) {
    return command
  },

  renderResultForAssistant({ output, exitCode }) {
    return `Exit code: ${exitCode}\n${output}`
  }
} satisfies Tool<typeof inputSchema, Output>
```

---

## 4. Flux de Données et Interactions

### Cycle de Vie d'une Requête Utilisateur

```
┌─────────────────────────────────────────────────────────┐
│ 1. Input Utilisateur (REPL.tsx)                        │
│    User types: "Create a function to parse JSON"       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Traitement Initial (query.ts)                       │
│    - Auto-compact si contexte > 92% limite             │
│    - Construction system prompt + contexte projet      │
│      (AGENTS.md, CLAUDE.md, git status, etc.)          │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Appel LLM (services/claude.ts ou openai.ts)         │
│    const response = await queryLLM(                     │
│      normalizeMessagesForAPI(messages),                 │
│      fullSystemPrompt,                                  │
│      maxThinkingTokens,                                 │
│      tools,  // Schémas des outils disponibles          │
│      abortController.signal,                            │
│      { model: 'main', safeMode, toolUseContext }        │
│    )                                                    │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Extraction Tool Use                                 │
│    const toolUseMessages = response.content.filter(    │
│      block => block.type === 'tool_use'                │
│    )                                                   │
│    // Ex: [{ type: 'tool_use', name: 'FileWrite',     │
│    //         input: { file_path: 'parse.ts', ... }}] │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Vérification Permissions (permissions.ts)           │
│    const permissionResult = await canUseTool(          │
│      tool, normalizedInput, context, assistantMessage  │
│    )                                                   │
│    if (!permissionResult.result) {                     │
│      // Demande permission UI                          │
│      // User choisit: [a]llow, [r]eject, [p]ermanent  │
│    }                                                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Exécution Outil (via AsyncGenerator)                │
│    for await (const result of tool.call(input, ctx)) { │
│      switch (result.type) {                            │
│        case 'progress':                                │
│          yield createProgressMessage(...)  // UI temps │
│          break                             // réel     │
│        case 'result':                                  │
│          yield createUserMessage([{                    │
│            type: 'tool_result',                        │
│            content: result.data,                       │
│            tool_use_id: toolUseID                      │
│          }])                                           │
│          break                                         │
│      }                                                 │
│    }                                                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Récursion                                           │
│    Les résultats sont ajoutés aux messages et          │
│    query() est rappelé:                                │
│                                                        │
│    yield* await query(                                 │
│      [...messages, assistantMessage, ...toolResults],  │
│      systemPrompt,                                     │
│      context,                                          │
│      canUseTool,                                       │
│      toolUseContext                                    │
│    )                                                   │
│                                                        │
│    → Retour à l'étape 2 jusqu'à ce que l'IA           │
│      ne demande plus d'outils                          │
└─────────────────────────────────────────────────────────┘
```

### Exemple Concret

**Requête utilisateur** : "Create a function to parse JSON and save it to utils.ts"

**Exécution** :

1. **Appel 1** : LLM → `FileRead(file_path='utils.ts')`
2. **Résultat** : File content returned
3. **Appel 2** : LLM (avec contenu) → `FileEdit(file_path='utils.ts', old_string='...', new_string='...')`
4. **Résultat** : File edited successfully
5. **Appel 3** : LLM → "Done! I've added the parseJSON function to utils.ts"
6. **Fin** : Pas de tool_use → fin de la récursion

---

## 5. Points d'Entrée et Cycle de Vie

### Point d'Entrée Principal

**Fichier** : `src/entrypoints/cli.tsx`

```typescript
async function main() {
  // 1. Initialisation Debug Logger
  initDebugLogger()

  // 2. Validation & Réparation Config
  enableConfigs()
  validateAndRepairAllGPT5Profiles()

  // 3. Parse Arguments (Commander.js)
  await parseArgs(stdinContent, renderContext)
}

async function parseArgs() {
  const program = new Command()
    .name('kode')
    .argument('[prompt]', 'Your prompt')
    .option('-c, --cwd <cwd>', 'Working directory')
    .option('--safe', 'Enable strict permissions')
    .option('-p, --print', 'Non-interactive mode')
    .action(async (prompt, options) => {
      // 4. Setup Environment
      await setup(options.cwd, options.safe)

      // 5. Load Tools & MCP Clients
      const [tools, mcpClients] = await Promise.all([
        getTools(enableArchitect),
        getClients()
      ])

      // 6. Mode Sélection
      if (options.print) {
        // Mode non-interactif
        const { resultText } = await ask({ prompt, tools, ... })
        console.log(resultText)
      } else {
        // Mode interactif : REPL
        const { render } = await import('ink')
        const { REPL } = await import('@screens/REPL')
        render(<REPL tools={tools} mcpClients={mcpClients} ... />)
      }
    })
}
```

### Fonction `setup()`

```typescript
async function setup(cwd: string, safeMode?: boolean) {
  // 1. Set Working Directory
  await setCwd(cwd)
  grantReadPermissionForOriginalDir()

  // 2. Start Agent File Watcher (hot reload)
  const { startAgentWatcher } = await import('@utils/agentLoader')
  await startAgentWatcher(() => {
    console.log('✅ Agent configurations hot-reloaded')
  })

  // 3. Security Check (safe mode)
  if (safeMode && process.getuid?.() === 0) {
    console.error('--safe mode cannot be used with root')
    process.exit(1)
  }

  // 4. Context Pre-fetch (parallèle)
  getContext() // Charge AGENTS.md, CLAUDE.md, git status, etc.

  // 5. Config Migration (backward compatibility)
  migrateOldConfigs()
}
```

### Chargement des Outils

**Fichier** : `src/tools.ts`

```typescript
export const getTools = memoize(async (enableArchitect?: boolean) => {
  // 1. Outils natifs
  const tools = [
    TaskTool, BashTool, FileReadTool, FileEditTool,
    GrepTool, GlobTool, WebSearchTool, ...
  ]

  // 2. Outils MCP (dynamiques)
  tools.push(...await getMCPTools())

  // 3. Outils optionnels
  if (enableArchitect) {
    tools.push(ArchitectTool)
  }

  // 4. Filtrage par disponibilité
  const isEnabled = await Promise.all(tools.map(t => t.isEnabled()))
  return tools.filter((_, i) => isEnabled[i])
})
```

---

## 6. Interactions Entre Modules

### Diagramme de Dépendances Critiques

```
┌─────────────────┐
│   REPL.tsx      │
└────────┬────────┘
         │ utilise
         ▼
┌─────────────────┐      ┌──────────────────┐
│   query.ts      │─────▶│  services/       │
│                 │      │  - claude.ts     │
└────────┬────────┘      │  - openai.ts     │
         │               │  - mcpClient.ts  │
         │ appelle       └──────────────────┘
         ▼
┌─────────────────┐      ┌──────────────────┐
│   tools/*       │─────▶│  utils/          │
│   (21 tools)    │      │  - config.ts     │
└────────┬────────┘      │  - model.ts      │
         │               │  - messages.ts   │
         │ utilise       │  - agentLoader.ts│
         ▼               └──────────────────┘
┌─────────────────┐
│  permissions.ts │
└─────────────────┘
```

### Exemple : TaskTool → Agent Loader → Tools

```typescript
// 1. TaskTool reçoit une demande
TaskTool.call({
  subagent_type: 'general-purpose',
  prompt: 'Find all TypeScript files'
})

// 2. Charge la configuration de l'agent
const agentConfig = await getAgentByType('general-purpose')
// Priorité : .kode/agents/ > .claude/agents/ > ~/.kode/agents/ > built-in

// 3. Applique le filtrage d'outils
let tools = await getTaskTools(safeMode)
if (agentConfig.tools !== '*') {
  tools = tools.filter(t => agentConfig.tools.includes(t.name))
}

// 4. Construit le prompt enrichi
const effectivePrompt = `${agentConfig.systemPrompt}\n\n${prompt}`

// 5. Lance une sous-requête query()
for await (const message of query(
  [createUserMessage(effectivePrompt)],
  taskPrompt,
  context,
  canUseTool,
  { ...toolUseContext, agentId: taskId, options: { tools, model: agentConfig.model_name } }
)) {
  yield { type: 'progress', content: message }
}
```

---

## 7. Patterns Architecturaux

### 1. Async Generator Pattern (Streaming)

**Utilisation** : Tous les outils et `query()`

```typescript
async function* query(...): AsyncGenerator<Message> {
  yield assistantMessage          // Message immédiat

  for await (const msg of runTools(...)) {
    yield msg                      // Progress reports
  }

  yield* await query(...)           // Récursion
}
```

**Avantages** :
- ✅ UI progressive (pas d'attente bloquante)
- ✅ Annulation possible à tout moment
- ✅ Memory-efficient pour longues conversations

---

### 2. Factory Pattern (Model Adapters)

**Fichier** : `src/services/modelAdapterFactory.ts`

```typescript
export class ModelAdapterFactory {
  static async createAdapter(provider: ProviderType, config: ModelProfile) {
    switch (provider) {
      case 'anthropic':
        return new AnthropicAdapter(config)
      case 'openai':
        return new OpenAIAdapter(config)
      case 'custom-openai':
        return new CustomOpenAIAdapter(config)
      // ... 15+ providers
    }
  }
}
```

**Avantages** :
- ✅ Interface unifiée pour tous les modèles
- ✅ Ajout de providers sans toucher core logic
- ✅ Gestion des spécificités (GPT-5, streaming, etc.)

---

### 3. Strategy Pattern (Context Retention)

**Fichier** : `src/utils/messageContextManager.ts`

```typescript
interface MessageRetentionStrategy {
  type: 'preserve_recent' | 'preserve_important' | 'smart_compression' | 'auto_compact'
  maxTokens: number
}

class MessageContextManager {
  async truncateMessages(messages: Message[], strategy: MessageRetentionStrategy) {
    switch (strategy.type) {
      case 'preserve_recent':
        return this.preserveRecentMessages(messages, strategy)
      case 'preserve_important':
        return this.preserveImportantMessages(messages, strategy)
      case 'auto_compact':
        return this.autoCompactStrategy(messages, strategy)
    }
  }
}
```

---

### 4. Repository Pattern (Configuration)

**Fichier** : `src/utils/config.ts`

```typescript
// Hiérarchie de configuration :
// 1. CLI params (--cwd, --safe)
// 2. Environment variables (ANTHROPIC_API_KEY)
// 3. Project config (.kode.json)
// 4. Global config (~/.kode.json)

export function getGlobalConfig(): GlobalConfig {
  // Lit ~/.kode.json avec fallback sur defaults
}

export function getCurrentProjectConfig(): ProjectConfig {
  // Lit ./.kode.json avec fallback sur global
}

export function saveGlobalConfig(config: GlobalConfig) {
  // Écrit ~/.kode.json atomiquement
}
```

---

### 5. Observer Pattern (Agent Hot Reload)

**Fichier** : `src/utils/agentLoader.ts`

```typescript
let watchers: FSWatcher[] = []

export async function startAgentWatcher(callback: () => void) {
  const dirs = [
    join(homedir(), '.claude/agents'),
    join(homedir(), '.kode/agents'),
    join(getCwd(), '.claude/agents'),
    join(getCwd(), '.kode/agents')
  ]

  for (const dir of dirs) {
    if (existsSync(dir)) {
      watchers.push(watch(dir, { recursive: true }, () => {
        clearAgentCache()
        callback()
      }))
    }
  }
}
```

---

### 6. Dependency Injection (Tool Context)

**Utilisation** : Tous les outils reçoivent `ToolUseContext`

```typescript
interface ToolUseContext {
  messageId: string
  agentId?: string
  safeMode?: boolean
  abortController: AbortController
  readFileTimestamps: Record<string, number>
  options: {
    commands: Command[]
    tools: Tool[]
    verbose: boolean
    model: string | ModelPointerType
    maxThinkingTokens: number
  }
  responseState?: {
    previousResponseId?: string
    conversationId?: string
  }
}

// Injection dans chaque call
await tool.call(input, toolUseContext)
```

---

### 7. Memento Pattern (Conversation Recovery)

**Fichier** : `src/utils/log.ts` + `src/utils/conversationRecovery.ts`

```typescript
// Sauvegarde automatique :
// ~/.kode/messages/{date}_{fork}.log

export function useLogMessages(messages: Message[], logName: string, fork: number) {
  useEffect(() => {
    const path = getMessagesPath(logName, fork)
    writeFileSync(path, JSON.stringify(messages, null, 2))
  }, [messages, logName, fork])
}

// Restauration :
// kode resume 0  // Reprend conversation #0
```

---

## 8. Points Clés d'Architecture

### 1. Gestion Multi-Modèle Sophistiquée

**ModelManager** (`src/utils/model.ts`) :

```typescript
export class ModelManager {
  // System de pointeurs : main, task, reasoning, quick
  getModelName(pointer: ModelPointerType): string {
    const pointerValue = this.config.modelPointers?.[pointer]
    if (pointerValue) {
      const profile = this.findModelProfile(pointerValue)
      if (profile?.isActive) return profile.modelName
    }
    return this.getMainAgentModel()
  }

  // Validation & Auto-repair GPT-5
  validateAndRepairAllGPT5Profiles() {
    for (const profile of this.modelProfiles) {
      if (profile.modelName.startsWith('gpt-5')) {
        // Convertir maxTokens → maxCompletionTokens
        // Ajouter reasoningEffort par défaut
        // Marquer isGPT5: true
      }
    }
  }
}
```

**Support** : Anthropic, OpenAI, Mistral, DeepSeek, Gemini, Ollama, Azure, +15 providers

---

### 2. Système de Permissions Granulaire

**Niveaux** :
- **Bypass** : Mode permissif (défaut)
- **Safe** : Mode strict avec approbation manuelle

**Exemples** :
```typescript
// Bash : permissions par préfixe de commande
"Bash(git:*)"           // Autorise toutes les commandes git
"Bash(git status)"      // Autorise seulement git status

// FileEdit : permissions par chemin
"FileEdit(/home/user/project/**)"  // Autorise tout dans project/

// Stockage : .kode.json (projet) ou ~/.kode.json (global)
{
  "allowedTools": [
    "Bash(git:*)",
    "FileEdit(/home/user/project/**)",
    "WebSearch"
  ]
}
```

---

### 3. Context Window Management

**Problème** : Conversations longues dépassent limite de tokens

**Solutions** :

1. **Auto-Compact** (`src/utils/autoCompactCore.ts`) :
   ```typescript
   async function checkAutoCompact(messages: Message[], context: ToolUseContext) {
     const tokenCount = countTokens(messages)
     const model = getModelManager().getModel('main')
     const contextLimit = model.contextLength

     if (tokenCount > contextLimit * 0.92) {  // Seuil 92%
       // Compaction automatique
       const { truncatedMessages } = await messageContextManager.truncateMessages(
         messages,
         { type: 'preserve_important', maxTokens: contextLimit * 0.6 }
       )
       return { messages: truncatedMessages, wasCompacted: true }
     }
     return { messages, wasCompacted: false }
   }
   ```

2. **Stratégies de rétention** :
   - `preserve_recent` : Garde N derniers messages
   - `preserve_important` : Garde erreurs + messages clés
   - `smart_compression` : Résumés intelligents
   - `auto_compact` : Compaction transparente

---

### 4. Système de Rappels Contextuels (System Reminders)

**Fichier** : `src/services/systemReminder.ts`

```typescript
// Injecte des rappels dynamiques dans le system prompt
export function generateSystemReminders(context: {
  agentId?: string,
  messageCount: number,
  tools: Tool[]
}): string {
  const reminders = []

  // Rappel sur claude.md si présent
  if (claudeMdExists()) {
    reminders.push('<system-reminder>\nIMPORTANT: Read CLAUDE.md for project-specific instructions.\n</system-reminder>')
  }

  // Rappel sur tools disponibles
  reminders.push(`<system-reminder>\nAvailable tools: ${tools.map(t => t.name).join(', ')}\n</system-reminder>`)

  return reminders.join('\n')
}
```

---

### 5. Agent Loader : Compatibilité Claude Code

**Hiérarchie de chargement** (ordre de priorité) :

```
1. Built-in agents (code)
2. ~/.claude/agents/*.md  (utilisateur Claude Code)
3. ~/.kode/agents/*.md    (utilisateur Kode)
4. ./.claude/agents/*.md  (projet Claude Code)
5. ./.kode/agents/*.md    (projet Kode, priorité maximale)
```

**Format d'agent** :
```markdown
---
name: general-purpose
description: "General-purpose agent for complex tasks"
tools: "*"
model_name: "claude-sonnet-4-20250514"
---

You are a general-purpose agent. Complete tasks using available tools.
```

**Hot Reload** : Watchers sur tous les répertoires, cache invalidé automatiquement

---

### 6. MCP Integration (Model Context Protocol)

**Architecture** :
- **Serveur MCP** : `kode mcp serve` (stdio/SSE)
- **Client MCP** : Charge outils dynamiques depuis serveurs configurés

**Configuration** :
```json
// .kode.json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "env": { "NODE_ENV": "production" }
    },
    "github": {
      "type": "sse",
      "url": "https://mcp.example.com/github"
    }
  }
}
```

**Flux** :
1. `getClients()` → Charge serveurs MCP
2. `getMCPTools()` → Expose outils MCP comme Tools natifs
3. LLM peut utiliser outils MCP comme n'importe quel autre outil

---

## Conclusion

L'architecture de Kode est un **système sophistiqué et bien pensé** qui combine :

1. **🎨 UI réactive** (Ink/React) pour une expérience CLI moderne
2. **⚙️ Orchestration intelligente** (query loop récursive) pour exécution multi-étapes
3. **🔧 Système d'outils modulaire** avec 21+ outils natifs + MCP
4. **🤖 Gestion multi-modèle** avec support de 20+ providers
5. **🎭 Agents spécialisés** chargeables dynamiquement
6. **📊 Context management** avec auto-compaction et stratégies de rétention
7. **🔐 Permissions granulaires** pour sécurité
8. **🔄 Compatibilité Claude Code** pour écosystème `.claude`

L'architecture suit des **patterns éprouvés** (Factory, Strategy, Observer, Dependency Injection) tout en innovant sur la **délégation d'agents** et la **gestion de contexte conversationnel**.

**Fichiers Clés à Comprendre** (par ordre d'importance) :
1. `src/screens/REPL.tsx` - Point central UI
2. `src/query.ts` - Cœur de l'orchestration
3. `src/Tool.ts` - Interface de base
4. `src/tools/TaskTool/TaskTool.tsx` - Système d'agents
5. `src/services/claude.ts` - Intégration LLM
6. `src/utils/model.ts` - Gestion multi-modèle
7. `src/utils/config.ts` - Configuration hiérarchique
8. `src/utils/agentLoader.ts` - Chargement dynamique d'agents

---

[← Retour à l'index](./README.md) | [Suivant : Système Multi-Modèles →](./03-systeme-modeles.md)
