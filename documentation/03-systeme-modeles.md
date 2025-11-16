# Système Multi-Modèles de Kode

> Architecture avancée pour orchestrer plusieurs modèles d'IA simultanément

[← Retour à l'index](./README.md) | [← Architecture](./02-architecture.md) | [Agents →](./04-systeme-agents.md)

---

## Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Architecture Globale](#2-architecture-globale)
3. [ModelManager - Orchestration](#3-modelmanager---orchestration)
4. [Model Capabilities](#4-model-capabilities)
5. [Adaptateurs de Modèles](#5-adaptateurs-de-modèles)
6. [Ajouter un Nouveau Modèle](#6-ajouter-un-nouveau-modèle)
7. [Capacités par Type de Modèle](#7-capacités-par-type-de-modèle)
8. [Exemples Pratiques](#8-exemples-pratiques)

---

## 1. Vue d'Ensemble

### Innovation Clé de Kode

Le système multi-modèles de Kode est **l'une des architectures les plus avancées** dans le domaine des outils de développement assistés par IA. Contrairement aux solutions traditionnelles limitées à un ou deux modèles, Kode permet :

- ✅ **Support illimité de modèles** (20+ providers supportés)
- ✅ **Switching dynamique** sans redémarrage de session
- ✅ **Model Pointers** pour usages spécifiques (main, task, reasoning, quick)
- ✅ **Adaptateurs unifiés** masquant les différences d'APIs
- ✅ **Auto-correction** des erreurs de paramètres
- ✅ **Support complet GPT-5 Responses API**

### Comparaison avec d'Autres Outils

| Fonctionnalité | Kode | Cursor | Copilot | Claude CLI |
|---|:---:|:---:|:---:|:---:|
| Modèles supportés | ∞ | 2-3 | 1 | 1 |
| Switching dynamique | ✅ | ❌ | ❌ | ❌ |
| Model Pointers | ✅ (4) | ❌ | ❌ | ❌ |
| Providers personnalisés | ✅ | ⚠️ Limité | ❌ | ❌ |
| GPT-5 Responses API | ✅ Complet | ❌ | ❌ | ❌ |
| Auto-correction erreurs | ✅ | ❌ | ❌ | ❌ |
| State Management | ✅ (GPT-5) | ❌ | ❌ | ❌ |

---

## 2. Architecture Globale

### Diagramme des Composants

```
┌─────────────────────────────────────────────────────────────┐
│                        Kode CLI                              │
├─────────────────────────────────────────────────────────────┤
│                      ModelManager                            │
│  - Gestion des profils de modèles                           │
│  - System de pointeurs (main, task, reasoning, quick)       │
│  - Switching dynamique                                       │
│  - Validation et auto-réparation                            │
├─────────────────────────────────────────────────────────────┤
│                  ModelAdapterFactory                         │
│  - Sélection de l'adaptateur approprié                      │
│  - Détermination de l'API à utiliser                        │
├──────────────────┬──────────────────┬───────────────────────┤
│  ResponsesAPI    │  ChatCompletions │   Anthropic SDK       │
│   Adapter        │     Adapter      │     Direct            │
│  (GPT-5)         │  (Standard)      │  (Claude, etc.)       │
├──────────────────┴──────────────────┴───────────────────────┤
│                    Service Layer                             │
│  openai.ts  |  claude.ts  |  mcpClient.ts                   │
├─────────────────────────────────────────────────────────────┤
│                  API Providers (20+)                         │
│  OpenAI | Anthropic | DeepSeek | Mistral | Groq |           │
│  Gemini | Ollama | Azure | Bedrock | Vertex | ...           │
└─────────────────────────────────────────────────────────────┘
```

### Flux d'Exécution

```
1. User Request
        ↓
2. ModelManager.resolveModel('main')
        ↓
3. ModelProfile {
     name: "GPT-5 Main"
     modelName: "gpt-5"
     provider: "openai"
     apiKey: "sk-..."
     ...
   }
        ↓
4. ModelAdapterFactory.createAdapter(profile)
        ↓
5. ┌────────────────────────────┐
   │ Quelle API?                │
   ├──────────────┬─────────────┤
   │ Responses    │ Chat        │
   │ API          │ Completions │
   │ (GPT-5)      │ (Standard)  │
   └──────────────┴─────────────┘
        ↓
6. API Request (avec transformations)
        ↓
7. API Response
        ↓
8. Unified Response Format
        ↓
9. Return to User
```

---

## 3. ModelManager - Orchestration

**Fichier** : `src/utils/model.ts`

### Structure d'un ModelProfile

```typescript
interface ModelProfile {
  // === IDENTIFICATION ===
  name: string                    // Nom convivial : "GPT-5 Main"
  modelName: string               // ID du modèle : "gpt-5"
  provider: string                // Provider : "openai"

  // === CONNEXION ===
  apiKey: string                  // Clé API
  baseURL?: string                // URL personnalisée

  // === CAPACITÉS ===
  contextLength: number           // Limite de contexte : 200000
  maxTokens?: number              // Tokens max de sortie : 32768

  // === GPT-5 SPÉCIFIQUE ===
  reasoningEffort?: 'minimal' | 'low' | 'medium' | 'high'
  isGPT5?: boolean
  validationStatus?: 'valid' | 'auto_repaired' | 'invalid'

  // === ÉTAT ===
  isActive: boolean               // Si le modèle est actif
  createdAt: number               // Timestamp de création
  lastUsed?: number               // Dernier usage
}
```

### Model Pointers - Innovation Majeure

**Concept** : Assigner différents modèles pour différents usages.

```typescript
interface ModelPointers {
  main: string        // Modèle principal pour chat interactif
  task: string        // Modèle pour sous-tâches (TaskTool/agents)
  reasoning: string   // Modèle pour raisonnement profond
  quick: string       // Modèle rapide pour opérations légères
}
```

**Exemple de configuration** :

```json
{
  "modelPointers": {
    "main": "gpt-5",
    "task": "claude-sonnet-4-20250514",
    "reasoning": "deepseek-reasoner",
    "quick": "gpt-4o-mini"
  }
}
```

**Avantages** :

1. **Optimisation des coûts** : Modèles économiques pour tâches simples
2. **Optimisation des performances** : Meilleur modèle pour chaque usage
3. **Flexibilité** : Changement facile sans reconfigurer tout
4. **Spécialisation** : Claude pour outils, GPT-5 pour reasoning

**Usage dans le code** :

```typescript
const modelManager = getModelManager()

// Résoudre un pointeur vers un profil
const mainModel = modelManager.getModel('main')
// → ModelProfile { name: "GPT-5 Main", modelName: "gpt-5", ... }

// Utilisation directe
const taskModel = modelManager.getModel('task')
const reasoningModel = modelManager.getModel('reasoning')
const quickModel = modelManager.getQuickModel()
```

### Switching Dynamique

**Fonctionnalité** : Changer de modèle en cours de session sans redémarrer.

```typescript
class ModelManager {
  switchToNextModelWithContextCheck(
    currentContextTokens: number
  ): {
    success: boolean
    modelName: string | null
    previousModelName: string | null
    contextOverflow: boolean      // Si contexte > 80% limite
    usagePercentage: number
    blocked: boolean
    message: string
  }
}
```

**Fonctionnement** :

1. Cycle à travers TOUS les modèles configurés (actifs + inactifs)
2. Vérifie compatibilité du contexte (80% du contextLength)
3. Active automatiquement le modèle s'il était inactif
4. Sauvegarde l'état pour persistance

**Exemple d'utilisation** :

```typescript
// Dans le REPL : raccourci clavier Shift+M
const result = modelManager.switchToNextModel(50000)  // 50K tokens actuels

if (result.success) {
  console.log(`✅ Switched to ${result.modelName}`)
  // → "✅ Switched to Claude Sonnet 4 (2/4) [anthropic]"

  if (result.contextOverflow) {
    console.warn(`⚠️ Context usage: ${result.usagePercentage}%`)
    // Peut déclencher auto-compaction
  }
} else {
  console.log(result.message)
  // → "⚠️ Only one model configured. Use /model to add more."
}
```

### Validation et Auto-Réparation GPT-5

**Innovation** : Détection et correction automatique des configurations.

```typescript
function validateAndRepairGPT5Profile(
  profile: ModelProfile
): ModelProfile {
  const repaired = { ...profile }
  let wasRepaired = false

  // Détection GPT-5
  if (isGPT5ModelName(profile.modelName)) {
    // 1. Répare reasoningEffort si invalide
    const validEfforts = ['minimal', 'low', 'medium', 'high']
    if (!validEfforts.includes(profile.reasoningEffort)) {
      repaired.reasoningEffort = 'medium'
      wasRepaired = true
    }

    // 2. Répare contextLength si trop petit
    if (profile.contextLength < 128000) {
      repaired.contextLength = 128000
      wasRepaired = true
    }

    // 3. Répare maxTokens si trop petit
    if (profile.maxTokens < 4000) {
      repaired.maxTokens = 8192
      wasRepaired = true
    }

    // 4. Marque comme GPT-5
    repaired.isGPT5 = true
  }

  // Status de validation
  repaired.validationStatus = wasRepaired ? 'auto_repaired' : 'valid'

  return repaired
}
```

**Exécution** : Au démarrage de Kode (`validateAndRepairAllGPT5Profiles()`)

**Avantages** :
- ✅ Migrations automatiques lors des mises à jour API
- ✅ Correction des erreurs de configuration utilisateur
- ✅ Adaptation aux changements de spécifications
- ✅ Expérience utilisateur améliorée (pas d'erreurs cryptiques)

---

## 4. Model Capabilities

**Fichier** : `src/constants/modelCapabilities.ts`

### Qu'est-ce que les Capabilities ?

Un système de **métadonnées techniques** qui décrit les capacités de chaque modèle. Permet à Kode d'adapter automatiquement les requêtes API.

### Structure des Capabilities

```typescript
interface ModelCapabilities {
  // === ARCHITECTURE API ===
  apiArchitecture: {
    primary: 'chat_completions' | 'responses_api' | 'anthropic_messages'
    fallback?: 'chat_completions'
  }

  // === GESTION DES PARAMÈTRES ===
  parameters: {
    maxTokensField: 'max_tokens' | 'max_completion_tokens'
    supportsReasoningEffort: boolean
    supportsVerbosity: boolean
    temperatureMode: 'flexible' | 'fixed_one' | 'restricted'
  }

  // === APPELS D'OUTILS ===
  toolCalling: {
    mode: 'none' | 'function_calling' | 'custom_tools'
    supportsFreeform: boolean
    supportsAllowedTools: boolean
    supportsParallelCalls: boolean
  }

  // === GESTION D'ÉTAT ===
  stateManagement: {
    supportsResponseId: boolean
    supportsConversationChaining: boolean
    supportsPreviousResponseId: boolean
  }

  // === STREAMING ===
  streaming: {
    supported: boolean
    includesUsage: boolean
  }
}
```

### Exemples de Capabilities

#### GPT-5 (Responses API)

```typescript
const GPT5_CAPABILITIES: ModelCapabilities = {
  apiArchitecture: {
    primary: 'responses_api',
    fallback: 'chat_completions'
  },
  parameters: {
    maxTokensField: 'max_completion_tokens',  // ⚠️ Différent de GPT-4
    supportsReasoningEffort: true,            // ✨ Contrôle reasoning
    supportsVerbosity: true,                  // ✨ Contrôle verbosité
    temperatureMode: 'fixed_one'              // 🔒 Toujours 1
  },
  toolCalling: {
    mode: 'custom_tools',                     // 🎯 Format personnalisé
    supportsFreeform: true,
    supportsAllowedTools: true,               // ✨ Filtrage d'outils
    supportsParallelCalls: true
  },
  stateManagement: {
    supportsResponseId: true,                 // ✨ Persistance session
    supportsConversationChaining: true,
    supportsPreviousResponseId: true
  },
  streaming: {
    supported: false,                         // ⚠️ Pas encore disponible
    includesUsage: true
  }
}
```

#### Claude Sonnet 4 (Anthropic)

```typescript
const CLAUDE_SONNET_4_CAPABILITIES: ModelCapabilities = {
  apiArchitecture: {
    primary: 'anthropic_messages'
  },
  parameters: {
    maxTokensField: 'max_tokens',
    supportsReasoningEffort: false,
    supportsVerbosity: false,
    temperatureMode: 'flexible'
  },
  toolCalling: {
    mode: 'function_calling',
    supportsFreeform: false,
    supportsAllowedTools: false,
    supportsParallelCalls: true
  },
  stateManagement: {
    supportsResponseId: false,
    supportsConversationChaining: false,
    supportsPreviousResponseId: false
  },
  streaming: {
    supported: true,                          // ✅ Streaming natif
    includesUsage: true
  }
}
```

#### DeepSeek Reasoner

```typescript
const DEEPSEEK_REASONER_CAPABILITIES: ModelCapabilities = {
  apiArchitecture: {
    primary: 'chat_completions'
  },
  parameters: {
    maxTokensField: 'max_tokens',
    supportsReasoningEffort: false,
    supportsVerbosity: false,
    temperatureMode: 'flexible'
  },
  toolCalling: {
    mode: 'function_calling',
    supportsFreeform: false,
    supportsAllowedTools: false,
    supportsParallelCalls: true
  },
  stateManagement: {
    supportsResponseId: false,
    supportsConversationChaining: false,
    supportsPreviousResponseId: false
  },
  streaming: {
    supported: true,
    includesUsage: true
  }
}
```

---

## 5. Adaptateurs de Modèles

### ModelAdapterFactory

**Fichier** : `src/services/modelAdapterFactory.ts`

**Rôle** : Choisir automatiquement l'adaptateur approprié pour chaque modèle.

```typescript
class ModelAdapterFactory {
  static createAdapter(modelProfile: ModelProfile): ModelAPIAdapter {
    const capabilities = getModelCapabilities(modelProfile.modelName)
    const apiType = this.determineAPIType(modelProfile, capabilities)

    switch (apiType) {
      case 'responses_api':
        return new ResponsesAPIAdapter(capabilities, modelProfile)

      case 'chat_completions':
      default:
        return new ChatCompletionsAdapter(capabilities, modelProfile)
    }
  }

  private static determineAPIType(
    modelProfile: ModelProfile,
    capabilities: ModelCapabilities
  ): 'responses_api' | 'chat_completions' {
    // Si pas de Responses API supporté
    if (capabilities.apiArchitecture.primary !== 'responses_api') {
      return 'chat_completions'
    }

    // Provider tiers : utiliser fallback
    const isOfficialOpenAI = !modelProfile.baseURL ||
      modelProfile.baseURL.includes('api.openai.com')

    if (!isOfficialOpenAI) {
      return capabilities.apiArchitecture.fallback || 'chat_completions'
    }

    // OpenAI officiel : utiliser Responses API
    return 'responses_api'
  }
}
```

### Base Adapter (Classe Abstraite)

**Fichier** : `src/services/adapters/base.ts`

```typescript
abstract class ModelAPIAdapter {
  constructor(
    protected capabilities: ModelCapabilities,
    protected modelProfile: ModelProfile
  ) {}

  // === MÉTHODES ABSTRAITES ===
  abstract createRequest(params: UnifiedRequestParams): any
  abstract parseResponse(response: any): Promise<UnifiedResponse>
  abstract buildTools(tools: Tool[]): any

  // === STREAMING OPTIONNEL ===
  async *parseStreamingResponse?(response: any): AsyncGenerator<StreamingEvent>

  // === UTILITAIRES PARTAGÉS ===
  protected getMaxTokensParam(): string {
    return this.capabilities.parameters.maxTokensField
  }

  protected getTemperature(): number {
    if (this.capabilities.parameters.temperatureMode === 'fixed_one') {
      return 1
    }
    if (this.capabilities.parameters.temperatureMode === 'restricted') {
      return Math.min(1, 0.7)
    }
    return 0.7
  }
}
```

### ResponsesAPIAdapter (GPT-5)

**Fichier** : `src/services/adapters/responsesAPI.ts` (544 lignes)

**Spécificités** :
- Format `input` au lieu de `messages`
- `instructions` au lieu de `system`
- Contrôles `reasoning.effort` et `text.verbosity`
- State management via `previous_response_id`
- Format d'outils plat (pas de nested `function`)

```typescript
class ResponsesAPIAdapter extends ModelAPIAdapter {
  createRequest(params: UnifiedRequestParams): any {
    return {
      model: this.modelProfile.modelName,
      input: this.convertMessagesToInput(params.messages),
      instructions: this.buildInstructions(params.systemPrompt),
      max_output_tokens: params.maxTokens,
      stream: true,
      temperature: 1,  // Forcé pour GPT-5

      // Reasoning control
      reasoning: {
        effort: params.reasoningEffort || 'medium'
      },

      // Verbosity control (pour coding)
      text: {
        verbosity: params.verbosity || 'high'
      },

      // Tools au format plat
      tools: this.buildTools(params.tools || []),
      tool_choice: 'auto',
      parallel_tool_calls: true,

      // State management
      previous_response_id: params.previousResponseId,

      // Include reasoning content
      include: ['reasoning.encrypted_content']
    }
  }

  buildTools(tools: Tool[]): any[] {
    // Format plat (pas de nested 'function' object comme Chat Completions)
    return tools.map(tool => ({
      type: 'function',
      name: tool.name,
      description: tool.description,
      parameters: tool.inputJSONSchema || zodToJsonSchema(tool.inputSchema)
    }))
  }

  private convertMessagesToInput(messages: any[]): any[] {
    // Convertit messages Chat Completions → format Responses API
    const inputItems = []

    for (const message of messages) {
      if (message.role === 'tool') {
        // Tool results
        inputItems.push({
          type: 'function_call_output',
          call_id: message.tool_call_id,
          output: message.content
        })
      }
      else if (message.role === 'assistant' && message.tool_calls) {
        // Tool calls
        for (const tc of message.tool_calls) {
          inputItems.push({
            type: 'function_call',
            name: tc.function.name,
            arguments: tc.function.arguments,
            call_id: tc.id
          })
        }
      }
      else {
        // Regular message
        inputItems.push({
          type: 'message',
          role: message.role === 'assistant' ? 'assistant' : 'user',
          content: [{ type: 'input_text', text: message.content }]
        })
      }
    }

    return inputItems
  }
}
```

### ChatCompletionsAdapter (Standard)

**Fichier** : `src/services/adapters/chatCompletions.ts` (91 lignes)

**Spécificités** :
- Format standard OpenAI
- Support de tous les providers compatibles
- Streaming natif

```typescript
class ChatCompletionsAdapter extends ModelAPIAdapter {
  createRequest(params: UnifiedRequestParams): any {
    const request: any = {
      model: this.modelProfile.modelName,
      messages: this.buildMessages(params.systemPrompt, params.messages),
      [this.getMaxTokensParam()]: params.maxTokens,
      temperature: this.getTemperature()
    }

    // Tools
    if (params.tools && params.tools.length > 0) {
      request.tools = this.buildTools(params.tools)
      request.tool_choice = 'auto'
    }

    // Reasoning effort (GPT-5 via Chat Completions)
    if (this.capabilities.parameters.supportsReasoningEffort && params.reasoningEffort) {
      request.reasoning_effort = params.reasoningEffort
    }

    // Streaming
    if (params.stream) {
      request.stream = true
      request.stream_options = { include_usage: true }
    }

    // O1 special handling
    if (this.modelProfile.modelName.startsWith('o1')) {
      delete request.temperature
      delete request.stream
    }

    return request
  }

  async parseResponse(response: any): Promise<UnifiedResponse> {
    return {
      id: response.id,
      content: response.choices[0].message.content,
      toolCalls: response.choices[0].message.tool_calls || [],
      usage: {
        promptTokens: response.usage.prompt_tokens,
        completionTokens: response.usage.completion_tokens
      }
    }
  }

  buildTools(tools: Tool[]): any[] {
    // Format standard OpenAI
    return tools.map(tool => ({
      type: 'function',
      function: {
        name: tool.name,
        description: tool.description,
        parameters: tool.inputJSONSchema || zodToJsonSchema(tool.inputSchema)
      }
    }))
  }
}
```

---

## 6. Ajouter un Nouveau Modèle

### Méthode Simple (Provider Connu)

Via la commande `/model` dans Kode CLI :

```bash
# 1. Lancer Kode
kode

# 2. Ouvrir la configuration de modèle
> /model

# 3. Interface interactive :
#    - Choisir "Add new model"
#    - Sélectionner le provider (ex: OpenAI)
#    - Entrer l'API Key
#    - Choisir le modèle dans la liste
#    - Configurer les paramètres
#    - Assigner les pointeurs
```

### Méthode Avancée (Provider Personnalisé)

#### Étape 1 : Ajouter le Provider

**Fichier** : `src/constants/models.ts`

```typescript
export const providers = {
  // ... providers existants
  'my-custom-provider': {
    name: 'My Custom AI Provider',
    baseURL: 'https://api.mycustom.ai/v1'
  }
}
```

#### Étape 2 : Définir les Modèles

```typescript
export default {
  // ... autres providers
  'my-custom-provider': [
    {
      model: 'my-custom-model-v1',
      max_tokens: 16384,
      max_input_tokens: 128000,
      max_output_tokens: 16384,
      input_cost_per_token: 0.000001,
      output_cost_per_token: 0.000002,
      provider: 'my-custom-provider',
      mode: 'chat',
      supports_function_calling: true,
      supports_vision: false,
      supports_prompt_caching: false
    }
  ]
}
```

#### Étape 3 : Définir les Capabilities

**Fichier** : `src/constants/modelCapabilities.ts`

```typescript
export function getModelCapabilities(modelName: string): ModelCapabilities {
  // ... cases existants

  if (modelName === 'my-custom-model-v1') {
    return {
      apiArchitecture: {
        primary: 'chat_completions'
      },
      parameters: {
        maxTokensField: 'max_tokens',
        supportsReasoningEffort: false,
        supportsVerbosity: false,
        temperatureMode: 'flexible'
      },
      toolCalling: {
        mode: 'function_calling',
        supportsFreeform: false,
        supportsAllowedTools: false,
        supportsParallelCalls: true
      },
      stateManagement: {
        supportsResponseId: false,
        supportsConversationChaining: false,
        supportsPreviousResponseId: false
      },
      streaming: {
        supported: true,
        includesUsage: true
      }
    }
  }
}
```

#### Étape 4 : Ajouter le Support dans OpenAI Service

**Fichier** : `src/services/openai.ts`

```typescript
const isOpenAICompatible = [
  'openai',
  'deepseek',
  'groq',
  'my-custom-provider',  // 🎯 Ajouter ici
  // ...
].includes(provider)
```

#### Étape 5 : Tester

```bash
kode
> /model
# Ajouter le modèle via l'interface
# Provider: my-custom-provider
# API Key: <votre-clé>
# Model: my-custom-model-v1

# Tester
> Hello, can you help me?
```

---

## 7. Capacités par Type de Modèle

### GPT-5 (Responses API)

**Capacités** :
- ✅ Raisonnement contrôlé (`reasoning_effort`: minimal, low, medium, high)
- ✅ Contrôle de verbosité (`verbosity`: low, medium, high)
- ✅ Persistance de session (`previous_response_id`)
- ✅ Outils personnalisés (format plat)
- ✅ `allowed_tools` pour restreindre les outils
- ✅ Reasoning encrypted content

**Limitations** :
- ⚠️ Température fixée à 1 (non configurable)
- ⚠️ Pas de streaming (encore)
- ⚠️ Format de requête différent
- ⚠️ Uniquement sur OpenAI officiel

**Paramètres Spécifiques** :
```typescript
{
  max_output_tokens: 32768,
  temperature: 1,
  reasoning: { effort: 'medium' },
  text: { verbosity: 'high' },
  previous_response_id: '...'
}
```

### GPT-4o (Chat Completions)

**Capacités** :
- ✅ Streaming rapide
- ✅ Vision (images)
- ✅ Prompt caching
- ✅ Function calling standard
- ✅ Température flexible (0-2)

**Limitations** :
- ❌ Pas de raisonnement contrôlé
- ❌ Pas de persistance de session
- ❌ Pas de verbosity control

**Paramètres Spécifiques** :
```typescript
{
  max_tokens: 16384,
  temperature: 0.7,
  stream: true,
  stream_options: { include_usage: true }
}
```

### Claude Sonnet 4 (Anthropic)

**Capacités** :
- ✅ Prompt caching (4 blocs max)
- ✅ Vision multimodale
- ✅ Prefill de l'assistant
- ✅ Streaming natif
- ✅ Très bon pour le coding

**Limitations** :
- ❌ Pas de reasoning effort control
- ❌ Contexte limité (200K tokens)
- ⚠️ Cache limité à 4 blocs

**Paramètres Spécifiques** :
```typescript
{
  max_tokens: 8192,
  system: [{
    type: 'text',
    text: '...',
    cache_control: { type: 'ephemeral' }
  }],
  thinking: { max_tokens: 10000 }
}
```

### O1 (Reasoning Models)

**Capacités** :
- ✅ Raisonnement profond
- ✅ Résolution de problèmes complexes
- ✅ `reasoning_effort` control

**Limitations** :
- ❌ Pas de streaming
- ❌ Température non configurable
- ⚠️ Plus lent

**Paramètres Spécifiques** :
```typescript
{
  max_completion_tokens: 100000,
  // PAS de temperature
  // PAS de stream
  reasoning_effort: 'high'
}
```

### DeepSeek Reasoner

**Capacités** :
- ✅ Très bon rapport qualité/prix
- ✅ Prompt caching
- ✅ `reasoning_content` dans réponse

**Limitations** :
- ⚠️ Moins de features avancées
- ⚠️ Contexte limité (65K tokens)

**Paramètres Spécifiques** :
```typescript
{
  max_tokens: 8192,
  // Retourne reasoning_content au lieu de reasoning
}
```

---

## 8. Exemples Pratiques

### Configuration Multi-Modèles pour Développeur

```json
{
  "modelProfiles": [
    {
      "name": "GPT-5 Main",
      "modelName": "gpt-5",
      "provider": "openai",
      "apiKey": "sk-...",
      "baseURL": "https://api.openai.com/v1",
      "contextLength": 200000,
      "maxTokens": 32768,
      "reasoningEffort": "medium",
      "isActive": true
    },
    {
      "name": "Claude Sonnet 4",
      "modelName": "claude-sonnet-4-20250514",
      "provider": "anthropic",
      "apiKey": "sk-ant-...",
      "contextLength": 200000,
      "maxTokens": 8192,
      "isActive": true
    },
    {
      "name": "DeepSeek Reasoner",
      "modelName": "deepseek-reasoner",
      "provider": "deepseek",
      "apiKey": "sk-...",
      "contextLength": 65536,
      "maxTokens": 8192,
      "isActive": true
    },
    {
      "name": "GPT-4o Mini Fast",
      "modelName": "gpt-4o-mini",
      "provider": "openai",
      "apiKey": "sk-...",
      "contextLength": 128000,
      "maxTokens": 16384,
      "isActive": true
    }
  ],
  "modelPointers": {
    "main": "gpt-5",
    "task": "claude-sonnet-4-20250514",
    "reasoning": "deepseek-reasoner",
    "quick": "gpt-4o-mini"
  }
}
```

### Switching de Modèle en Action

```typescript
// Initial: main = GPT-5
const current = modelManager.getCurrentModel()
// → "gpt-5"

// User presses Shift+M
const result = modelManager.switchToNextModel(150000)  // 150K tokens actuels

if (result.success) {
  console.log(result.message)
  // → "✅ Switched to Claude Sonnet 4 (2/4) [anthropic]"

  if (result.contextOverflow) {
    console.warn(`Context usage: ${result.usagePercentage}%`)
    // → "Context usage: 75%" (150K / 200K)
  }
}

// Prochain switch
const result2 = modelManager.switchToNextModel(150000)
// → "✅ Switched to DeepSeek Reasoner (3/4) [deepseek]"

// Attention: contexte trop grand pour DeepSeek (65K max)
if (result2.blocked) {
  console.warn('⚠️ Context overflow! Auto-compaction needed.')
}
```

### Utilisation des Pointeurs

```typescript
// Dans TaskTool : utiliser le modèle "task"
const taskModel = modelManager.getModel('task')
// → Claude Sonnet 4 (meilleur pour outils)

// Pour problème complexe : utiliser "reasoning"
const reasoningModel = modelManager.getModel('reasoning')
// → DeepSeek Reasoner (spécialisé en raisonnement)

// Pour question rapide : utiliser "quick"
const quickModel = modelManager.getQuickModel()
// → GPT-4o Mini (rapide et économique)
```

---

## Conclusion

Le système multi-modèles de Kode est une **architecture sophistiquée et innovante** qui offre :

1. **Flexibilité maximale** : Support illimité de modèles et providers
2. **Abstraction élégante** : Adaptateurs qui masquent les différences d'APIs
3. **Intelligence** : Auto-correction, gestion du contexte, switching optimal
4. **Innovation** : Model Pointers, State Management, Interface unifiée
5. **Extensibilité** : Facile d'ajouter nouveaux modèles et providers
6. **Production-ready** : Gestion d'erreurs robuste, retry, validation

C'est un système qui positionne Kode comme l'un des outils de développement IA les plus avancés et flexibles du marché.

---

[← Retour à l'index](./README.md) | [← Architecture](./02-architecture.md) | [Agents →](./04-systeme-agents.md)
