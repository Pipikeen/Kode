# Système d'Outils - Architecture et Catalogue Complet

> Documentation technique du système d'outils extensible de Kode : 21+ outils natifs, permissions granulaires, validation, et intégration MCP

---

## Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Architecture des Outils](#2-architecture-des-outils)
3. [Interface Tool Standard](#3-interface-tool-standard)
4. [Système de Permissions](#4-système-de-permissions)
5. [Catalogue des Outils Natifs](#5-catalogue-des-outils-natifs)
6. [Intégration MCP](#6-intégration-mcp)
7. [Patterns d'Implémentation](#7-patterns-dimplémentation)
8. [Guide de Création d'Outils](#8-guide-de-création-doutils)

---

## 1. Vue d'Ensemble

### 1.1 Philosophie

Le système d'outils de Kode repose sur trois principes fondamentaux :

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  1. SÉCURITÉ PAR DÉFAUT                        │
│     └─ Permissions granulaires                 │
│     └─ Validation stricte des entrées          │
│     └─ Sandboxing des opérations              │
│                                                 │
│  2. EXTENSIBILITÉ                              │
│     └─ Interface Tool standardisée             │
│     └─ Support MCP pour outils externes        │
│     └─ Type-safe avec Zod schemas              │
│                                                 │
│  3. TRANSPARENCE                               │
│     └─ Rendu utilisateur clair                 │
│     └─ Résultats structurés pour l'assistant   │
│     └─ Logging détaillé                        │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 1.2 Statistiques du Système

**Outils Natifs** : 21+ outils intégrés
**Outils MCP** : Illimités (dynamiques)
**Categories** : 6 catégories principales
**Fichiers Sources** : 22 répertoires dans `src/tools/`

---

## 2. Architecture des Outils

### 2.1 Organisation du Code

```
src/tools/
├── Tool.ts                    # Interface de base (86 lignes)
├── tools.ts                   # Registry et exports (68 lignes)
├── permissions.ts             # Système de permissions (269 lignes)
│
├── TaskTool/                  # Orchestration d'agents
├── BashTool/                  # Exécution shell
├── FileReadTool/              # Lecture de fichiers
├── FileEditTool/              # Édition de fichiers
├── FileWriteTool/             # Création de fichiers
├── MultiEditTool/             # Édition multiple
├── GrepTool/                  # Recherche de contenu
├── GlobTool/                  # Recherche par pattern
├── LSTool/                    # Listage de répertoires
├── NotebookReadTool/          # Lecture de notebooks
├── NotebookEditTool/          # Édition de notebooks
├── MemoryReadTool/            # Mémoire persistante (lecture)
├── MemoryWriteTool/           # Mémoire persistante (écriture)
├── ThinkTool/                 # Réflexion structurée
├── TodoWriteTool/             # Gestion de tâches
├── WebSearchTool/             # Recherche web
├── URLFetcherTool/            # Récupération d'URLs
├── AskExpertModelTool/        # Requêtes experts
├── ArchitectTool/             # Architecture (optionnel)
└── MCPTool/                   # Intégration MCP
```

### 2.2 Flux d'Exécution d'un Outil

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  1. SÉLECTION                                         │
│     ┌──────────────────────────────────────┐          │
│     │ Agent sélectionne l'outil           │          │
│     │ via getTools() ou getReadOnlyTools() │          │
│     └──────────────────────────────────────┘          │
│                      ↓                                 │
│  2. VALIDATION                                        │
│     ┌──────────────────────────────────────┐          │
│     │ Validation du schema Zod            │          │
│     │ validateInput() pour logique custom  │          │
│     └──────────────────────────────────────┘          │
│                      ↓                                 │
│  3. PERMISSIONS                                       │
│     ┌──────────────────────────────────────┐          │
│     │ needsPermissions() vérifie besoin   │          │
│     │ hasPermissionsToUseTool() vérifie   │          │
│     │ Demande utilisateur si nécessaire    │          │
│     └──────────────────────────────────────┘          │
│                      ↓                                 │
│  4. EXÉCUTION                                         │
│     ┌──────────────────────────────────────┐          │
│     │ call() avec AsyncGenerator pattern  │          │
│     │ Yields progress et result            │          │
│     └──────────────────────────────────────┘          │
│                      ↓                                 │
│  5. RENDU                                             │
│     ┌──────────────────────────────────────┐          │
│     │ renderResultForAssistant() → Agent  │          │
│     │ renderToolResultMessage() → User     │          │
│     └──────────────────────────────────────┘          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 2.3 Catégories d'Outils

| Catégorie | Outils | Description |
|-----------|--------|-------------|
| **Système de fichiers** | FileRead, FileEdit, FileWrite, MultiEdit, Glob, LS | Manipulation de fichiers |
| **Recherche** | Grep, Glob, WebSearch | Recherche de code et web |
| **Exécution** | Bash, Task | Commandes shell et agents |
| **Notebooks** | NotebookRead, NotebookEdit | Jupyter notebooks |
| **Mémoire** | MemoryRead, MemoryWrite | Persistance contexte |
| **Utilitaires** | Think, TodoWrite, URLFetcher, AskExpertModel | Outils spécialisés |

---

## 3. Interface Tool Standard

### 3.1 Contrat TypeScript

```typescript
export interface Tool<
  TInput extends z.ZodObject<any> = z.ZodObject<any>,
  TOutput = any,
> {
  // MÉTADONNÉES
  name: string
  description?: () => Promise<string>
  userFacingName?: () => string

  // SCHÉMAS
  inputSchema: TInput
  inputJSONSchema?: Record<string, unknown>

  // PROMPT SYSTÈME
  prompt: (options?: { safeMode?: boolean }) => Promise<string>

  // CAPACITÉS
  isEnabled: () => Promise<boolean>
  isReadOnly: () => boolean
  isConcurrencySafe: () => boolean

  // PERMISSIONS
  needsPermissions: (input?: z.infer<TInput>) => boolean
  validateInput?: (
    input: z.infer<TInput>,
    context?: ToolUseContext,
  ) => Promise<ValidationResult>

  // RENDU
  renderResultForAssistant: (output: TOutput) => string | any[]
  renderToolUseMessage: (
    input: z.infer<TInput>,
    options: { verbose: boolean },
  ) => string
  renderToolUseRejectedMessage?: (...args: any[]) => React.ReactElement
  renderToolResultMessage?: (output: TOutput) => React.ReactElement

  // EXÉCUTION
  call: (
    input: z.infer<TInput>,
    context: ToolUseContext,
  ) => AsyncGenerator<
    | { type: 'result'; data: TOutput; resultForAssistant?: string }
    | { type: 'progress'; content: any; normalizedMessages?: any[]; tools?: any[] },
    void,
    unknown
  >
}
```

### 3.2 ToolUseContext

Contexte d'exécution fourni à chaque outil :

```typescript
export interface ToolUseContext {
  // Identifiants
  messageId: string | undefined
  agentId?: string

  // Contrôle
  abortController: AbortController
  safeMode?: boolean

  // État
  readFileTimestamps: { [filePath: string]: number }

  // Options
  options?: {
    commands?: any[]
    tools?: any[]
    verbose?: boolean
    slowAndCapableModel?: string
    safeMode?: boolean
    forkNumber?: number
    messageLogName?: string
    maxThinkingTokens?: any
    isKodingRequest?: boolean
    kodingContext?: string
    isCustomCommand?: boolean
  }

  // GPT-5 Responses API state management
  responseState?: {
    previousResponseId?: string
    conversationId?: string
  }
}
```

### 3.3 ValidationResult

Résultat de validation d'un outil :

```typescript
export interface ValidationResult {
  result: boolean          // Validation réussie ?
  message?: string         // Message d'erreur
  errorCode?: number       // Code d'erreur optionnel
  meta?: any              // Métadonnées supplémentaires
}
```

---

## 4. Système de Permissions

### 4.1 Niveaux de Permissions

Kode implémente un système de permissions à **deux niveaux** :

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  NIVEAU 1 : PERMISSIONS PROJET                 │
│  ────────────────────────────────────          │
│  Stockage    : .kode.json (disque)             │
│  Persistance : Oui (permanent)                 │
│  Scope       : Projet spécifique               │
│  Usage       : BashTool, outils génériques     │
│                                                 │
│  Exemple :                                      │
│  {                                              │
│    "allowedTools": [                            │
│      "Bash(git:*)",       // git prefixé       │
│      "Bash(npm:*)",       // npm prefixé       │
│      "Bash(ls -la)",      // commande exacte   │
│      "WebSearch"          // outil complet     │
│    ]                                            │
│  }                                              │
│                                                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  NIVEAU 2 : PERMISSIONS SESSION                │
│  ────────────────────────────────────          │
│  Stockage    : Mémoire (runtime)               │
│  Persistance : Non (session uniquement)        │
│  Scope       : Répertoire de démarrage         │
│  Usage       : FileEdit, FileWrite, NotebookEdit│
│                                                 │
│  Fonctionnement :                               │
│  • Permission demandée à la première édition   │
│  • Stockée en mémoire via                      │
│    grantWritePermissionForOriginalDir()        │
│  • Expire à la fin de la session               │
│  • Limitée au répertoire original (sandbox)    │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 4.2 Safe Mode

Le **Safe Mode** active les vérifications de permissions :

```typescript
// Si Safe Mode désactivé → Permissif (tout autorisé)
if (!context.options.safeMode) {
  return { result: true }
}

// Si Safe Mode activé → Vérifications strictes
if (tool.needsPermissions(input)) {
  // Demande permission utilisateur
}
```

**Activation Safe Mode** :
- Par défaut : `safeMode: false` (permissif)
- Configuration : `.kode.json` → `"safeMode": true`
- CLI : `kode --safe-mode`

### 4.3 Permissions BashTool

Le BashTool implémente le système de permissions le plus sophistiqué :

#### 4.3.1 Commandes Sûres

Certaines commandes sont toujours autorisées sans demande :

```typescript
const SAFE_COMMANDS = new Set([
  'git status',
  'git diff',
  'git log',
  'git branch',
  'pwd',
  'tree',
  'date',
  'which',
])
```

#### 4.3.2 Commandes Bannies

Certaines commandes sont strictement interdites :

```typescript
const BANNED_COMMANDS = [
  'rm',      // Suppression fichiers
  'dd',      // Opérations disque dangereuses
  'mkfs',    // Formatage
  'fdisk',   // Partitionnement
  // ... autres commandes dangereuses
]
```

#### 4.3.3 Détection d'Injection

Kode utilise un modèle LLM rapide pour détecter les injections de commandes :

```
┌────────────────────────────────────────────┐
│                                            │
│  Commande : npm install && rm -rf /        │
│              ↓                             │
│  getCommandSubcommandPrefix()              │
│              ↓                             │
│  Requête au modèle "quick" :               │
│  "Detect command injection in: ..."       │
│              ↓                             │
│  Résultat : {                              │
│    commandInjectionDetected: true,         │
│    commandPrefix: "npm",                   │
│    subcommandPrefixes: [...]               │
│  }                                         │
│              ↓                             │
│  Si injection → Demande permission exacte  │
│  Sinon → Autorise avec prefix              │
│                                            │
└────────────────────────────────────────────┘
```

#### 4.3.4 Système de Préfixes

Les permissions peuvent être accordées par préfixe :

```json
{
  "allowedTools": [
    "Bash(git:*)",      // Toutes commandes git
    "Bash(npm:*)",      // Toutes commandes npm
    "Bash(docker:*)",   // Toutes commandes docker
    "Bash(ls -la)"      // Commande exacte uniquement
  ]
}
```

Logique de vérification :

```typescript
// 1. Exact match
if (allowedTools.includes(`Bash(${command})`)) {
  return true
}

// 2. Prefix match
if (allowedTools.includes(`Bash(${prefix}:*)`)) {
  return true
}

// 3. Subcommands (pour commandes chaînées)
if (command.includes('&&') || command.includes(';')) {
  // Vérifie chaque sous-commande individuellement
  for (const subCmd of splitCommand(command)) {
    if (!hasPermission(subCmd)) {
      return false
    }
  }
}
```

### 4.4 Permissions Système de Fichiers

#### 4.4.1 Permissions en Lecture

```typescript
// src/utils/permissions/filesystem.ts

export function hasReadPermission(filePath: string): boolean {
  const normalizedPath = normalizeFilePath(filePath)
  const originalCwd = getOriginalCwd()

  // Toujours autorisé dans le répertoire de démarrage
  if (isInDirectory(normalizedPath, originalCwd)) {
    return true
  }

  // Refusé ailleurs (sandbox)
  return false
}
```

#### 4.4.2 Permissions en Écriture

```typescript
let writePermissionGrantedForOriginalDir = false

export function hasWritePermission(filePath: string): boolean {
  if (!writePermissionGrantedForOriginalDir) {
    return false
  }

  const normalizedPath = normalizeFilePath(filePath)
  const originalCwd = getOriginalCwd()

  // Vérifie que le fichier est dans le répertoire autorisé
  return isInDirectory(normalizedPath, originalCwd)
}

export function grantWritePermissionForOriginalDir(): void {
  writePermissionGrantedForOriginalDir = true
}
```

**Fonctionnement** :
1. Première demande d'édition → Demande permission utilisateur
2. Si accordée → `grantWritePermissionForOriginalDir()` appelée
3. Stockage en mémoire pour la session
4. Toutes éditions futures dans le répertoire autorisées
5. Expire à la fin de la session

### 4.5 Fonction hasPermissionsToUseTool

Fonction centrale de vérification des permissions :

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool,
  input,
  context,
): Promise<PermissionResult> => {
  // 1. Safe Mode désactivé → tout autorisé
  if (!context.options.safeMode) {
    return { result: true }
  }

  // 2. Vérifier si l'outil nécessite des permissions
  if (!tool.needsPermissions(input)) {
    return { result: true }
  }

  // 3. Charger les permissions du projet
  const projectConfig = getCurrentProjectConfig()
  const allowedTools = projectConfig.allowedTools ?? []

  // 4. Logique spécifique par outil
  switch (tool) {
    case BashTool:
      return await bashToolHasPermission(tool, command, context, allowedTools)

    case FileEditTool:
    case FileWriteTool:
    case NotebookEditTool:
      if (!tool.needsPermissions(input)) {
        return { result: true }
      }
      return {
        result: false,
        message: `Kode requested permissions to use ${tool.name}...`
      }

    default:
      const permissionKey = getPermissionKey(tool, input, null)
      if (allowedTools.includes(permissionKey)) {
        return { result: true }
      }
      return { result: false, message: '...' }
  }
}
```

### 4.6 Sauvegarde des Permissions

```typescript
export async function savePermission(
  tool: Tool,
  input: { [k: string]: unknown },
  prefix: string | null,
): Promise<void> {
  const key = getPermissionKey(tool, input, prefix)

  // Permissions session (mémoire uniquement)
  if (
    tool === FileEditTool ||
    tool === FileWriteTool ||
    tool === NotebookEditTool
  ) {
    grantWritePermissionForOriginalDir()
    return
  }

  // Permissions projet (disque)
  const projectConfig = getCurrentProjectConfig()
  if (projectConfig.allowedTools.includes(key)) {
    return // Déjà autorisé
  }

  projectConfig.allowedTools.push(key)
  projectConfig.allowedTools.sort()

  saveCurrentProjectConfig(projectConfig)
}
```

---

## 5. Catalogue des Outils Natifs

### 5.1 Outils Système de Fichiers

#### **FileReadTool** (`View`)

**Description** : Lecture de fichiers avec support multimédia

**Schéma d'entrée** :
```typescript
{
  file_path: string,        // Chemin absolu
  offset?: number,          // Ligne de départ
  limit?: number            // Nombre de lignes
}
```

**Capacités** :
- ✅ Lecture de fichiers texte
- ✅ Images (PNG, JPG, GIF, WebP) → Rendu visuel
- ✅ PDFs → Extraction texte + visuel
- ✅ Jupyter Notebooks (.ipynb)
- ✅ Détection encoding automatique
- ✅ Limite 2000 lignes par défaut
- ✅ Truncation > 2000 caractères par ligne

**Permissions** : Read-only, concurrence-safe

**Exemple** :
```typescript
{
  file_path: "/home/user/project/src/App.tsx",
  offset: 0,
  limit: 100
}
```

---

#### **FileEditTool** (`Edit`)

**Description** : Édition de fichiers par remplacement exact

**Schéma d'entrée** :
```typescript
{
  file_path: string,
  old_string: string,       // Texte à remplacer
  new_string: string        // Nouveau texte
}
```

**Mécanisme** :
1. Lecture du fichier existant
2. Recherche de `old_string` (doit être unique)
3. Remplacement par `new_string`
4. Génération d'un diff structuré
5. Écriture du fichier modifié

**Sécurité** :
- ⚠️ Échec si `old_string` non unique
- ⚠️ Préservation de l'encoding
- ⚠️ Préservation des line endings
- ⚠️ Permission session requise

**Permissions** : Write (session)

**Rendu** :
```
  ⎿ Updated file: src/App.tsx

    ┌─ Lines 45-52 ─────────────────────────
    │ - const theme = 'light'
    │ + const theme = 'dark'
    └────────────────────────────────────────
```

---

#### **FileWriteTool** (`Write`)

**Description** : Création de nouveaux fichiers

**Schéma d'entrée** :
```typescript
{
  file_path: string,
  content: string
}
```

**Comportement** :
- Crée le fichier s'il n'existe pas
- **Écrase** le fichier s'il existe déjà
- Crée les répertoires parents si nécessaires

**Permissions** : Write (session)

**Exemple** :
```typescript
{
  file_path: "/home/user/project/src/NewComponent.tsx",
  content: "import React from 'react';\n\nexport const NewComponent = () => {\n  return <div>Hello</div>;\n};"
}
```

---

#### **MultiEditTool**

**Description** : Édition multiple de fichiers en une seule opération

**Schéma d'entrée** :
```typescript
{
  edits: Array<{
    file_path: string,
    old_string: string,
    new_string: string
  }>
}
```

**Avantages** :
- Transaction atomique (tout ou rien)
- Plus rapide que plusieurs FileEdit
- Moins de demandes de permissions

**Permissions** : Write (session)

---

#### **GlobTool**

**Description** : Recherche de fichiers par pattern glob

**Schéma d'entrée** :
```typescript
{
  pattern: string,          // e.g., "**/*.ts"
  path?: string             // Répertoire de recherche
}
```

**Exemples de patterns** :
```
"**/*.tsx"              → Tous les fichiers .tsx
"src/**/*.test.ts"      → Tous les tests dans src/
"*.{js,ts}"             → JS et TS à la racine
"!node_modules/**"      → Exclure node_modules
```

**Performance** :
- Tri par date de modification
- Résultats limités pour grandes bases de code

**Permissions** : Read-only, concurrence-safe

---

#### **LSTool**

**Description** : Listage de répertoires (équivalent `ls`)

**Schéma d'entrée** :
```typescript
{
  path?: string,
  all?: boolean,            // Fichiers cachés
  long?: boolean            // Format détaillé
}
```

**Sortie** :
- Noms de fichiers
- Tailles
- Dates de modification
- Permissions (si `long: true`)

**Permissions** : Read-only, concurrence-safe

---

### 5.2 Outils de Recherche

#### **GrepTool** (`Search`)

**Description** : Recherche de contenu dans les fichiers (ripgrep)

**Schéma d'entrée** :
```typescript
{
  pattern: string,          // Regex pattern
  path?: string,            // Répertoire
  include?: string          // Pattern de fichiers (e.g., "*.ts")
}
```

**Moteur** : `ripgrep` (ultra-rapide)

**Capacités** :
- Recherche regex complète
- Filtrage par type de fichier
- Exclusion automatique .gitignore
- Limite 100 résultats par défaut

**Sortie** :
```typescript
{
  durationMs: number,
  numFiles: number,
  filenames: string[]
}
```

**Permissions** : Read-only, concurrence-safe

**Exemple** :
```typescript
{
  pattern: "function\\s+\\w+",
  path: "src/",
  include: "*.ts"
}
```

---

#### **WebSearchTool**

**Description** : Recherche web (intégration moteur de recherche)

**Schéma d'entrée** :
```typescript
{
  query: string
}
```

**Capacités** :
- Résultats de recherche web
- Snippets de contenu
- URLs des résultats

**Permissions** : Nécessite configuration API

---

### 5.3 Outils d'Exécution

#### **BashTool** (`Bash`)

**Description** : Exécution de commandes shell avec sécurité avancée

**Schéma d'entrée** :
```typescript
{
  command: string,
  timeout?: number          // Max 600000ms (10 min)
}
```

**Sécurité** :
- ✅ Commandes sûres pré-approuvées
- ⚠️ Commandes dangereuses bannies
- 🔍 Détection d'injection de commandes
- 🔒 Sandboxing au répertoire de démarrage
- ⏱️ Timeout configurable

**Shell Persistant** :
- Une seule instance de shell par session
- État préservé (variables, cd, etc.)
- Performances optimales

**Chaînage de Commandes** :
```bash
# Vérifié commande par commande
git add . && git commit -m "message" && git push

# Séquentiel sans dépendance
command1 ; command2 ; command3
```

**Sortie** :
```typescript
{
  stdout: string,
  stdoutLines: number,
  stderr: string,
  stderrLines: number,
  interrupted: boolean
}
```

**Permissions** : Projet (avec préfixes)

**Rendu** :
```
  ⎿ Bash: npm install

    added 243 packages in 12s
```

---

#### **TaskTool** (`Task`)

**Description** : Orchestration d'agents pour tâches complexes

**Schéma d'entrée** :
```typescript
{
  description: string,      // 3-5 mots
  prompt: string,           // Description complète
  model_name?: string,      // Modèle spécifique
  subagent_type?: string    // Type d'agent
}
```

**Agents Disponibles** :
- `general-purpose` : Tâches génériques (défaut)
- `code-reviewer` : Revue de code
- `test-writer` : Création de tests
- + Agents personnalisés (voir [04-systeme-agents.md](./04-systeme-agents.md))

**Fonctionnement** :
```
┌──────────────────────────────────────┐
│                                      │
│  TaskTool.call()                    │
│    ↓                                 │
│  Charge agent config                 │
│    ↓                                 │
│  Applique prompt + tools filter      │
│    ↓                                 │
│  Lance query() recursive             │
│    ↓                                 │
│  Streams progress                    │
│    ↓                                 │
│  Retourne résultats                  │
│                                      │
└──────────────────────────────────────┘
```

**Permissions** : Hérite de l'agent configuré

**Exemple** :
```typescript
{
  description: "Run tests",
  prompt: "Run the test suite and fix any failing tests",
  subagent_type: "test-writer"
}
```

---

### 5.4 Outils Notebooks

#### **NotebookReadTool**

**Description** : Lecture de Jupyter Notebooks (.ipynb)

**Schéma d'entrée** :
```typescript
{
  notebook_path: string
}
```

**Sortie** :
- Toutes les cellules (code + markdown)
- Outputs des cellules code
- Visualisations embarquées
- Métadonnées du notebook

**Permissions** : Read-only, concurrence-safe

---

#### **NotebookEditTool**

**Description** : Édition de cellules dans notebooks Jupyter

**Schéma d'entrée** :
```typescript
{
  notebook_path: string,
  cell_id: string,
  cell_type?: 'code' | 'markdown',
  edit_mode?: 'replace' | 'insert' | 'delete',
  new_source: string
}
```

**Modes d'édition** :
- `replace` : Remplace la cellule existante
- `insert` : Insère nouvelle cellule après cell_id
- `delete` : Supprime la cellule

**Permissions** : Write (session)

---

### 5.5 Outils de Mémoire

#### **MemoryReadTool**

**Description** : Lecture de mémoire persistante entre sessions

**Disponibilité** : **Anthropic uniquement** (Claude)

**Schéma d'entrée** :
```typescript
{
  key?: string              // Clé spécifique ou tout
}
```

**Usage** :
- Stockage préférences utilisateur
- Contexte projet
- Décisions architecturales
- Historique des tâches

**Permissions** : Read-only, concurrence-safe

---

#### **MemoryWriteTool**

**Description** : Écriture de mémoire persistante

**Disponibilité** : **Anthropic uniquement** (Claude)

**Schéma d'entrée** :
```typescript
{
  key: string,
  value: string
}
```

**Exemples d'usage** :
```typescript
// Préférence utilisateur
{ key: "preferred_test_framework", value: "vitest" }

// Architecture
{ key: "project_structure", value: "monorepo with pnpm workspaces" }

// Conventions
{ key: "commit_message_format", value: "conventional commits" }
```

**Permissions** : Nécessite approbation

---

### 5.6 Outils Utilitaires

#### **ThinkTool**

**Description** : Réflexion structurée avant action

**Schéma d'entrée** :
```typescript
{
  thought: string
}
```

**Usage** :
- Planification de tâches complexes
- Analyse de problèmes
- Debugging mental

**Rendu** : Masqué à l'utilisateur (interne à l'agent)

**Permissions** : Aucune

---

#### **TodoWriteTool**

**Description** : Gestion de liste de tâches

**Schéma d'entrée** :
```typescript
{
  todos: Array<{
    content: string,
    status: 'pending' | 'in_progress' | 'completed',
    activeForm: string
  }>
}
```

**Affichage utilisateur** :
```
┌─ Tasks ────────────────────────────┐
│ ✓ Install dependencies             │
│ ⧗ Run tests                         │
│ ○ Fix failing tests                 │
│ ○ Commit changes                    │
└────────────────────────────────────┘
```

**Permissions** : Aucune

---

#### **URLFetcherTool**

**Description** : Récupération de contenu web

**Schéma d'entrée** :
```typescript
{
  url: string
}
```

**Capacités** :
- HTML → Markdown conversion
- Extraction de contenu principal
- Support JavaScript (limité)

**Permissions** : Nécessite configuration réseau

---

#### **AskExpertModelTool**

**Description** : Requête à un modèle expert spécialisé

**Schéma d'entrée** :
```typescript
{
  query: string,
  expert_type?: string      // Type d'expertise
}
```

**Usage** :
- Questions complexes nécessitant expertise
- Délégation à modèle plus capable
- Vérification croisée

**Permissions** : Nécessite configuration modèle

---

#### **ArchitectTool**

**Description** : Outil d'architecture (expérimental, opt-in)

**Activation** :
```json
// .kode.json
{
  "enableArchitect": true
}
```

**Schéma** : (Varie selon implémentation)

**Permissions** : Configurable

---

### 5.7 Récapitulatif des Outils

| Outil | Nom | Read-Only | Concurrence-Safe | Permissions |
|-------|-----|-----------|------------------|-------------|
| FileReadTool | `View` | ✅ | ✅ | Aucune (sandbox) |
| FileEditTool | `Edit` | ❌ | ❌ | Session |
| FileWriteTool | `Write` | ❌ | ❌ | Session |
| MultiEditTool | `MultiEdit` | ❌ | ❌ | Session |
| GlobTool | `Glob` | ✅ | ✅ | Aucune (sandbox) |
| GrepTool | `Search` | ✅ | ✅ | Aucune (sandbox) |
| LSTool | `LS` | ✅ | ✅ | Aucune (sandbox) |
| BashTool | `Bash` | ❌ | ❌ | Projet + préfixes |
| TaskTool | `Task` | Dépend | Dépend | Héritées |
| NotebookReadTool | `NotebookRead` | ✅ | ✅ | Aucune (sandbox) |
| NotebookEditTool | `NotebookEdit` | ❌ | ❌ | Session |
| MemoryReadTool | `MemoryRead` | ✅ | ✅ | Aucune |
| MemoryWriteTool | `MemoryWrite` | ❌ | ✅ | Demande |
| ThinkTool | `Think` | ✅ | ✅ | Aucune |
| TodoWriteTool | `TodoWrite` | ❌ | ✅ | Aucune |
| WebSearchTool | `WebSearch` | ✅ | ✅ | API |
| URLFetcherTool | `URLFetcher` | ✅ | ✅ | Réseau |
| AskExpertModelTool | `AskExpert` | ✅ | ✅ | API |
| ArchitectTool | `Architect` | Varie | Varie | Config |

---

## 6. Intégration MCP

### 6.1 Model Context Protocol

**MCP** permet d'étendre Kode avec des outils externes dynamiques.

**Architecture** :
```
┌────────────────────────────────────────────────┐
│                                                │
│  Kode                                         │
│    │                                           │
│    ├─ src/services/mcpClient.ts               │
│    │   └─ Client MCP                          │
│    │       │                                   │
│    │       ├─ StdioClientTransport            │
│    │       └─ SSEClientTransport              │
│    │                                           │
│    └─ src/tools/MCPTool/                      │
│        └─ Wrapper pour outils MCP             │
│                                                │
│  ↕ MCP Protocol                                │
│                                                │
│  Serveurs MCP                                  │
│    ├─ Serveur Filesystem                      │
│    ├─ Serveur Database                        │
│    ├─ Serveur API                             │
│    └─ ... (illimités)                         │
│                                                │
└────────────────────────────────────────────────┘
```

### 6.2 Configuration MCP

**3 niveaux de configuration** (similaire aux agents) :

1. **Projet** : `.kode.json`
2. **Global** : `~/.kode.json`
3. **MCPRC** : `.mcprc` (fichier spécial)

**Exemple de configuration** :

```json
// .kode.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/directory"],
      "env": {
        "NODE_ENV": "production"
      }
    },
    "postgres": {
      "command": "docker",
      "args": ["run", "-i", "postgres-mcp-server"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    }
  }
}
```

### 6.3 Types de Transport

**1. Stdio Transport** (Standard)
```typescript
const transport = new StdioClientTransport({
  command: 'npx',
  args: ['-y', '@modelcontextprotocol/server-filesystem', '/path'],
  env: { NODE_ENV: 'production' }
})
```

**2. SSE Transport** (Server-Sent Events)
```typescript
const transport = new SSEClientTransport({
  url: 'http://localhost:3000/mcp',
  headers: { 'Authorization': 'Bearer token' }
})
```

### 6.4 Gestion des Serveurs MCP

**Commandes CLI** :
```bash
# Ajouter un serveur
kode mcp add <name> --command <cmd> --args <args> --scope project

# Lister les serveurs
kode mcp list

# Supprimer un serveur
kode mcp remove <name> --scope project

# Tester un serveur
kode mcp test <name>
```

**API Programmatique** :
```typescript
import { addMcpServer, removeMcpServer, getMCPTools } from '@services/mcpClient'

// Ajouter un serveur
addMcpServer('my-server', {
  command: 'node',
  args: ['server.js'],
  env: { PORT: '3000' }
}, 'project')

// Charger les outils MCP
const mcpTools = await getMCPTools()
// → Tableau d'outils Tool compatibles
```

### 6.5 Cycle de Vie des Serveurs MCP

```
┌────────────────────────────────────────────┐
│                                            │
│  1. DÉMARRAGE                              │
│     ┌────────────────────────────┐         │
│     │ createClient(serverConfig) │         │
│     │   → Spawn process          │         │
│     │   → Initialize transport   │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  2. CONNEXION                              │
│     ┌────────────────────────────┐         │
│     │ client.connect()           │         │
│     │   → Handshake              │         │
│     │   → Version check          │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  3. DÉCOUVERTE                             │
│     ┌────────────────────────────┐         │
│     │ client.listTools()         │         │
│     │   → Schémas des outils     │         │
│     │   → Conversion Tool        │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  4. UTILISATION                            │
│     ┌────────────────────────────┐         │
│     │ client.callTool(name, args)│         │
│     │   → Exécution              │         │
│     │   → Résultats              │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  5. ARRÊT                                  │
│     ┌────────────────────────────┐         │
│     │ client.close()             │         │
│     │   → Cleanup                │         │
│     │   → Kill process           │         │
│     └────────────────────────────┘         │
│                                            │
└────────────────────────────────────────────┘
```

### 6.6 Conversion MCP → Tool

Kode convertit automatiquement les outils MCP en outils `Tool` :

```typescript
// Outil MCP (schema MCP)
{
  name: "read_file",
  description: "Read a file from the filesystem",
  inputSchema: {
    type: "object",
    properties: {
      path: { type: "string" }
    }
  }
}

// Converti en Tool Kode
{
  name: "mcp__filesystem__read_file",
  description: async () => "Read a file from the filesystem",
  inputSchema: z.object({ path: z.string() }),
  prompt: async () => "...",
  isEnabled: async () => true,
  isReadOnly: () => false,
  isConcurrencySafe: () => false,
  needsPermissions: () => true,
  call: async function* (input, context) {
    const result = await mcpClient.callTool("read_file", input)
    yield { type: 'result', data: result }
  },
  // ... autres méthodes
}
```

**Préfixe** : `mcp__<serveur>__<outil>`
- Évite les collisions de noms
- Identifie la source de l'outil
- Permet filtrage dans agents

### 6.7 Exemple Complet

**Fichier** : `.kode.json`
```json
{
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
```

**Utilisation** :
```typescript
// L'agent Kode peut maintenant utiliser :
// - mcp__github__search_repositories
// - mcp__github__get_file_contents
// - mcp__github__create_issue
// - ... tous les outils du serveur GitHub MCP
```

**Filtrage dans agents** :
```markdown
---
name: github-manager
tools:
  - mcp__github__*
  - FileRead
  - FileWrite
---

You are a GitHub management agent with access to GitHub operations.
```

---

## 7. Patterns d'Implémentation

### 7.1 AsyncGenerator Pattern

Tous les outils utilisent `AsyncGenerator` pour permettre le streaming :

```typescript
async *call(input, context): AsyncGenerator<
  | { type: 'result'; data: TOutput; resultForAssistant?: string }
  | { type: 'progress'; content: any; normalizedMessages?: any[] },
  void,
  unknown
> {
  // 1. Yield progress
  yield {
    type: 'progress',
    content: 'Starting operation...'
  }

  // 2. Perform operation
  const result = await performOperation(input)

  // 3. Yield progress
  yield {
    type: 'progress',
    content: 'Operation 50% complete...'
  }

  // 4. Final result
  yield {
    type: 'result',
    data: result,
    resultForAssistant: formatForAssistant(result)
  }
}
```

**Avantages** :
- ✅ Feedback temps réel à l'utilisateur
- ✅ Possibilité d'annulation (`abortController`)
- ✅ Streaming de résultats partiels
- ✅ Gestion d'erreurs progressive

### 7.2 Validation avec Zod

Tous les schémas d'entrée utilisent Zod pour validation type-safe :

```typescript
const inputSchema = z.strictObject({
  file_path: z.string().describe('The absolute path to the file'),
  content: z.string().describe('The content to write'),
  encoding: z.enum(['utf8', 'utf16le', 'latin1']).optional()
})

// TypeScript infère automatiquement le type
type Input = z.infer<typeof inputSchema>
// → { file_path: string; content: string; encoding?: 'utf8' | 'utf16le' | 'latin1' }
```

**Avantages** :
- Type safety compile-time
- Validation runtime automatique
- Documentation via `.describe()`
- Génération de JSON Schema

### 7.3 Memoization

Les outils coûteux sont mémoïsés :

```typescript
import { memoize } from 'lodash-es'

export const getTools = memoize(
  async (enableArchitect?: boolean): Promise<Tool[]> => {
    const tools = [...getAllTools(), ...(await getMCPTools())]

    if (enableArchitect) {
      tools.push(ArchitectTool)
    }

    const isEnabled = await Promise.all(tools.map(tool => tool.isEnabled()))
    return tools.filter((_, i) => isEnabled[i])
  }
)
```

**Bénéfices** :
- ⚡ Pas de rechargement à chaque utilisation
- 💾 Cache en mémoire
- 🔄 Invalidation automatique (lodash)

### 7.4 React Components pour Rendu

Les outils utilisent React (Ink) pour rendu terminal :

```typescript
renderToolResultMessage(output) {
  return (
    <Box flexDirection="column">
      <Text>
        {'  '}⎿ <Text color="green">Updated file:</Text>{' '}
        <Text bold>{output.filePath}</Text>
      </Text>
      <Box paddingLeft={5}>
        <StructuredDiff patch={output.structuredPatch} />
      </Box>
    </Box>
  )
}
```

**Composants utilisés** :
- `Box` : Layout flexbox
- `Text` : Texte avec couleurs/styles
- `HighlightedCode` : Coloration syntaxique
- `StructuredDiff` : Affichage de diffs

### 7.5 Error Handling

Pattern de gestion d'erreurs standard :

```typescript
async *call(input, context) {
  try {
    // Vérification abort
    if (context.abortController.signal.aborted) {
      throw new AbortError()
    }

    // Opération
    const result = await operation(input)

    yield {
      type: 'result',
      data: result
    }
  } catch (error) {
    if (error instanceof AbortError) {
      throw error // Re-throw abort errors
    }

    // Log error
    logError(`Tool error: ${error}`)

    // Return error result
    yield {
      type: 'result',
      data: {
        error: true,
        message: error.message
      }
    }
  }
}
```

---

## 8. Guide de Création d'Outils

### 8.1 Structure de Fichiers

```
src/tools/MyNewTool/
├── MyNewTool.tsx          # Implémentation principale
├── prompt.ts              # Prompts système
├── utils.ts               # Utilitaires (optionnel)
└── components/            # Composants React (optionnel)
    └── MyToolResult.tsx
```

### 8.2 Template de Base

```typescript
// src/tools/MyNewTool/MyNewTool.tsx

import { z } from 'zod'
import { Tool, ToolUseContext, ValidationResult } from '@tool'
import { DESCRIPTION, PROMPT } from './prompt'

// 1. Définir le schéma d'entrée
const inputSchema = z.strictObject({
  required_param: z.string().describe('Description'),
  optional_param: z.number().optional().describe('Description')
})

type Input = z.infer<typeof inputSchema>
type Output = {
  // Define your output type
  result: string
}

// 2. Implémenter l'outil
export const MyNewTool: Tool<typeof inputSchema, Output> = {
  // Métadonnées
  name: 'MyNewTool',

  async description() {
    return DESCRIPTION
  },

  async prompt({ safeMode }) {
    return PROMPT
  },

  userFacingName() {
    return 'My Tool'
  },

  // Schéma
  inputSchema,

  // Capacités
  async isEnabled() {
    return true // Ou condition spécifique
  },

  isReadOnly() {
    return false // true si lecture seule
  },

  isConcurrencySafe() {
    return false // true si safe pour concurrence
  },

  // Permissions
  needsPermissions(input) {
    return true // ou logique conditionnelle
  },

  async validateInput(input, context): Promise<ValidationResult> {
    // Validation personnalisée
    if (!input.required_param) {
      return {
        result: false,
        message: 'required_param is required'
      }
    }

    return { result: true }
  },

  // Rendu
  renderToolUseMessage(input, { verbose }) {
    return `required_param: ${input.required_param}`
  },

  renderResultForAssistant(output) {
    return `Result: ${output.result}`
  },

  renderToolResultMessage(output) {
    return (
      <Box>
        <Text>⎿ Result: {output.result}</Text>
      </Box>
    )
  },

  // Exécution
  async *call(input, context): AsyncGenerator<...> {
    // Progress
    yield {
      type: 'progress',
      content: 'Starting...'
    }

    // Opération
    const result = await performOperation(input)

    // Result
    yield {
      type: 'result',
      data: { result },
      resultForAssistant: `Completed: ${result}`
    }
  }
}
```

### 8.3 Prompts

```typescript
// src/tools/MyNewTool/prompt.ts

export const DESCRIPTION = 'Short description for tool selection'

export const PROMPT = `
# MyNewTool

## Description
Detailed description of what this tool does.

## Usage
When to use this tool and how.

## Parameters
- required_param: Description
- optional_param: Description

## Examples
\`\`\`typescript
{
  "required_param": "example value",
  "optional_param": 42
}
\`\`\`

## Notes
Important notes and caveats.
`
```

### 8.4 Enregistrement

```typescript
// src/tools.ts

import { MyNewTool } from './tools/MyNewTool/MyNewTool'

export const getAllTools = (): Tool[] => {
  return [
    // ... existing tools
    MyNewTool as unknown as Tool,
  ]
}
```

### 8.5 Tests

```typescript
// src/tools/MyNewTool/MyNewTool.test.ts

import { describe, test, expect } from 'bun:test'
import { MyNewTool } from './MyNewTool'

describe('MyNewTool', () => {
  test('validates input correctly', async () => {
    const result = await MyNewTool.validateInput(
      { required_param: 'test' },
      mockContext
    )

    expect(result.result).toBe(true)
  })

  test('executes successfully', async () => {
    const generator = MyNewTool.call(
      { required_param: 'test' },
      mockContext
    )

    const results = []
    for await (const item of generator) {
      results.push(item)
    }

    expect(results[results.length - 1].type).toBe('result')
  })
})
```

### 8.6 Bonnes Pratiques

**DO** ✅ :
- Utiliser `z.strictObject` pour validation stricte
- Implémenter `validateInput` pour logique complexe
- Fournir descriptions claires avec `.describe()`
- Gérer les erreurs gracieusement
- Vérifier `abortController.signal.aborted`
- Utiliser `logError` pour logging
- Rendu React pour messages utilisateur
- Tests unitaires complets

**DON'T** ❌ :
- Bloquer le thread principal
- Ignorer les erreurs
- Permissions trop larges par défaut
- Oublier la gestion d'annulation
- Rendu inconsistant avec autres outils
- Opérations non-idempotentes sans validation

---

## Annexes

### A. Fichiers Clés

| Fichier | Lignes | Description |
|---------|--------|-------------|
| `src/Tool.ts` | 86 | Interface de base |
| `src/tools.ts` | 68 | Registry des outils |
| `src/permissions.ts` | 269 | Système de permissions |
| `src/utils/permissions/filesystem.ts` | ~150 | Permissions fichiers |
| `src/services/mcpClient.ts` | ~800 | Client MCP |
| `src/tools/TaskTool/TaskTool.tsx` | ~600 | Orchestration agents |
| `src/tools/BashTool/BashTool.tsx` | ~400 | Exécution shell |

### B. Types de Permissions

| Type | Stockage | Scope | Expiration | Outils |
|------|----------|-------|------------|--------|
| **Projet** | `.kode.json` | Projet | Permanent | Bash, WebSearch, etc. |
| **Session** | Mémoire | Répertoire | Session | FileEdit, FileWrite, NotebookEdit |
| **Aucune** | - | - | - | FileRead, Grep, Think, etc. |

### C. Commandes Dangereuses

```typescript
const BANNED_COMMANDS = [
  'rm',        // Suppression
  'dd',        // Écriture disque
  'mkfs',      // Formatage
  'fdisk',     // Partitionnement
  'chmod',     // Permissions (peut casser système)
  'chown',     // Propriétaire
  'kill',      // Processus
  'killall',   // Processus multiples
  'reboot',    // Redémarrage
  'shutdown',  // Extinction
  'halt',      // Arrêt
  // ... autres
]
```

### D. Ressources

**Documentation Externe** :
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Zod Documentation](https://zod.dev/)
- [Ink (React for CLI)](https://github.com/vadimdemedes/ink)
- [ripgrep](https://github.com/BurntSushi/ripgrep)

**Code Source** :
- Interface Tool : [`src/Tool.ts`](../src/Tool.ts)
- Permissions : [`src/permissions.ts`](../src/permissions.ts)
- MCP Client : [`src/services/mcpClient.ts`](../src/services/mcpClient.ts)

---

**Navigation** :
- ← [Système d'Agents](./04-systeme-agents.md)
- → [Interface Utilisateur](./06-interface-utilisateur.md) *(à créer)*
- ↑ [Index](./INDEX.md)
- ⌂ [README](./README.md)

---

**Dernière mise à jour** : 16 janvier 2025
**Version de Kode** : 0.1.0
