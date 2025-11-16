# Système de Configuration - Hiérarchie et Auto-Compaction

> Documentation technique du système de configuration de Kode : 4 niveaux hiérarchiques, auto-compaction à 92%, model profiles, et gestion intelligente du contexte

---

## Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Hiérarchie de Configuration](#2-hiérarchie-de-configuration)
3. [Format des Fichiers](#3-format-des-fichiers)
4. [Configuration Globale](#4-configuration-globale)
5. [Configuration Projet](#5-configuration-projet)
6. [Système Multi-Modèles](#6-système-multi-modèles)
7. [Auto-Compaction](#7-auto-compaction)
8. [Gestion du Contexte](#8-gestion-du-contexte)
9. [API et Fonctions](#9-api-et-fonctions)

---

## 1. Vue d'Ensemble

### 1.1 Philosophie

Le système de configuration de Kode suit plusieurs principes clés :

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  1. HIÉRARCHIE CLAIRE                          │
│     CLI → ENV → Project → Global               │
│     (priorité décroissante)                     │
│                                                 │
│  2. SAUVEGARDE INTELLIGENTE                    │
│     Seules les valeurs != defaults sauvées     │
│     Fichiers JSON compacts                     │
│                                                 │
│  3. ISOLATION PAR PROJET                       │
│     Chaque projet a sa config                  │
│     Pas de pollution globale                   │
│                                                 │
│  4. AUTO-RÉPARATION                            │
│     Détection erreurs de config                │
│     Fallback sur defaults                      │
│     Logging détaillé                           │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 1.2 Fichiers de Configuration

| Fichier | Emplacement | Scope | Priorité |
|---------|-------------|-------|----------|
| **Arguments CLI** | N/A | Session | 1 (Max) |
| **Variables d'env** | N/A | Session | 2 |
| **.kode.json** (projet) | `$(pwd)/.kode.json` | Projet | 3 |
| **.mcprc** | `$(pwd)/.mcprc` | MCP Projet | 3.5 |
| **~/.kode.json** | `~/.kode.json` | Global | 4 (Min) |

### 1.3 Fichier Principal

**Fichier** : `src/utils/config.ts` (939 lignes)

**Responsabilités** :
- Chargement et sauvegarde de configuration
- Merge hiérarchique
- Validation et migration
- Model profiles management
- MCP servers configuration
- Project-specific settings

---

## 2. Hiérarchie de Configuration

### 2.1 Les 4 Niveaux

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  NIVEAU 1 : ARGUMENTS CLI                      │
│  ─────────────────────────────                 │
│  kode --model=claude-opus-4 --verbose          │
│                                                 │
│  Priorité  : MAXIMALE                          │
│  Persistance : NON (session uniquement)        │
│  Scope    : Command invocation                 │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  NIVEAU 2 : VARIABLES D'ENVIRONNEMENT          │
│  ─────────────────────────────────────         │
│  export ANTHROPIC_API_KEY=sk-...               │
│  export DISABLE_PROMPT_CACHING=true            │
│  export MAX_THINKING_TOKENS=10000              │
│                                                 │
│  Priorité  : HAUTE                             │
│  Persistance : Session shell                   │
│  Scope    : Environment                        │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  NIVEAU 3 : CONFIGURATION PROJET               │
│  ─────────────────────────────────             │
│  Fichier : $(pwd)/.kode.json                   │
│  Contenu : {                                   │
│    "allowedTools": ["Bash(npm:*)"],           │
│    "enableArchitectTool": true,               │
│    "mcpServers": { ... }                      │
│  }                                              │
│                                                 │
│  Priorité  : MOYENNE                           │
│  Persistance : Permanent (versionné)           │
│  Scope    : Projet spécifique                  │
│                                                 │
│  ──────────────────────                        │
│  Fichier : $(pwd)/.mcprc                       │
│  Contenu : {                                   │
│    "server-name": { ... }                     │
│  }                                              │
│                                                 │
│  Priorité  : MOYENNE                           │
│  Persistance : Permanent (versionné)           │
│  Scope    : MCP servers projet                 │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  NIVEAU 4 : CONFIGURATION GLOBALE              │
│  ─────────────────────────────────             │
│  Fichier : ~/.kode.json                        │
│  Contenu : {                                   │
│    "modelProfiles": [...],                    │
│    "modelPointers": { ... },                  │
│    "theme": "dark",                           │
│    "verbose": false,                          │
│    "projects": {                              │
│      "/path/to/project": { ... }             │
│    }                                          │
│  }                                              │
│                                                 │
│  Priorité  : MINIMALE (fallback)               │
│  Persistance : Permanent (non versionné)       │
│  Scope    : Utilisateur                        │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 2.2 Résolution de Configuration

```typescript
function resolveConfig(key: string): any {
  // 1. CLI args (runtime)
  if (cliArgs[key] !== undefined) {
    return cliArgs[key]
  }

  // 2. Environment variables
  if (process.env[`KODE_${key.toUpperCase()}`] !== undefined) {
    return process.env[`KODE_${key.toUpperCase()}`]
  }

  // 3. Project config (.kode.json in pwd)
  const projectConfig = getCurrentProjectConfig()
  if (projectConfig[key] !== undefined) {
    return projectConfig[key]
  }

  // 4. Global config (~/.kode.json)
  const globalConfig = getGlobalConfig()
  if (globalConfig[key] !== undefined) {
    return globalConfig[key]
  }

  // 5. Default value
  return DEFAULT_CONFIG[key]
}
```

### 2.3 Exemple Pratique

```bash
# Scénario : Configuration du modèle

# Global (~/.kode.json)
{
  "modelPointers": {
    "main": "claude-sonnet-4"
  }
}

# Projet (/project/.kode.json)
{
  "modelPointers": {
    "main": "gpt-4-turbo"  # Override global
  }
}

# Environment
export KODE_MODEL="claude-opus-4"  # Override projet

# CLI
kode --model=deepseek-chat  # Override tout

# Résultat final : "deepseek-chat"
```

---

## 3. Format des Fichiers

### 3.1 Structure GlobalConfig

**Fichier** : `~/.kode.json`

```typescript
export type GlobalConfig = {
  // Système
  numStartups: number                    // Nombre de démarrages
  userID?: string                        // ID utilisateur unique
  theme: ThemeNames                      // Thème UI (dark/light)
  verbose: boolean                       // Mode verbeux
  stream?: boolean                       // Streaming activé
  proxy?: string                         // Proxy HTTP

  // Onboarding
  hasCompletedOnboarding?: boolean       // Onboarding fait
  lastOnboardingVersion?: string         // Dernière version onboarding
  lastReleaseNotesSeen?: string         // Dernières release notes

  // Auto-updater
  autoUpdaterStatus?: AutoUpdaterStatus  // disabled|enabled|...
  lastDismissedUpdateVersion?: string    // Dernière version ignorée

  // Notifications
  preferredNotifChannel: NotificationChannel // iterm2|terminal_bell|...
  iterm2KeyBindingInstalled?: boolean    // Legacy
  shiftEnterKeyBindingInstalled?: boolean // Shift+Enter installé

  // API
  primaryProvider?: ProviderType         // Provider par défaut
  maxTokens?: number                     // Tokens max (override)
  hasAcknowledgedCostThreshold?: boolean // Alerte coût vue
  customApiKeyResponses?: {
    approved?: string[]                  // API keys approuvées
    rejected?: string[]                  // API keys rejetées
  }
  oauthAccount?: AccountInfo             // Compte OAuth

  // Model System (Innovation)
  modelProfiles?: ModelProfile[]         // Liste des modèles
  modelPointers?: ModelPointers          // Pointeurs main/task/...
  defaultModelName?: string              // Modèle par défaut

  // MCP Servers
  mcpServers?: Record<string, McpServerConfig> // Serveurs MCP globaux

  // Projects (stockés dans le même fichier)
  projects?: Record<string, ProjectConfig> // Config par projet
}
```

**Exemple** :
```json
{
  "numStartups": 42,
  "theme": "dark",
  "verbose": false,
  "modelProfiles": [
    {
      "name": "Claude Sonnet 4",
      "provider": "anthropic",
      "modelName": "claude-sonnet-4",
      "apiKey": "sk-ant-...",
      "maxTokens": 8192,
      "contextLength": 200000,
      "isActive": true,
      "createdAt": 1705401600000
    },
    {
      "name": "GPT-4 Turbo",
      "provider": "openai",
      "modelName": "gpt-4-turbo",
      "apiKey": "sk-...",
      "maxTokens": 4096,
      "contextLength": 128000,
      "isActive": true,
      "createdAt": 1705401700000
    }
  ],
  "modelPointers": {
    "main": "claude-sonnet-4",
    "task": "gpt-4-turbo",
    "reasoning": "claude-opus-4",
    "quick": "claude-haiku"
  },
  "projects": {
    "/home/user/my-project": {
      "allowedTools": ["Bash(git:*)", "Bash(npm:*)"],
      "enableArchitectTool": true,
      "hasTrustDialogAccepted": true
    }
  }
}
```

### 3.2 Structure ProjectConfig

**Scope** : Projet spécifique, stocké dans `globalConfig.projects[projectPath]`

```typescript
export type ProjectConfig = {
  // Permissions
  allowedTools: string[]                 // Outils autorisés
  hasTrustDialogAccepted?: boolean       // Trust dialog accepté

  // Features
  enableArchitectTool?: boolean          // Activer ArchitectTool
  dontCrawlDirectory?: boolean           // Désactiver crawl (ex: ~/)

  // Contexte
  context: Record<string, string>        // Contexte custom
  contextFiles?: string[]                // Fichiers de contexte
  exampleFiles?: string[]                // Fichiers d'exemple
  exampleFilesGeneratedAt?: number       // Timestamp génération

  // MCP
  mcpServers?: Record<string, McpServerConfig> // MCP servers projet
  mcpContextUris: string[]               // URIs de contexte MCP
  approvedMcprcServers?: string[]        // Serveurs .mcprc approuvés
  rejectedMcprcServers?: string[]        // Serveurs .mcprc rejetés

  // Historique
  history: string[]                      // Historique de commandes

  // Metrics
  lastAPIDuration?: number               // Durée dernière API call
  lastCost?: number                      // Coût dernière requête
  lastDuration?: number                  // Durée totale dernière requête
  lastSessionId?: string                 // ID dernière session

  // Onboarding
  hasCompletedProjectOnboarding?: boolean // Onboarding projet fait
}
```

### 3.3 ModelProfile

**Description** : Configuration d'un modèle IA

```typescript
export type ModelProfile = {
  // Identification
  name: string                           // Nom affiché (ex: "Claude Sonnet 4")
  modelName: string                      // ID modèle (PRIMARY KEY)
  provider: ProviderType                 // Provider (anthropic, openai, ...)

  // API
  apiKey: string                         // Clé API
  baseURL?: string                       // Endpoint custom (optionnel)

  // Limites
  maxTokens: number                      // Tokens de sortie max
  contextLength: number                  // Fenêtre de contexte

  // GPT-5 Responses API (optionnel)
  reasoningEffort?: 'low' | 'medium' | 'high' // Effort de reasoning
  isGPT5?: boolean                       // Flag GPT-5 auto-détecté
  validationStatus?: 'valid' | 'needs_repair' | 'auto_repaired'
  lastValidation?: number                // Timestamp validation

  // État
  isActive: boolean                      // Modèle activé
  createdAt: number                      // Timestamp création
  lastUsed?: number                      // Timestamp dernière utilisation
}
```

**Exemple** :
```json
{
  "name": "GPT-5 Preview",
  "modelName": "gpt-5-preview",
  "provider": "openai",
  "apiKey": "sk-...",
  "maxTokens": 16384,
  "contextLength": 200000,
  "reasoningEffort": "high",
  "isGPT5": true,
  "validationStatus": "valid",
  "isActive": true,
  "createdAt": 1705401600000,
  "lastUsed": 1705488000000
}
```

### 3.4 ModelPointers

**Description** : Système de pointeurs pour sélection rapide de modèles

```typescript
export type ModelPointers = {
  main: string       // Modèle principal (dialog)
  task: string       // Modèle pour TaskTool (agents)
  reasoning: string  // Modèle pour raisonnement complexe
  quick: string      // Modèle rapide/économique
}
```

**Exemple** :
```json
{
  "main": "claude-sonnet-4",
  "task": "gpt-4-turbo",
  "reasoning": "claude-opus-4",
  "quick": "claude-haiku"
}
```

**Usage** :
```typescript
// Obtenir le modèle main
const modelManager = getModelManager()
const mainModel = modelManager.getModelName('main')
// → "claude-sonnet-4"

// Changer le pointeur main
modelManager.setModelPointer('main', 'gpt-5-preview')
```

### 3.5 McpServerConfig

**Description** : Configuration d'un serveur MCP

**Type 1 : Stdio** (processus local)
```typescript
{
  type?: 'stdio',
  command: string,
  args: string[],
  env?: Record<string, string>
}
```

**Type 2 : SSE** (serveur HTTP)
```typescript
{
  type: 'sse',
  url: string
}
```

**Exemple Stdio** :
```json
{
  "filesystem": {
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"],
    "env": {
      "NODE_ENV": "production"
    }
  }
}
```

**Exemple SSE** :
```json
{
  "my-api": {
    "type": "sse",
    "url": "http://localhost:3000/mcp"
  }
}
```

### 3.6 .mcprc

**Fichier** : `.mcprc` (à la racine du projet)

**Format** : Identique à `mcpServers` dans `.kode.json`

**Différence** :
- `.kode.json` → `mcpServers` : Global ou projet (Kode-managed)
- `.mcprc` : Projet uniquement (user-managed, versionnable)
- Les deux sont mergés, `.mcprc` pour faciliter le versioning

**Exemple** :
```json
{
  "github": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {
      "GITHUB_TOKEN": "${GITHUB_TOKEN}"
    }
  }
}
```

---

## 4. Configuration Globale

### 4.1 Fichier ~/.kode.json

**Emplacement** : `~/.kode.json` (ou défini par `GLOBAL_CLAUDE_FILE`)

**Permissions** : Lecture/Écriture utilisateur uniquement

**Taille typique** : 5-20 KB (selon nombre de projets et modèles)

### 4.2 Champs Clés

#### 4.2.1 Système

```json
{
  "numStartups": 42,
  "userID": "uuid-v4-here",
  "theme": "dark",
  "verbose": false,
  "stream": true
}
```

#### 4.2.2 Modèles

```json
{
  "modelProfiles": [
    { "name": "...", "modelName": "...", ... }
  ],
  "modelPointers": {
    "main": "claude-sonnet-4",
    "task": "gpt-4-turbo",
    "reasoning": "claude-opus-4",
    "quick": "claude-haiku"
  },
  "defaultModelName": "claude-sonnet-4",
  "primaryProvider": "anthropic",
  "maxTokens": 8192
}
```

#### 4.2.3 Projets

```json
{
  "projects": {
    "/home/user/project-1": {
      "allowedTools": ["Bash(git:*)", "Bash(npm:*)"],
      "enableArchitectTool": true,
      "hasTrustDialogAccepted": true,
      "history": ["git status", "npm test"]
    },
    "/home/user/project-2": {
      "allowedTools": [],
      "dontCrawlDirectory": false,
      "mcpServers": {
        "local-db": { ... }
      }
    }
  }
}
```

### 4.3 Sauvegarde Intelligente

**Principe** : Seules les valeurs différentes des defaults sont sauvées

```typescript
function saveConfig<A extends object>(
  file: string,
  config: A,
  defaultConfig: A,
): void {
  // Filtre les valeurs = defaults
  const filteredConfig = Object.fromEntries(
    Object.entries(config).filter(
      ([key, value]) =>
        JSON.stringify(value) !== JSON.stringify(defaultConfig[key]),
    ),
  )

  writeFileSync(file, JSON.stringify(filteredConfig, null, 2), 'utf-8')
}
```

**Exemple** :

```typescript
// Config complète en mémoire
{
  numStartups: 42,
  theme: "dark",        // = default
  verbose: false,       // = default
  modelProfiles: [...]
}

// Sauvegardé dans ~/.kode.json
{
  "numStartups": 42,     // ≠ default (0)
  "modelProfiles": [...] // ≠ default ([])
  // theme et verbose omis (= defaults)
}
```

**Avantages** :
- ✅ Fichiers compacts
- ✅ Diffs Git clairs
- ✅ Mise à jour des defaults transparente
- ✅ Merge facile

---

## 5. Configuration Projet

### 5.1 Isolation par Projet

Chaque projet a sa propre configuration, stockée dans `globalConfig.projects[absolutePath]`.

**Résolution** :
```typescript
export function getCurrentProjectConfig(): ProjectConfig {
  const absolutePath = resolve(getCwd())
  const config = getGlobalConfig()

  const projectConfig = config.projects?.[absolutePath]
    ?? defaultConfigForProject(absolutePath)

  return projectConfig
}
```

### 5.2 Permissions Outils

**Stockage** : `allowedTools: string[]`

**Format** :
- Outil simple : `"WebSearch"`
- Outil avec input : `"Bash(git status)"`
- Outil avec préfixe : `"Bash(npm:*)"`

**Exemples** :
```json
{
  "allowedTools": [
    "WebSearch",              // Outil complet
    "Bash(git status)",       // Commande exacte
    "Bash(git diff)",         // Commande exacte
    "Bash(git:*)",            // Toutes commandes git
    "Bash(npm:*)",            // Toutes commandes npm
    "Bash(docker:*)"          // Toutes commandes docker
  ]
}
```

### 5.3 Contexte Projet

**contextFiles** : Fichiers chargés dans le contexte système

```json
{
  "contextFiles": [
    ".kode/context.md",
    "ARCHITECTURE.md",
    "API.md"
  ]
}
```

**context** : Clé-valeur personnalisées

```json
{
  "context": {
    "project_type": "monorepo",
    "package_manager": "pnpm",
    "test_framework": "vitest"
  }
}
```

### 5.4 Serveurs MCP Projet

```json
{
  "mcpServers": {
    "postgres": {
      "command": "docker",
      "args": ["run", "-i", "postgres-mcp-server"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/project_db"
      }
    }
  },
  "approvedMcprcServers": ["github", "gitlab"],
  "rejectedMcprcServers": ["untrusted-server"]
}
```

---

## 6. Système Multi-Modèles

Voir [03-systeme-modeles.md](./03-systeme-modeles.md) pour détails complets.

### 6.1 Gestion des Profiles

```typescript
// Ajouter un modèle
const profile: ModelProfile = {
  name: "My Custom Model",
  modelName: "custom-model-id",
  provider: "custom-openai",
  baseURL: "https://api.custom.com/v1",
  apiKey: "sk-...",
  maxTokens: 4096,
  contextLength: 32000,
  isActive: true,
  createdAt: Date.now()
}

const config = getGlobalConfig()
config.modelProfiles.push(profile)
saveGlobalConfig(config)
```

### 6.2 Model Pointers

```typescript
// Obtenir le modèle actuel
const modelManager = getModelManager()
const mainModel = modelManager.getModelName('main')

// Changer le modèle
modelManager.setModelPointer('main', 'gpt-5-preview')

// Lister les modèles disponibles
const profiles = modelManager.listModels()
```

---

## 7. Auto-Compaction

### 7.1 Principe

**Problème** : Contexte limité des modèles (ex: 200K tokens)

**Solution** : Auto-compaction à **92%** du contexte

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  DÉCLENCHEMENT                                  │
│  ──────────────                                │
│                                                 │
│  Tokens utilisés / Context Length ≥ 0.92       │
│                                                 │
│  Exemple :                                      │
│  184,000 / 200,000 = 0.92 → TRIGGER            │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  STRATÉGIE                                      │
│  ──────────────                                │
│                                                 │
│  1. Préserver messages récents (5 derniers)    │
│  2. Préserver messages importants :            │
│     - Erreurs d'outils                         │
│     - Décisions utilisateur                    │
│     - Contexte projet                          │
│  3. Résumer le reste avec AI                   │
│  4. Créer message de récupération contexte     │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  RÉSULTAT                                       │
│  ──────────────                                │
│                                                 │
│  Conversation compactée ~50-70% moins de tokens│
│  Contexte important préservé                   │
│  Continuation transparente pour l'utilisateur   │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 7.2 Implémentation

**Fichier** : `src/utils/autoCompactCore.ts`

**Fonction principale** :
```typescript
export async function tryAutoCompact(
  messages: Message[],
  modelInfo: { contextLength: number },
  toolUseContext: ToolUseContext
): Promise<AutoCompactResult> {
  // 1. Calculer tokens
  const totalTokens = countTokens(messages)
  const threshold = modelInfo.contextLength * 0.92

  // 2. Vérifier besoin de compaction
  if (totalTokens < threshold) {
    return {
      needsCompaction: false,
      messages
    }
  }

  // 3. Exécuter compaction
  const compactedMessages = await executeAutoCompact(
    messages,
    toolUseContext
  )

  return {
    needsCompaction: true,
    messages: compactedMessages
  }
}
```

### 7.3 Stratégie de Compaction

```typescript
async function executeAutoCompact(
  messages: Message[],
  context: ToolUseContext
): Promise<Message[]> {
  // 1. Préserver les 5 derniers messages (contexte récent)
  const recentMessages = messages.slice(-5)

  // 2. Identifier messages importants dans l'historique
  const importantMessages = messages.slice(0, -5).filter(msg =>
    isImportantMessage(msg)
  )

  // 3. Résumer le reste avec AI
  const summaryMessage = await summarizeMessages(
    messages.slice(0, -5).filter(msg => !isImportantMessage(msg)),
    context
  )

  // 4. Créer message de récupération de fichiers
  const recoveryMessage = await createFileRecoveryMessage(context)

  // 5. Reconstruire historique compacté
  const compactedMessages = [
    summaryMessage,
    ...importantMessages,
    recoveryMessage,
    ...recentMessages
  ]

  return compactedMessages
}
```

### 7.4 Messages Importants

**Critères** :
```typescript
function isImportantMessage(message: Message): boolean {
  if (message.type === 'user') {
    // User queries toujours importants
    return true
  }

  if (message.type === 'assistant') {
    for (const block of message.message.content) {
      // Tool use avec erreur
      if (block.type === 'tool_use' && hasError(block)) {
        return true
      }

      // Thinking avec décision importante
      if (block.type === 'thinking' && containsDecision(block)) {
        return true
      }
    }
  }

  return false
}
```

### 7.5 File Recovery

**Principe** : Re-charger les fichiers importants après compaction

```typescript
async function createFileRecoveryMessage(
  context: ToolUseContext
): Promise<Message> {
  // 1. Identifier fichiers récemment lus
  const recentFiles = Object.keys(context.readFileTimestamps)
    .sort((a, b) =>
      context.readFileTimestamps[b] - context.readFileTimestamps[a]
    )
    .slice(0, 10) // Top 10 fichiers

  // 2. Re-lire ces fichiers
  const fileContents = await Promise.all(
    recentFiles.map(path => readFile(path))
  )

  // 3. Créer message de récupération
  return {
    type: 'assistant',
    message: {
      role: 'assistant',
      content: [
        {
          type: 'text',
          text: `Context compacted. Key files reloaded:\n${recentFiles.map((file, i) => `\n### ${file}\n\`\`\`\n${fileContents[i]}\n\`\`\``).join('\n')}`
        }
      ]
    }
  }
}
```

### 7.6 Commande Manuelle

**Commande** : `/compact`

**Usage** :
```
> /compact

Compacting conversation...
✓ Preserved 5 recent messages
✓ Preserved 12 important messages
✓ Summarized 47 older messages
✓ Reloaded 8 key files

Context reduced: 184,000 → 92,000 tokens (50% reduction)
```

---

## 8. Gestion du Contexte

### 8.1 MessageContextManager

**Fichier** : `src/utils/messageContextManager.ts`

**Responsabilité** : Gestion intelligente du contexte de messages

### 8.2 Stratégies de Rétention

**4 stratégies disponibles** :

```typescript
export interface MessageRetentionStrategy {
  type:
    | 'preserve_recent'      // Préserver N messages récents
    | 'preserve_important'   // Préserver messages importants
    | 'smart_compression'    // Compression intelligente
    | 'auto_compact'         // Auto-compaction (à 92%)
  maxTokens: number
  preserveCount?: number
  importanceThreshold?: number
}
```

#### 8.2.1 Preserve Recent

```typescript
// Préserver les 20 derniers messages
const strategy: MessageRetentionStrategy = {
  type: 'preserve_recent',
  maxTokens: 100000,
  preserveCount: 20
}

const result = await manager.truncateMessages(messages, strategy)
// result.truncatedMessages contient les 20 derniers
```

#### 8.2.2 Preserve Important

```typescript
// Préserver messages importants + récents
const strategy: MessageRetentionStrategy = {
  type: 'preserve_important',
  maxTokens: 100000
}

const result = await manager.truncateMessages(messages, strategy)
// result.truncatedMessages contient :
//   - Messages avec erreurs
//   - Décisions utilisateur
//   - 5 derniers messages
```

#### 8.2.3 Smart Compression

```typescript
// Compression avec résumé AI
const strategy: MessageRetentionStrategy = {
  type: 'smart_compression',
  maxTokens: 100000
}

const result = await manager.truncateMessages(messages, strategy)
// result.truncatedMessages contient :
//   - Résumé de l'historique ancien
//   - Messages importants
//   - Messages récents
```

#### 8.2.4 Auto Compact

```typescript
// Auto-compaction (utilisé automatiquement à 92%)
const strategy: MessageRetentionStrategy = {
  type: 'auto_compact',
  maxTokens: modelInfo.contextLength * 0.92
}

const result = await manager.truncateMessages(messages, strategy)
// result.truncatedMessages contient :
//   - Conversation compactée
//   - File recovery
```

### 8.3 Résultat de Truncation

```typescript
export interface MessageTruncationResult {
  truncatedMessages: Message[]  // Messages résultants
  removedCount: number           // Nombre de messages supprimés
  preservedTokens: number        // Tokens préservés
  strategy: string               // Stratégie utilisée
  summary?: string               // Résumé de l'opération
}
```

**Exemple** :
```typescript
{
  truncatedMessages: [...],
  removedCount: 47,
  preservedTokens: 92000,
  strategy: "Auto-compact at 92% threshold",
  summary: "Removed 47 older messages, preserved 12 important messages, reloaded 8 key files"
}
```

---

## 9. API et Fonctions

### 9.1 Fonctions de Configuration

#### getGlobalConfig()

```typescript
export function getGlobalConfig(): GlobalConfig
```

**Usage** :
```typescript
const config = getGlobalConfig()
console.log(config.theme) // "dark"
console.log(config.verbose) // false
```

#### saveGlobalConfig()

```typescript
export function saveGlobalConfig(config: GlobalConfig): void
```

**Usage** :
```typescript
const config = getGlobalConfig()
config.verbose = true
saveGlobalConfig(config)
```

#### getCurrentProjectConfig()

```typescript
export function getCurrentProjectConfig(): ProjectConfig
```

**Usage** :
```typescript
const projectConfig = getCurrentProjectConfig()
console.log(projectConfig.allowedTools)
```

#### saveCurrentProjectConfig()

```typescript
export function saveCurrentProjectConfig(projectConfig: ProjectConfig): void
```

**Usage** :
```typescript
const projectConfig = getCurrentProjectConfig()
projectConfig.enableArchitectTool = true
saveCurrentProjectConfig(projectConfig)
```

### 9.2 Fonctions Utilitaires

#### checkHasTrustDialogAccepted()

```typescript
export function checkHasTrustDialogAccepted(): boolean
```

**Comportement** : Vérifie récursivement dans les répertoires parents

**Usage** :
```typescript
if (!checkHasTrustDialogAccepted()) {
  // Afficher trust dialog
}
```

#### getCustomApiKeyStatus()

```typescript
export function getCustomApiKeyStatus(
  truncatedApiKey: string
): 'approved' | 'rejected' | 'new'
```

**Usage** :
```typescript
const truncatedKey = apiKey.slice(-20)
const status = getCustomApiKeyStatus(truncatedKey)

if (status === 'rejected') {
  throw new Error('API key rejected by user')
}
```

#### getMcprcConfig()

```typescript
export const getMcprcConfig = memoize(
  (): Record<string, McpServerConfig> => { ... }
)
```

**Usage** :
```typescript
const mcprcServers = getMcprcConfig()
console.log(Object.keys(mcprcServers)) // ["github", "postgres"]
```

### 9.3 Fonctions de Migration

#### migrateModelProfilesRemoveId()

```typescript
function migrateModelProfilesRemoveId(config: GlobalConfig): GlobalConfig
```

**Responsabilité** : Migre anciennes configs avec `id` vers `modelName`

---

## Annexes

### A. Fichiers Clés

| Fichier | Lignes | Description |
|---------|--------|-------------|
| `src/utils/config.ts` | 939 | Système de configuration principal |
| `src/utils/autoCompactCore.ts` | ~250 | Auto-compaction à 92% |
| `src/utils/messageContextManager.ts` | ~200 | Gestion du contexte |
| `src/utils/fileRecoveryCore.ts` | ~100 | Récupération de fichiers |

### B. Variables d'Environnement

| Variable | Type | Description |
|----------|------|-------------|
| `ANTHROPIC_API_KEY` | string | Clé API Anthropic |
| `OPENAI_API_KEY` | string | Clé API OpenAI |
| `DISABLE_PROMPT_CACHING` | boolean | Désactiver cache prompts |
| `API_TIMEOUT_MS` | number | Timeout API (ms) |
| `MAX_THINKING_TOKENS` | number | Tokens max thinking |
| `KODE_MODEL` | string | Override modèle |
| `KODE_VERBOSE` | boolean | Override verbose |

### C. Providers Supportés

| Provider | Type | Description |
|----------|------|-------------|
| **anthropic** | `ProviderType` | Claude (Anthropic) |
| **openai** | `ProviderType` | GPT (OpenAI) |
| **mistral** | `ProviderType` | Mistral AI |
| **deepseek** | `ProviderType` | DeepSeek |
| **kimi** | `ProviderType` | Kimi (Moonshot) |
| **qwen** | `ProviderType` | Qwen (Alibaba) |
| **glm** | `ProviderType` | ChatGLM (Zhipu) |
| **minimax** | `ProviderType` | Minimax |
| **baidu-qianfan** | `ProviderType` | Baidu Qianfan |
| **siliconflow** | `ProviderType` | SiliconFlow |
| **bigdream** | `ProviderType` | BigDream |
| **opendev** | `ProviderType` | OpenDev |
| **xai** | `ProviderType` | xAI (Grok) |
| **groq** | `ProviderType` | Groq |
| **gemini** | `ProviderType` | Google Gemini |
| **ollama** | `ProviderType` | Ollama (local) |
| **azure** | `ProviderType` | Azure OpenAI |
| **custom** | `ProviderType` | Custom provider |
| **custom-openai** | `ProviderType` | Custom OpenAI-compatible |

### D. Exemples Complets

#### Configuration Minimale

```json
{
  "modelProfiles": [
    {
      "name": "Claude",
      "modelName": "claude-sonnet-4",
      "provider": "anthropic",
      "apiKey": "sk-ant-...",
      "maxTokens": 8192,
      "contextLength": 200000,
      "isActive": true,
      "createdAt": 1705401600000
    }
  ],
  "modelPointers": {
    "main": "claude-sonnet-4",
    "task": "claude-sonnet-4",
    "reasoning": "claude-sonnet-4",
    "quick": "claude-sonnet-4"
  }
}
```

#### Configuration Avancée

```json
{
  "theme": "dark",
  "verbose": true,
  "modelProfiles": [
    {
      "name": "Claude Sonnet 4",
      "modelName": "claude-sonnet-4",
      "provider": "anthropic",
      "apiKey": "sk-ant-...",
      "maxTokens": 8192,
      "contextLength": 200000,
      "isActive": true,
      "createdAt": 1705401600000
    },
    {
      "name": "GPT-5 Preview",
      "modelName": "gpt-5-preview",
      "provider": "openai",
      "apiKey": "sk-...",
      "maxTokens": 16384,
      "contextLength": 200000,
      "reasoningEffort": "high",
      "isGPT5": true,
      "isActive": true,
      "createdAt": 1705401700000
    },
    {
      "name": "DeepSeek V3",
      "modelName": "deepseek-chat",
      "provider": "deepseek",
      "apiKey": "sk-...",
      "maxTokens": 8192,
      "contextLength": 64000,
      "isActive": true,
      "createdAt": 1705401800000
    }
  ],
  "modelPointers": {
    "main": "claude-sonnet-4",
    "task": "gpt-5-preview",
    "reasoning": "claude-opus-4",
    "quick": "deepseek-chat"
  },
  "mcpServers": {
    "postgres": {
      "command": "docker",
      "args": ["run", "-i", "postgres-mcp-server"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    }
  },
  "projects": {
    "/home/user/my-project": {
      "allowedTools": [
        "Bash(git:*)",
        "Bash(npm:*)",
        "Bash(docker:*)",
        "WebSearch"
      ],
      "enableArchitectTool": true,
      "hasTrustDialogAccepted": true,
      "contextFiles": [
        "ARCHITECTURE.md",
        "API.md"
      ],
      "mcpServers": {
        "github": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-github"],
          "env": {
            "GITHUB_TOKEN": "${GITHUB_TOKEN}"
          }
        }
      }
    }
  }
}
```

---

**Navigation** :
- ← [Interface Utilisateur](./06-interface-utilisateur.md)
- → [Services et Intégrations](./08-services-integrations.md) *(à créer)*
- ↑ [Index](./INDEX.md)
- ⌂ [README](./README.md)

---

**Dernière mise à jour** : 16 janvier 2025
**Version de Kode** : 0.1.0
