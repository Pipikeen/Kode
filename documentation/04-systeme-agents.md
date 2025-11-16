# Système d'Agents Dynamiques de Kode

> Architecture sophistiquée avec 5 niveaux de priorité et hot reload

[← Retour à l'index](./README.md) | [← Modèles](./03-systeme-modeles.md) | [Outils →](./05-systeme-outils.md)

---

## Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Architecture du Chargeur d'Agents](#2-architecture-du-chargeur-dagents)
3. [Système de Priorité à 5 Niveaux](#3-système-de-priorité-à-5-niveaux)
4. [Format des Fichiers Agents](#4-format-des-fichiers-agents)
5. [Compatibilité Claude Code](#5-compatibilité-claude-code)
6. [Système de Cache et Mémoïsation](#6-système-de-cache-et-mémoïsation)
7. [Hot Reload et Watchers](#7-hot-reload-et-watchers)
8. [Sélection et Exécution](#8-sélection-et-exécution)
9. [Guide de Création d'Agents](#9-guide-de-création-dagents)
10. [Exemples Pratiques](#10-exemples-pratiques)

---

## 1. Vue d'Ensemble

### Innovation Clé

Le système d'agents de Kode est un **système sophistiqué de chargement dynamique** avec une hiérarchie à 5 niveaux de priorité, permettant la création d'agents spécialisés réutilisables.

**Caractéristiques** :
- ✅ **5 niveaux de priorité** pour résolution d'agents
- ✅ **Hot reload** automatique des fichiers agents
- ✅ **Compatibilité Claude Code** (`.claude/agents/`)
- ✅ **Format simple** : YAML frontmatter + Markdown
- ✅ **Cache intelligent** avec mémoïsation
- ✅ **Filtrage d'outils** par agent
- ✅ **Surcharge de modèle** par agent

### Architecture Globale

```
┌─────────────────────────────────────────────────────┐
│                    Kode CLI                          │
├─────────────────────────────────────────────────────┤
│                   TaskTool                           │
│  (Délégation de tâches à des agents)                │
├─────────────────────────────────────────────────────┤
│                Agent Loader                          │
│  (Chargement dynamique avec priorités)              │
├─────────────────────────────────────────────────────┤
│              5 Niveaux de Priorité                   │
│  5. ./.kode/agents/      ← Priorité MAXIMALE        │
│  4. ./.claude/agents/    ← Projet Claude Code       │
│  3. ~/.kode/agents/      ← Utilisateur Kode         │
│  2. ~/.claude/agents/    ← Utilisateur Claude Code  │
│  1. Built-in             ← Code intégré             │
└─────────────────────────────────────────────────────┘
```

---

## 2. Architecture du Chargeur d'Agents

### Fichier Principal

**Localisation** : `src/utils/agentLoader.ts`

### Structure d'un AgentConfig

```typescript
interface AgentConfig {
  agentType: string          // Identifiant unique (ex: "general-purpose")
  whenToUse: string          // Description de quand l'utiliser
  tools: string[] | '*'      // Permissions d'outils ('*' = tous)
  systemPrompt: string       // Prompt système de l'agent
  location: 'built-in' | 'user' | 'project'  // Niveau
  color?: string            // Couleur UI optionnelle
  model_name?: string       // Surcharge de modèle optionnelle
}
```

### Fonctions Principales

```typescript
// Scanner un répertoire pour fichiers .md
async function scanAgentDirectory(
  dirPath: string,
  location: 'user' | 'project'
): Promise<AgentConfig[]>

// Charger tous les agents avec priorité
async function loadAllAgents(): Promise<{
  activeAgents: AgentConfig[]
  allAgents: AgentConfig[]
}>

// Obtenir agents actifs (mémorisé)
export const getActiveAgents = memoize(
  async (): Promise<AgentConfig[]> => {
    const { activeAgents } = await loadAllAgents()
    return activeAgents
  }
)

// Récupérer un agent spécifique
export const getAgentByType = memoize(
  async (agentType: string): Promise<AgentConfig | undefined> => {
    const agents = await getActiveAgents()
    return agents.find(agent => agent.agentType === agentType)
  }
)

// Surveiller changements (hot reload)
async function startAgentWatcher(callback?: () => void)

// Invalider le cache
function clearAgentCache()
```

---

## 3. Système de Priorité à 5 Niveaux

### Hiérarchie Complète

Le système applique une **résolution par écrasement progressif** :

```
┌──────────────────────────────────────────────────────┐
│ Niveau 1 (Priorité minimale)                         │
│   Built-in agents (code intégré)                     │
│   Exemple : BUILTIN_GENERAL_PURPOSE                  │
└──────────────────────────────────────────────────────┘
                     ↓ écrasé par
┌──────────────────────────────────────────────────────┐
│ Niveau 2                                             │
│   ~/.claude/agents/*.md                              │
│   Agents utilisateur (compatibilité Claude Code)     │
│   Partagés entre tous les projets                    │
└──────────────────────────────────────────────────────┘
                     ↓ écrasé par
┌──────────────────────────────────────────────────────┐
│ Niveau 3                                             │
│   ~/.kode/agents/*.md                                │
│   Agents utilisateur (natif Kode)                    │
│   Partagés entre tous les projets                    │
└──────────────────────────────────────────────────────┘
                     ↓ écrasé par
┌──────────────────────────────────────────────────────┐
│ Niveau 4                                             │
│   ./.claude/agents/*.md                              │
│   Agents projet (compatibilité Claude Code)          │
│   Spécifiques au projet courant                      │
└──────────────────────────────────────────────────────┘
                     ↓ écrasé par
┌──────────────────────────────────────────────────────┐
│ Niveau 5 (Priorité MAXIMALE)                         │
│   ./.kode/agents/*.md                                │
│   Agents projet (natif Kode)                         │
│   Spécifiques au projet courant                      │
└──────────────────────────────────────────────────────┘
```

### Mécanisme de Résolution

```typescript
async function loadAllAgents() {
  // Charger tous les niveaux en parallèle
  const [builtinAgents, userClaudeAgents, userKodeAgents, 
         projectClaudeAgents, projectKodeAgents] = await Promise.all([
    getBuiltinAgents(),
    scanAgentDirectory(join(homedir(), '.claude', 'agents'), 'user'),
    scanAgentDirectory(join(homedir(), '.kode', 'agents'), 'user'),
    scanAgentDirectory(join(getCwd(), '.claude', 'agents'), 'project'),
    scanAgentDirectory(join(getCwd(), '.kode', 'agents'), 'project')
  ])

  // Application de la priorité (Map écrase les clés existantes)
  const agentMap = new Map<string, AgentConfig>()

  // Ajout par ordre de priorité croissante
  for (const agent of builtinAgents) {
    agentMap.set(agent.agentType, agent)
  }
  for (const agent of userClaudeAgents) {
    agentMap.set(agent.agentType, agent)  // Écrase built-in
  }
  for (const agent of userKodeAgents) {
    agentMap.set(agent.agentType, agent)  // Écrase .claude user
  }
  for (const agent of projectClaudeAgents) {
    agentMap.set(agent.agentType, agent)  // Écrase .kode user
  }
  for (const agent of projectKodeAgents) {
    agentMap.set(agent.agentType, agent)  // Écrase tout (PRIORITÉ MAX)
  }

  return {
    activeAgents: Array.from(agentMap.values()),
    allAgents: [...all agents from all levels]
  }
}
```

**Exemple** :

Si vous avez un agent `code-writer` dans :
- Built-in
- `~/.claude/agents/code-writer.md`
- `./.kode/agents/code-writer.md`

→ C'est la version `./.kode/agents/code-writer.md` qui sera utilisée.

---

## 4. Format des Fichiers Agents

### Structure YAML Frontmatter + Markdown

Chaque agent est un fichier `.md` avec :

```markdown
---
name: agent-identifier          # REQUIS : Identifiant kebab-case
description: "Description"      # REQUIS : Quand utiliser l'agent
tools: ["Tool1", "Tool2"]       # OPTIONNEL : Liste d'outils ou "*"
model_name: main                # OPTIONNEL : Modèle spécifique
color: blue                     # OPTIONNEL : Couleur UI
---

Votre système prompt détaillé ici.
Peut contenir plusieurs paragraphes.

## Sections avec Markdown

- Instructions
- Exemples
- Directives
```

### Champs Disponibles

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `name` | string | ✅ | Identifiant unique (kebab-case) |
| `description` | string | ✅ | Quand utiliser l'agent |
| `tools` | string[] ou "*" | ❌ | Outils autorisés (défaut: "*") |
| `model_name` | string | ❌ | Surcharge de modèle |
| `color` | string | ❌ | Couleur UI (cyan, blue, green, etc.) |

**Note** : Le champ `model` est déprécié, utilisez `model_name`.

### Exemples de Fichiers

#### Agent Spécialisé en Recherche

```markdown
---
name: search-specialist
description: "Specialized in finding files and code patterns quickly"
tools: ["Grep", "Glob", "Read", "LS"]
color: green
---

You are a search specialist optimized for quickly finding files and code patterns.

Your approach:
1. Use Grep for content searches
2. Use Glob for pattern matching
3. Read files when needed
4. Provide precise results

Be efficient and fast.
```

#### Agent Écrivain de Code

```markdown
---
name: code-writer
description: "Specialized in writing and modifying code"
tools: ["Read", "Write", "Edit", "MultiEdit", "Bash"]
color: blue
model_name: main
---

You are a code writing specialist focused on implementing features.

Guidelines:
- Always read existing code first
- Follow project conventions
- Write clean, maintainable code
- Add comments where needed
- Test your changes

Workflow:
1. Understand the requirement
2. Read relevant files
3. Plan the implementation
4. Write the code
5. Verify with tests if available
```

#### Agent Accès Complet

```markdown
---
name: test-agent
description: "Test agent for validation"
tools: '*'
model_name: claude-3-5-sonnet-20241022
---

You are a test agent with full tool access.
Use all available tools as needed.
```

---

## 5. Compatibilité Claude Code

### Double Système de Répertoires

Kode maintient une **compatibilité bidirectionnelle** avec Claude Code :

**Répertoires Scannés** :
- `~/.claude/agents/` - Agents utilisateur Claude Code
- `~/.kode/agents/` - Agents utilisateur Kode
- `./.claude/agents/` - Agents projet Claude Code
- `./.kode/agents/` - Agents projet Kode

**Avantages** :

1. **Migration progressive** : Passez de Claude Code à Kode sans tout reconfigurer
2. **Partage d'équipe** : Agents `.claude/agents/` fonctionnent dans les deux outils
3. **Spécialisation** : Utilisez `.kode/agents/` pour fonctionnalités Kode-spécifiques

### Format Partagé

Le format YAML frontmatter + Markdown est identique, permettant :
- ✅ Copie directe d'agents entre systèmes
- ✅ Partage via git de configurations `.claude/agents/`
- ✅ Utilisation des mêmes packs d'agents

**Exemple de migration** :

```bash
# Copier un agent Claude Code vers Kode
cp ~/.claude/agents/my-agent.md ~/.kode/agents/my-agent.md

# Ou partager dans un projet
mkdir -p ./.claude/agents
# Agents disponibles pour Claude Code ET Kode
```

---

## 6. Système de Cache et Mémoïsation

### Mémoïsation avec Lodash

```typescript
import { memoize } from 'lodash-es'

export const getActiveAgents = memoize(
  async (): Promise<AgentConfig[]> => {
    const { activeAgents } = await loadAllAgents()
    return activeAgents
  }
)

export const getAgentByType = memoize(
  async (agentType: string): Promise<AgentConfig | undefined> => {
    const agents = await getActiveAgents()
    return agents.find(agent => agent.agentType === agentType)
  }
)
```

**Avantages** :
- ✅ Évite lectures de fichiers répétées
- ✅ Performance optimale
- ✅ Cache partagé entre appels

### Invalidation du Cache

```typescript
export function clearAgentCache() {
  getActiveAgents.cache?.clear?.()
  getAllAgents.cache?.clear?.()
  getAgentByType.cache?.clear?.()
  getAvailableAgentTypes.cache?.clear?.()
}
```

**Appelé lors de** :
- Modification de fichier détectée (watcher)
- Rechargement manuel via `/agents`
- Modifications programmatiques

### Cache des Mentions

Le `MentionProcessorService` a son propre cache avec TTL :

```typescript
private agentCache: Map<string, boolean> = new Map()
private lastAgentCheck: number = 0
private CACHE_TTL = 60000  // 1 minute

private async refreshAgentCache() {
  const now = Date.now()
  if (now - this.lastAgentCheck < this.CACHE_TTL) {
    return  // Cache still fresh
  }

  this.agentCache.clear()
  const agents = await getActiveAgents()
  
  for (const agent of agents) {
    this.agentCache.set(agent.agentType, true)
  }
  
  this.lastAgentCheck = now
}
```

---

## 7. Hot Reload et Watchers

### Surveillance des Fichiers

Le système surveille **4 répertoires simultanément** :

```typescript
export async function startAgentWatcher(onChange?: () => void) {
  const userClaudeDir = join(homedir(), '.claude', 'agents')
  const userKodeDir = join(homedir(), '.kode', 'agents')
  const projectClaudeDir = join(getCwd(), '.claude', 'agents')
  const projectKodeDir = join(getCwd(), '.kode', 'agents')

  // Watch all directories
  watchDirectory(userClaudeDir, 'user/.claude')
  watchDirectory(userKodeDir, 'user/.kode')
  watchDirectory(projectClaudeDir, 'project/.claude')
  watchDirectory(projectKodeDir, 'project/.kode')
}

function watchDirectory(dirPath: string, label: string) {
  if (!existsSync(dirPath)) return

  const watcher = watch(dirPath, { recursive: false }, async (eventType, filename) => {
    if (filename && filename.endsWith('.md')) {
      console.log(`🔄 Agent configuration changed in ${label}: ${filename}`)
      clearAgentCache()
      onChange?.()
    }
  })

  watchers.push(watcher)
}
```

### Détection de Changements

**Événements surveillés** :
- Création de fichier `.md`
- Modification de fichier `.md`
- Suppression de fichier `.md`

**Réaction** :
1. Log du changement
2. Invalidation immédiate du cache
3. Callback optionnel (ex: rafraîchir UI)
4. Rechargement automatique au prochain accès

### Gestion du Cycle de Vie

```typescript
let watchers: FSWatcher[] = []

export async function stopAgentWatcher() {
  for (const watcher of watchers) {
    try {
      watcher.close()
    } catch (err) {
      console.error('Failed to close file watcher:', err)
    }
  }
  watchers = []
}
```

**Exemple d'utilisation** :

```bash
# Terminal 1 : Kode en cours
kode

# Terminal 2 : Éditer un agent
nano ~/.kode/agents/my-agent.md
# Sauvegarder

# Terminal 1 : Kode détecte automatiquement
# 🔄 Agent configuration changed in user/.kode: my-agent.md
# L'agent est immédiatement disponible !
```

---

## 8. Sélection et Exécution

### Invocation via TaskTool

```typescript
const inputSchema = z.object({
  description: z.string(),           // Description courte
  prompt: z.string(),                // Prompt complet
  model_name: z.string().optional(), // Surcharge modèle
  subagent_type: z.string().optional() // Type d'agent
})
```

### Résolution de l'Agent

```typescript
async function resolveAgent(subagent_type?: string) {
  // Défaut: general-purpose
  const agentType = subagent_type || 'general-purpose'

  // Chargement dynamique
  const agentConfig = await getAgentByType(agentType)

  if (!agentConfig) {
    // Afficher les agents disponibles
    const availableTypes = await getAvailableAgentTypes()
    throw new Error(`Agent "${agentType}" not found. Available: ${availableTypes.join(', ')}`)
  }

  return agentConfig
}
```

### Application de la Configuration

```typescript
// 1. Application du système prompt
let effectivePrompt = prompt
if (agentConfig.systemPrompt) {
  effectivePrompt = `${agentConfig.systemPrompt}\n\n${prompt}`
}

// 2. Application du modèle
let effectiveModel = model_name || 'task'
if (!model_name && agentConfig.model_name) {
  if (agentConfig.model_name !== 'inherit') {
    effectiveModel = agentConfig.model_name
  }
}

// 3. Filtrage des outils
let tools = await getTaskTools(safeMode)
const toolFilter = agentConfig.tools

if (toolFilter && toolFilter !== '*') {
  tools = tools.filter(tool => toolFilter.includes(tool.name))
}
```

### Exécution

```typescript
for await (const message of query(
  [createUserMessage(effectivePrompt)],
  taskPrompt,
  context,
  hasPermissionsToUseTool,
  {
    options: { 
      model: effectiveModel, 
      tools, 
      verbose, 
      safeMode 
    },
    agentId: generateAgentId(),
    messageId,
    abortController
  }
)) {
  yield { type: 'progress', content: message }
}
```

---

## 9. Guide de Création d'Agents

### Étapes de Création

#### 1. Choisir l'Emplacement

**Global** (tous vos projets) :
```bash
~/.kode/agents/mon-agent.md
```

**Projet spécifique** :
```bash
./.kode/agents/mon-agent.md
```

**Compatible Claude Code** :
```bash
~/.claude/agents/mon-agent.md  # Global
./.claude/agents/mon-agent.md  # Projet
```

#### 2. Créer le Fichier

```bash
touch ~/.kode/agents/security-auditor.md
```

#### 3. Définir la Configuration

```markdown
---
name: security-auditor
description: "Specialized in finding security vulnerabilities"
tools: ["Read", "Grep", "Glob"]
color: red
model_name: main
---

You are a security auditing specialist.

Your expertise:
1. Common vulnerabilities (SQL injection, XSS, CSRF, etc.)
2. Authentication and authorization issues
3. Cryptographic weaknesses
4. Dependency vulnerabilities
5. Security best practices

Guidelines:
- Always check for input validation
- Look for hardcoded secrets
- Verify secure communication
- Check error handling
- Suggest fixes with code

When auditing:
1. Start with high-risk areas
2. Check dependencies
3. Review configs
4. Verify access controls
5. Provide prioritized recommendations
```

#### 4. Tester l'Agent

L'agent est **immédiatement disponible** (hot reload) :

```bash
kode

# Vérifier qu'il est chargé
> /agents
# Vous devez voir "security-auditor"

# Ou invoquer directement
> @security-auditor Audit this codebase
```

### Conventions de Nommage

**Identifiants (name)** :
- kebab-case obligatoire
- Descriptif et unique
- Exemples : `code-writer`, `security-auditor`, `test-specialist`

**Fichiers** :
- Même nom que l'identifiant + `.md`
- Exemples : `code-writer.md`, `security-auditor.md`

**Couleurs** :
- `blue`, `green`, `red`, `yellow`, `cyan`, `magenta`, `white`
- Utilisé pour différenciation visuelle dans l'UI

---

## 10. Exemples Pratiques

### Agent de Documentation

```markdown
---
name: docs-writer
description: "Documentation specialist for technical docs"
tools: ["Read", "Write", "Edit", "Grep", "Glob"]
color: purple
---

You are a documentation specialist.

Mission: Create clear, complete, well-structured documentation.

Format:
- Professional Markdown
- Code examples
- Well-organized sections
- Table of contents

Always:
1. Read existing code
2. Identify key points
3. Write pedagogically
4. Include examples
```

### Agent de Tests

```markdown
---
name: test-writer
description: "Specialized in writing comprehensive tests"
tools: ["Read", "Write", "Edit", "Bash", "Grep"]
model_name: main
---

You are a testing expert.

Create tests:
- Complete unit tests
- Edge cases covered
- Clear assertions
- Proper setup/teardown

Framework: Auto-detect (Jest, Vitest, etc.)

Workflow:
1. Read code to test
2. Identify test cases
3. Write comprehensive tests
4. Run tests to verify
```

### Agent de Recherche Rapide

```markdown
---
name: quick-search
description: "Fast file and pattern finding"
tools: ["Grep", "Glob"]
color: green
model_name: quick
---

You are optimized for speed.

Use Grep and Glob efficiently.

Be concise in results.
Provide exact matches.
```

### Agent Architect

```markdown
---
name: architect
description: "Software architecture and design"
tools: ["Read", "Grep", "Glob", "Write"]
color: cyan
model_name: reasoning
---

You are a software architect.

Analyze:
- Code structure
- Design patterns
- Architecture decisions
- Scalability issues
- Best practices

Provide:
- Architecture diagrams (text)
- Improvement suggestions
- Refactoring recommendations
- Documentation
```

---

## Conclusion

Le système d'agents de Kode est un **système sophistiqué et flexible** qui combine :

✅ **Hiérarchie à 5 niveaux** pour configuration granulaire  
✅ **Compatibilité bidirectionnelle** avec Claude Code  
✅ **Hot reload** pour développement rapide  
✅ **Cache multi-couches** pour performance  
✅ **Format simple** (YAML + Markdown)  
✅ **Filtrage d'outils** pour sécurité  
✅ **Surcharge de modèles** pour optimisation  

Cette architecture permet de créer des **agents hautement spécialisés** tout en maintenant la flexibilité et la performance nécessaires pour un environnement de développement professionnel.

---

[← Retour à l'index](./README.md) | [← Modèles](./03-systeme-modeles.md) | [Outils →](./05-systeme-outils.md)
