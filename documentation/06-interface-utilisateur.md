# Interface Utilisateur - REPL Ink/React et Composants

> Documentation technique de l'interface terminal interactive de Kode : REPL à 809 lignes, 70+ composants React/Ink, 3 modes d'interaction

---

## Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Architecture React/Ink](#2-architecture-reactink)
3. [REPL Principal](#3-repl-principal)
4. [Les 3 Modes d'Interaction](#4-les-3-modes-dinteraction)
5. [Composants Majeurs](#5-composants-majeurs)
6. [Système de Rendu](#6-système-de-rendu)
7. [Interactions Utilisateur](#7-interactions-utilisateur)
8. [Gestion d'État](#8-gestion-détat)
9. [Catalogue des Composants](#9-catalogue-des-composants)

---

## 1. Vue d'Ensemble

### 1.1 Philosophie de l'Interface

Kode implémente une interface terminal **full-stack React** via Ink :

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  POURQUOI REACT DANS LE TERMINAL ?             │
│                                                 │
│  ✅ Composants réutilisables                   │
│  ✅ État déclaratif (useState, useEffect)      │
│  ✅ Hooks personnalisés                        │
│  ✅ Rendu conditionnel propre                  │
│  ✅ Ecosystem React complet                    │
│  ✅ Test avec React Testing Library            │
│  ✅ Hot reload pendant développement           │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Ink** = React pour les terminaux
- Même syntaxe JSX que React web
- Même patterns de composants
- Rendu dans le terminal au lieu du DOM
- Flexbox pour layouts
- Composants Box et Text de base

### 1.2 Stack Technique

```typescript
┌─────────────────────────────────────┐
│                                     │
│  COUCHE PRÉSENTATION                │
│  ━━━━━━━━━━━━━━━━━━━━              │
│                                     │
│  • React 18                         │
│  • Ink 4.x (React renderer)        │
│  • TypeScript strict               │
│  • 70+ composants custom           │
│                                     │
├─────────────────────────────────────┤
│                                     │
│  COUCHE LOGIQUE                     │
│  ━━━━━━━━━━━━━━━━━━━━              │
│                                     │
│  • 15+ hooks personnalisés         │
│  • Context API (PermissionContext) │
│  • État global (ModelManager)      │
│  • Reactive updates                │
│                                     │
├─────────────────────────────────────┤
│                                     │
│  COUCHE INTERACTION                 │
│  ━━━━━━━━━━━━━━━━━━━━              │
│                                     │
│  • useInput (Ink) - Gestion clavier│
│  • Readline integration            │
│  • ANSI escape codes               │
│  • Terminal control                │
│                                     │
└─────────────────────────────────────┘
```

### 1.3 Statistiques

| Métrique | Valeur |
|----------|--------|
| **Fichier principal** | `REPL.tsx` (809 lignes) |
| **Composants totaux** | 70+ fichiers `.tsx` |
| **Hooks personnalisés** | 15+ hooks |
| **Lignes de code UI** | ~8,000+ lignes |
| **Dépendances UI** | Ink, React, chalk, boxen, etc. |
| **Modes d'interaction** | 3 (Prompt, Bash, Koding) |

---

## 2. Architecture React/Ink

### 2.1 Point d'Entrée

```
src/entrypoints/cli.tsx
    ↓
src/screens/REPL.tsx (809 lignes)
    ↓
    ├─ Logo (bannière)
    ├─ Messages (historique)
    ├─ ToolJSX (outil actif)
    ├─ PermissionRequest (demande permission)
    ├─ BinaryFeedback (choix A/B)
    ├─ MessageSelector (sélection message)
    ├─ PromptInput (saisie utilisateur)
    └─ ModeIndicator (mode actuel)
```

### 2.2 Hiérarchie de Composants

```
<REPL>
  <PermissionProvider>
    <Box flexDirection="column">

      {/* Bannière */}
      <Logo
        mcpClients={mcpClients}
        isDefaultModel={isDefaultModel}
        updateBannerVersion={updateVersion}
      />

      {/* Historique messages */}
      <Static items={visibleMessages}>
        {message => (
          <Message
            message={message}
            tools={tools}
            verbose={verbose}
            {...other}
          />
        )}
      </Static>

      {/* Outil actif (ex: TaskTool progress) */}
      {toolJSX?.jsx}

      {/* Demande de permission */}
      {toolUseConfirm && (
        <PermissionRequest
          toolUseConfirm={toolUseConfirm}
          setToolUseConfirm={setToolUseConfirm}
          {...other}
        />
      )}

      {/* Binary feedback (A/B choice) */}
      {binaryFeedbackContext && (
        <BinaryFeedback
          context={binaryFeedbackContext}
          onResolve={...}
        />
      )}

      {/* Message selector (Ctrl+M) */}
      {isMessageSelectorVisible && (
        <MessageSelector
          messages={messages}
          onSelect={...}
          onCancel={...}
        />
      )}

      {/* Saisie utilisateur */}
      {shouldShowPromptInput && (
        <PromptInput
          commands={commands}
          mode={inputMode}
          input={inputValue}
          onInputChange={setInputValue}
          onModeChange={setInputMode}
          onQuery={handleQuery}
          {...other}
        />
      )}

      {/* Indicateur de mode */}
      <ModeIndicator />

    </Box>
  </PermissionProvider>
</REPL>
```

### 2.3 Flux de Données

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  UTILISATEUR                                    │
│    ↓ (saisie)                                   │
│  PromptInput                                    │
│    ↓ (processUserInput)                        │
│  REPL State (setMessages)                      │
│    ↓                                            │
│  query() / processUserInput()                  │
│    ↓ (AsyncGenerator)                          │
│  Streaming updates                             │
│    ↓                                            │
│  REPL State Update                             │
│    ↓                                            │
│  React Re-render                               │
│    ↓                                            │
│  Terminal Display                              │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 3. REPL Principal

### 3.1 Structure du Fichier

**Fichier** : `src/screens/REPL.tsx` (809 lignes)

**Sections** :
1. **Imports** (1-72) : Dépendances et types
2. **Types** (73-99) : Props, BinaryFeedbackContext
3. **Component** (101-809) : Fonction REPL principale
4. **Hooks** (118-223) : État et effets
5. **Handlers** (225-600) : Gestion événements
6. **Render** (600-809) : JSX de rendu

### 3.2 Props du REPL

```typescript
type Props = {
  commands: Command[]              // Commandes disponibles
  safeMode?: boolean              // Mode sécurisé
  debug?: boolean                 // Mode debug
  initialForkNumber?: number      // Numéro de fork
  initialPrompt: string | undefined // Prompt initial
  messageLogName: string          // Nom du log
  shouldShowPromptInput: boolean  // Afficher input
  tools: Tool[]                   // Outils disponibles
  verbose: boolean | undefined    // Mode verbeux
  initialMessages?: MessageType[] // Messages initiaux
  mcpClients?: WrappedClient[]   // Clients MCP
  isDefaultModel?: boolean        // Modèle par défaut ?
  initialUpdateVersion?: string   // Version de mise à jour
  initialUpdateCommands?: string[] // Commandes de mise à jour
}
```

### 3.3 État du REPL

**État React** (via `useState`) :

```typescript
// Contrôle
const [abortController, setAbortController] = useState<AbortController | null>(null)
const [isLoading, setIsLoading] = useState(false)
const [forkNumber, setForkNumber] = useState(initialForkNumber)

// Messages
const [messages, setMessages] = useState<MessageType[]>(initialMessages ?? [])

// Input
const [inputValue, setInputValue] = useState('')
const [inputMode, setInputMode] = useState<'bash' | 'prompt' | 'koding'>('prompt')
const [submitCount, setSubmitCount] = useState(0)

// UI State
const [toolJSX, setToolJSX] = useState<{
  jsx: React.ReactNode | null
  shouldHidePromptInput: boolean
} | null>(null)
const [toolUseConfirm, setToolUseConfirm] = useState<ToolUseConfirm | null>(null)
const [isMessageSelectorVisible, setIsMessageSelectorVisible] = useState(false)
const [showCostDialog, setShowCostDialog] = useState(false)
const [binaryFeedbackContext, setBinaryFeedbackContext] = useState<BinaryFeedbackContext | null>(null)

// Autres
const readFileTimestamps = useRef<{ [filename: string]: number }>({})
```

### 3.4 Hooks Utilisés

| Hook | Usage |
|------|-------|
| `useState` | Gestion état local |
| `useEffect` | Effets de bord, sync |
| `useRef` | Références mutables |
| `useCallback` | Mémoïsation callbacks |
| `useMemo` | Mémoïsation valeurs |
| `useApiKeyVerification` | Vérification clé API |
| `useCancelRequest` | Annulation requête (Ctrl+C) |
| `useCanUseTool` | Vérification permissions |
| `useLogMessages` | Logging des messages |
| `useLogStartupTime` | Log temps de démarrage |
| `useCostSummary` | Résumé des coûts |

### 3.5 Cycle de Vie

```
┌────────────────────────────────────────────┐
│                                            │
│  1. MONTAGE                                │
│     ┌────────────────────────────┐         │
│     │ REPL() appelée             │         │
│     │ État initialisé            │         │
│     │ Hooks configurés           │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  2. INITIALISATION                         │
│     ┌────────────────────────────┐         │
│     │ onInit() si initialPrompt  │         │
│     │ API key verification       │         │
│     │ Context loading            │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  3. RENDU INITIAL                          │
│     ┌────────────────────────────┐         │
│     │ Logo affiché               │         │
│     │ Messages vides             │         │
│     │ PromptInput prêt           │         │
│     └────────────────────────────┘         │
│              ↓                             │
│  4. BOUCLE INTERACTIVE                     │
│     ┌────────────────────────────┐         │
│     │ User input                 │         │
│     │ → handleSubmit()           │         │
│     │ → query()                  │         │
│     │ → Streaming updates        │         │
│     │ → Re-render                │         │
│     └────────────────────────────┘         │
│              ↓ (loop)                      │
│  5. DÉMONTAGE (Ctrl+D)                     │
│     ┌────────────────────────────┐         │
│     │ Cleanup abortController    │         │
│     │ Save state                 │         │
│     │ Exit process               │         │
│     └────────────────────────────┘         │
│                                            │
└────────────────────────────────────────────┘
```

---

## 4. Les 3 Modes d'Interaction

Kode propose **3 modes** d'interaction, switchables avec **Shift+Tab**.

### 4.1 Mode Prompt (Défaut)

**Description** : Conversation normale avec l'agent IA

**Icon** : `💬`

**Couleur** : Bleu

**Comportement** :
- Input envoyé directement à l'agent
- Historique de commandes (↑ / ↓)
- Autocomplétion intelligente
- Support images (copier-coller)
- Support pasted text

**Exemple** :
```
┌─────────────────────────────────────────┐
│ 💬 Prompt Mode                          │
│ Type your message to the AI assistant   │
│                                         │
│ Press Shift+Tab to cycle modes          │
└─────────────────────────────────────────┘

> Help me refactor this function
```

### 4.2 Mode Bash

**Description** : Exécution directe de commandes shell

**Icon** : `$`

**Couleur** : Vert

**Comportement** :
- Input exécuté comme commande Bash
- Pas d'interprétation AI
- Historique shell séparé
- Autocomplétion fichiers
- Permissions BashTool appliquées

**Exemple** :
```
┌─────────────────────────────────────────┐
│ $ Bash Mode                             │
│ Execute shell commands directly         │
│                                         │
│ Press Shift+Tab to cycle modes          │
└─────────────────────────────────────────┘

$ ls -la src/
```

**Implémentation** :
```typescript
if (inputMode === 'bash') {
  // Wrap dans BashTool
  const bashInput = `Execute: ${input}`
  // Envoie à l'agent avec instruction d'utiliser BashTool
}
```

### 4.3 Mode Koding

**Description** : Prise de notes rapide dans KODING.md

**Icon** : `#`

**Couleur** : Jaune

**Comportement** :
- Notes ajoutées automatiquement à `KODING.md`
- Interprétation AI du contenu (structuration)
- Timestamp automatique
- Format Markdown
- Pas de requête AI complète

**Exemple** :
```
┌─────────────────────────────────────────┐
│ # Koding Mode                           │
│ Quick notes saved to KODING.md          │
│                                         │
│ Press Shift+Tab to cycle modes          │
└─────────────────────────────────────────┘

# Remember to update the API docs after refactoring
```

**Fonctionnement** :
```typescript
// 1. Input utilisateur
const note = "Remember to update API docs"

// 2. Interprétation AI (optionnelle)
const interpretedNote = await interpretHashCommand(note)
// → "# API Documentation Update\n\nRemember to update..."

// 3. Ajout à KODING.md
await handleHashCommand(interpretedNote, getCwd())
```

### 4.4 Switching de Mode

**Raccourci** : `Shift + Tab`

**Code** :
```typescript
useInput((input, key) => {
  if (key.shift && key.tab) {
    // Cycle: prompt → bash → koding → prompt
    const modes: Array<'prompt' | 'bash' | 'koding'> = ['prompt', 'bash', 'koding']
    const currentIndex = modes.indexOf(inputMode)
    const nextIndex = (currentIndex + 1) % modes.length
    setInputMode(modes[nextIndex])
  }
})
```

**Indicateur visuel** :
```typescript
// src/components/ModeIndicator.tsx
export function ModeIndicator() {
  const { currentMode, getModeConfig } = usePermissionContext()
  const modeConfig = getModeConfig()

  return (
    <Box borderStyle="single" padding={1}>
      <Text color={modeConfig.color} bold>
        {modeConfig.icon} {modeConfig.label}
      </Text>
      <Text color="gray">
        {modeConfig.description}
      </Text>
      <Text color="gray">
        Press Shift+Tab to cycle modes
      </Text>
    </Box>
  )
}
```

---

## 5. Composants Majeurs

### 5.1 Logo

**Fichier** : `src/components/Logo.tsx` (150 lignes)

**Responsabilité** : Bannière de démarrage avec infos système

**Affichage** :
```
┌───────────────────────────────────────────────┐
│ ✻ Welcome to Kode research preview!          │
│                                               │
│   /help for help                              │
│   cwd: /home/user/project                     │
│                                               │
│   Model: claude-sonnet-4                      │
│   MCP Servers: 2 connected                    │
│                                               │
│   New version available: 0.2.0 (current: 0.1.0)│
│   Run: npm install -g @shareai-lab/kode@latest│
└───────────────────────────────────────────────┘
```

**Props** :
```typescript
{
  mcpClients: WrappedClient[]
  isDefaultModel?: boolean
  updateBannerVersion?: string | null
  updateBannerCommands?: string[] | null
}
```

**Logique** :
- Détecte overrides via env vars
- Affiche modèle actuel
- Liste MCP servers connectés
- Bannière de mise à jour si disponible
- Adapte largeur au cwd

---

### 5.2 PromptInput

**Fichier** : `src/components/PromptInput.tsx` (600+ lignes)

**Responsabilité** : Saisie utilisateur avec autocomplétion

**Fonctionnalités** :

#### 5.2.1 Autocomplétion Unifiée

**Hook** : `useUnifiedCompletion`

**Sources d'autocomplétion** :
1. **Commandes** (`/help`, `/model`, etc.)
2. **Fichiers** (chemin relatifs)
3. **Agents** (types d'agents disponibles)
4. **Modèles** (noms de modèles configurés)

**Déclenchement** : `Tab` ou automatique après `/`

**Exemple** :
```
User types: /mod<Tab>
          → /model

User types: src/c<Tab>
          → src/components/

User types: --model=clau<Tab>
          → --model=claude-sonnet-4
```

#### 5.2.2 Historique de Commandes

**Hook** : `useArrowKeyHistory`

**Raccourcis** :
- `↑` : Commande précédente
- `↓` : Commande suivante
- Stockage persistant dans `~/.kode/history`

**Code** :
```typescript
const { historyIndex, setHistoryIndex, getHistoryAtIndex } = useArrowKeyHistory()

useInput((input, key) => {
  if (key.upArrow) {
    const previousCommand = getHistoryAtIndex(historyIndex - 1)
    if (previousCommand) {
      setInputValue(previousCommand)
      setHistoryIndex(historyIndex - 1)
    }
  }
  // Similar for downArrow
})
```

#### 5.2.3 Support Image

**Capacité** : Copier-coller d'images dans le prompt

**Formats** : PNG, JPG, GIF, WebP

**Mécanisme** :
```typescript
// Détection clipboard image
const [pastedImage, setPastedImage] = useState<string | null>(null)

// Encodage base64
const imageBase64 = await readImageAsBase64(imagePath)

// Ajout au message
const imageBlock: ImageBlockParam = {
  type: 'image',
  source: {
    type: 'base64',
    media_type: 'image/png',
    data: imageBase64
  }
}
```

**Affichage** :
```
> [Image: screenshot.png (1024x768)] Help me debug this UI issue
```

#### 5.2.4 Token Warning

**Seuil** : 20,000 tokens

**Affichage** :
```
┌───────────────────────────────────────────┐
│ ⚠️  Warning: Input is large (23,456 tokens)│
│                                           │
│ This may consume significant context.     │
│ Consider splitting into smaller chunks.   │
└───────────────────────────────────────────┘
```

**Composant** : `TokenWarning.tsx`

```typescript
export const WARNING_THRESHOLD = 20_000

export function TokenWarning({ tokenCount }: { tokenCount: number }) {
  if (tokenCount < WARNING_THRESHOLD) return null

  return (
    <Box borderStyle="single" borderColor="yellow" padding={1}>
      <Text color="yellow">
        ⚠️  Warning: Input is large ({formatNumber(tokenCount)} tokens)
      </Text>
    </Box>
  )
}
```

---

### 5.3 Message

**Fichier** : `src/components/Message.tsx` (200+ lignes)

**Responsabilité** : Rendu d'un message (User ou Assistant)

**Architecture** :
```typescript
export function Message({ message, ... }: Props) {
  if (message.type === 'assistant') {
    // Rendu message assistant
    return (
      <Box flexDirection="column">
        {message.message.content.map((block, index) => (
          <AssistantMessage
            key={index}
            param={block}
            tools={tools}
            verbose={verbose}
            {...other}
          />
        ))}
      </Box>
    )
  }

  // Rendu message utilisateur
  return (
    <Box flexDirection="column">
      {content.map((block, index) => (
        <UserMessage
          key={index}
          message={message}
          param={block}
          {...other}
        />
      ))}
    </Box>
  )
}
```

**Types de blocs** :
- **Assistant** : Text, ToolUse, Thinking
- **User** : Text, Image, ToolResult

---

### 5.4 Composants de Messages

#### 5.4.1 Messages Assistant

**AssistantTextMessage** :
```
  Claude says:
  The function can be refactored using...
```

**AssistantToolUseMessage** :
```
  ⎿ Bash: npm test

    Running tests...
```

**AssistantThinkingMessage** :
```
  💭 [Thinking for 2.3s]

  Let me analyze the requirements...
  I should start by checking the current implementation...
```

**AssistantRedactedThinkingMessage** :
```
  💭 [Thinking redacted - enable verbose mode to see]
```

#### 5.4.2 Messages Utilisateur

**UserTextMessage** :
```
You said:
Help me refactor this function
```

**UserCommandMessage** :
```
You ran: /model claude-opus-4
```

**UserBashInputMessage** :
```
$ ls -la src/
```

**UserKodingInputMessage** :
```
# Note added to KODING.md:
Remember to update API docs
```

**UserToolResultMessage** :
```
  ⎿ Result: Tests passed (42/42)

    Duration: 2.3s
    Coverage: 87%
```

---

### 5.5 PermissionRequest

**Fichier** : `src/components/permissions/PermissionRequest.tsx`

**Responsabilité** : Demande de permission utilisateur

**Variantes** :
- `BashPermissionRequest` : Permissions commandes
- `FileEditPermissionRequest` : Permissions édition
- `FileWritePermissionRequest` : Permissions écriture
- `FilesystemPermissionRequest` : Permissions générales
- `FallbackPermissionRequest` : Permissions génériques

**Exemple (BashPermissionRequest)** :
```
┌───────────────────────────────────────────────┐
│ Kode wants to run:                            │
│                                               │
│   $ npm install lodash                        │
│                                               │
│ Options:                                      │
│   [A] Allow once                              │
│   [P] Allow all npm:* commands (project)      │
│   [D] Deny                                    │
│                                               │
│ Your choice:                                  │
└───────────────────────────────────────────────┘
```

**Interface** :
```typescript
export type ToolUseConfirm = {
  tool: Tool
  input: any
  onApprove: (prefix: string | null) => void
  onDeny: () => void
  onAbort: () => void
  assistantMessage: AssistantMessage
}
```

**Workflow** :
```
1. Agent veut utiliser outil
   ↓
2. hasPermissionsToUseTool() → false
   ↓
3. setToolUseConfirm({ tool, input, ... })
   ↓
4. PermissionRequest affiché
   ↓
5. User choix (A/P/D)
   ↓
6. onApprove() ou onDeny()
   ↓
7. Permission sauvegardée (si P)
   ↓
8. Outil exécuté ou rejeté
```

---

### 5.6 BinaryFeedback

**Fichier** : `src/components/binary-feedback/BinaryFeedback.tsx`

**Responsabilité** : Choix entre 2 options (A/B testing de réponses)

**Usage** : Agent génère 2 réponses, utilisateur choisit la meilleure

**Affichage** :
```
┌───────────────────────────────────────────────┐
│ Which response do you prefer?                 │
│                                               │
│ [1] Response A                                │
│     The function can be refactored using...   │
│                                               │
│ [2] Response B                                │
│     I suggest restructuring the function...   │
│                                               │
│ Press 1 or 2 to choose, Esc to skip           │
└───────────────────────────────────────────────┘
```

**Context** :
```typescript
export type BinaryFeedbackContext = {
  m1: AssistantMessage        // Option A
  m2: AssistantMessage        // Option B
  resolve: (result: BinaryFeedbackResult) => void
}

export type BinaryFeedbackResult =
  | { choice: 'm1' | 'm2' }
  | { choice: 'skip' }
```

**Workflow** :
```typescript
// 1. Agent demande binary feedback
const result = await getBinaryFeedbackResponse(messageA, messageB)

// 2. UI affiche les options
setBinaryFeedbackContext({ m1: messageA, m2: messageB, resolve })

// 3. User choisit
useInput((input, key) => {
  if (input === '1') {
    binaryFeedbackContext.resolve({ choice: 'm1' })
  } else if (input === '2') {
    binaryFeedbackContext.resolve({ choice: 'm2' })
  } else if (key.escape) {
    binaryFeedbackContext.resolve({ choice: 'skip' })
  }
})

// 4. Résultat retourné à l'agent
```

---

### 5.7 MessageSelector

**Fichier** : `src/components/MessageSelector.tsx`

**Responsabilité** : Sélection d'un message dans l'historique

**Déclenchement** : `Ctrl+M`

**Affichage** :
```
┌───────────────────────────────────────────────┐
│ Select a message to fork from:                │
│                                               │
│   1. [You] Help me refactor...                │
│ → 2. [Assistant] The function can be...       │
│   3. [You] Now add tests                      │
│   4. [Assistant] Here are the tests...        │
│                                               │
│ Use ↑↓ to navigate, Enter to select, Esc to cancel│
└───────────────────────────────────────────────┘
```

**Usage** : Fork conversation à partir d'un message spécifique

**Workflow** :
```
1. User: Ctrl+M
   ↓
2. MessageSelector affiché
   ↓
3. User: ↑↓ pour naviguer
   ↓
4. User: Enter pour sélectionner
   ↓
5. Conversation forkée
   ↓
6. Nouveau fork number
   ↓
7. Messages tronqués jusqu'au message sélectionné
```

---

### 5.8 ModelSelector

**Fichier** : `src/components/ModelSelector.tsx`

**Responsabilité** : Sélection interactive de modèle

**Déclenchement** : `/model` sans argument

**Affichage** :
```
┌───────────────────────────────────────────────┐
│ Select a model:                                │
│                                               │
│ → claude-sonnet-4                 [ACTIVE]    │
│   claude-opus-4                               │
│   gpt-4-turbo                                 │
│   gpt-4o                                      │
│   deepseek-chat                               │
│                                               │
│ Use ↑↓ to navigate, Enter to select, Esc to cancel│
└───────────────────────────────────────────────┘
```

**Features** :
- Liste tous les modèles configurés
- Indique le modèle actif
- Affiche les model pointers
- Permet sélection rapide

---

### 5.9 Cost Components

#### CostThresholdDialog

**Fichier** : `src/components/CostThresholdDialog.tsx`

**Trigger** : Coût total > $5

**Affichage** :
```
┌───────────────────────────────────────────────┐
│ ⚠️  Cost Alert                                 │
│                                               │
│ You've spent over $5.00 on API calls.        │
│                                               │
│ Current total: $5.47                          │
│                                               │
│ Please review your usage.                     │
│                                               │
│ Press Enter to acknowledge                    │
└───────────────────────────────────────────────┘
```

#### Cost Display

**Composant** : `Cost.tsx`

**Affichage** : Inline avec chaque message

```
  ⎿ Result: Tests passed       $0.02 | 1.2s
```

**Code** :
```typescript
export function Cost({ costUSD, durationMs, debug }: Props) {
  if (!debug && costUSD === 0) return null

  return (
    <Box justifyContent="flex-end">
      {costUSD > 0 && (
        <Text color="gray">
          ${costUSD.toFixed(4)}
        </Text>
      )}
      {durationMs > 0 && (
        <Text color="gray" dimColor>
          {' | '}{formatDuration(durationMs)}
        </Text>
      )}
    </Box>
  )
}
```

---

## 6. Système de Rendu

### 6.1 Rendu Streaming

Kode utilise **Static** (Ink) pour historique immuable et rendu live en dessous :

```typescript
<Box flexDirection="column">
  {/* Historique (immuable) */}
  <Static items={completedMessages}>
    {message => (
      <Message message={message} shouldAnimate={false} />
    )}
  </Static>

  {/* Message en cours (streaming) */}
  {currentMessage && (
    <Message message={currentMessage} shouldAnimate={true} />
  )}

  {/* Outil actif */}
  {toolJSX?.jsx}

  {/* Input */}
  <PromptInput ... />
</Box>
```

**Avantages** :
- ✅ Pas de re-render de tout l'historique
- ✅ Performances optimales
- ✅ Scrolling natif
- ✅ Animation seulement sur message actif

### 6.2 Rendu Conditionnel

Pattern React standard pour affichage conditionnel :

```typescript
{/* Permission request */}
{toolUseConfirm && (
  <PermissionRequest
    toolUseConfirm={toolUseConfirm}
    onApprove={handleApprove}
    onDeny={handleDeny}
  />
)}

{/* Binary feedback */}
{binaryFeedbackContext && (
  <BinaryFeedback
    context={binaryFeedbackContext}
    onResolve={handleResolve}
  />
)}

{/* Message selector */}
{isMessageSelectorVisible && (
  <MessageSelector
    messages={messages}
    onSelect={handleSelect}
    onCancel={() => setIsMessageSelectorVisible(false)}
  />
)}

{/* Cost dialog */}
{showCostDialog && (
  <CostThresholdDialog
    onAcknowledge={() => {
      setShowCostDialog(false)
      setHaveShownCostDialog(true)
    }}
  />
)}
```

### 6.3 Coloration Syntaxique

**Composant** : `HighlightedCode.tsx`

**Usage** : Code blocks avec coloration

**Librairie** : `highlight.js` + `chalk`

**Exemple** :
```typescript
<HighlightedCode
  language="typescript"
  code={codeContent}
  theme={getTheme()}
/>
```

**Rendu** :
```typescript
function example() {  // violet
  const x = 42        // bleu
  return x * 2        // vert
}
```

### 6.4 Diffs Structurés

**Composant** : `StructuredDiff.tsx`

**Usage** : Affichage de modifications de fichiers

**Format** :
```
┌─ Lines 45-52 ─────────────────────────
│ - const theme = 'light'
│ + const theme = 'dark'
│
│ - fontSize: 12
│ + fontSize: 14
└────────────────────────────────────────
```

**Code** :
```typescript
export function StructuredDiff({ patch }: { patch: Hunk[] }) {
  return (
    <Box flexDirection="column">
      {patch.map((hunk, index) => (
        <Box key={index} borderStyle="single" padding={1}>
          <Text bold>
            Lines {hunk.oldStart}-{hunk.oldStart + hunk.oldLines}
          </Text>
          <Box flexDirection="column">
            {hunk.lines.map((line, i) => (
              <Text key={i} color={getLineColor(line)}>
                {line}
              </Text>
            ))}
          </Box>
        </Box>
      ))}
    </Box>
  )
}

function getLineColor(line: string): string {
  if (line.startsWith('+')) return 'green'
  if (line.startsWith('-')) return 'red'
  return 'gray'
}
```

---

## 7. Interactions Utilisateur

### 7.1 Raccourcis Clavier

| Raccourci | Action |
|-----------|--------|
| **Enter** | Envoyer le prompt |
| **Shift + Enter** | Nouvelle ligne (si configuré) |
| **Ctrl + C** | Annuler requête en cours |
| **Ctrl + D** | Quitter Kode |
| **Ctrl + L** | Effacer le terminal |
| **Ctrl + M** | Ouvrir MessageSelector |
| **Shift + Tab** | Changer de mode (Prompt/Bash/Koding) |
| **Tab** | Autocomplétion |
| **↑** | Commande précédente (historique) |
| **↓** | Commande suivante (historique) |
| **Esc** | Annuler/Fermer dialogs |

### 7.2 Gestion du Clavier

**Hook** : `useInput` (Ink)

```typescript
useInput((input, key) => {
  // Ctrl+C - Cancel
  if (key.ctrl && input === 'c') {
    onCancel()
    return
  }

  // Ctrl+D - Exit
  if (key.ctrl && input === 'd') {
    process.exit(0)
  }

  // Ctrl+L - Clear
  if (key.ctrl && input === 'l') {
    clearTerminal()
    return
  }

  // Ctrl+M - Message selector
  if (key.ctrl && input === 'm') {
    setIsMessageSelectorVisible(true)
    return
  }

  // Shift+Tab - Mode switch
  if (key.shift && key.tab) {
    cycleMode()
    return
  }

  // Esc - Close dialogs
  if (key.escape) {
    setToolUseConfirm(null)
    setBinaryFeedbackContext(null)
    setIsMessageSelectorVisible(false)
    return
  }
})
```

### 7.3 Copier-Coller

**Texte** :
- Ctrl+Shift+V : Paste
- Détection de grandes pastes
- Prompt affiché : `[Pasted text +42 lines]`

**Images** :
- Support clipboard images
- Conversion base64 automatique
- Affichage preview : `[Image: screenshot.png (1024x768)]`
- Envoi dans le message comme `ImageBlockParam`

### 7.4 Autocomplétion

**Trigger** : `Tab` ou automatique après `/`

**Sources** :
```typescript
const completions = [
  ...commandCompletions,    // /help, /model, etc.
  ...fileCompletions,       // src/components/
  ...agentCompletions,      // --subagent-type=test-writer
  ...modelCompletions       // --model=claude-sonnet-4
]
```

**Algorithme** :
1. Détecte le contexte (commande, fichier, flag)
2. Filtre les completions pertinentes
3. Applique fuzzy matching
4. Affiche top 5 matches
5. Tab pour sélectionner

**Exemple** :
```
User types: /mo
Completions:
  → /model
    /move
    /monitor

Press Tab to complete
```

---

## 8. Gestion d'État

### 8.1 État Local (useState)

**Messages** :
```typescript
const [messages, setMessages] = useState<MessageType[]>([])

// Ajout message
setMessages(prev => [...prev, newMessage])

// Mise à jour message (streaming)
setMessages(prev => {
  const copy = [...prev]
  copy[copy.length - 1] = updatedMessage
  return copy
})
```

**Input** :
```typescript
const [inputValue, setInputValue] = useState('')
const [inputMode, setInputMode] = useState<'prompt' | 'bash' | 'koding'>('prompt')
```

**UI State** :
```typescript
const [isLoading, setIsLoading] = useState(false)
const [toolJSX, setToolJSX] = useState<{ jsx: ReactNode } | null>(null)
const [toolUseConfirm, setToolUseConfirm] = useState<ToolUseConfirm | null>(null)
```

### 8.2 État Global (Context API)

**PermissionContext** :
```typescript
const PermissionContext = createContext<PermissionContextType>({
  currentMode: 'default',
  permissionContext: defaultPermissionContext,
  cycleMode: () => {},
  getModeConfig: () => defaultModeConfig
})

export function PermissionProvider({ children }) {
  const [currentMode, setCurrentMode] = useState('default')
  const [permissionContext, setPermissionContext] = useState(defaultContext)

  const cycleMode = () => {
    // Logic to cycle modes
  }

  return (
    <PermissionContext.Provider value={{ currentMode, permissionContext, cycleMode, getModeConfig }}>
      {children}
    </PermissionContext.Provider>
  )
}
```

**Usage** :
```typescript
const { currentMode, cycleMode } = usePermissionContext()
```

### 8.3 État Réactif (Refs)

**readFileTimestamps** :
```typescript
const readFileTimestamps = useRef<{ [filename: string]: number }>({})

// Mise à jour (mutable, pas de re-render)
readFileTimestamps.current[filePath] = Date.now()
```

**Pourquoi useRef** :
- Mutation directe sans re-render
- Persistance entre renders
- Évite re-création d'objets

---

## 9. Catalogue des Composants

### 9.1 Composants d'Affichage

| Composant | Fichier | Responsabilité |
|-----------|---------|----------------|
| **Logo** | `Logo.tsx` | Bannière de démarrage |
| **AsciiLogo** | `AsciiLogo.tsx` | Logo ASCII art |
| **Message** | `Message.tsx` | Rendu d'un message |
| **MessageResponse** | `MessageResponse.tsx` | Réponse complète de l'assistant |
| **Spinner** | `Spinner.tsx` | Indicateur de chargement |
| **Cost** | `Cost.tsx` | Affichage du coût |
| **HighlightedCode** | `HighlightedCode.tsx` | Coloration syntaxique |
| **StructuredDiff** | `StructuredDiff.tsx` | Diffs de fichiers |
| **ModeIndicator** | `ModeIndicator.tsx` | Indicateur de mode |
| **TodoItem** | `TodoItem.tsx` | Item de todo list |

### 9.2 Composants d'Interaction

| Composant | Fichier | Responsabilité |
|-----------|---------|----------------|
| **PromptInput** | `PromptInput.tsx` | Saisie utilisateur |
| **TextInput** | `TextInput.tsx` | Input texte de base |
| **MessageSelector** | `MessageSelector.tsx` | Sélection de message |
| **ModelSelector** | `ModelSelector.tsx` | Sélection de modèle |
| **CustomSelect** | `CustomSelect/select.tsx` | Select personnalisé |
| **BinaryFeedback** | `binary-feedback/BinaryFeedback.tsx` | Choix A/B |

### 9.3 Composants de Permissions

| Composant | Fichier | Responsabilité |
|-----------|---------|----------------|
| **PermissionRequest** | `permissions/PermissionRequest.tsx` | Demande générique |
| **BashPermissionRequest** | `permissions/BashPermissionRequest.tsx` | Permissions Bash |
| **FileEditPermissionRequest** | `permissions/FileEditPermissionRequest.tsx` | Permissions édition |
| **FileWritePermissionRequest** | `permissions/FileWritePermissionRequest.tsx` | Permissions écriture |
| **FilesystemPermissionRequest** | `permissions/FilesystemPermissionRequest.tsx` | Permissions FS |
| **FallbackPermissionRequest** | `permissions/FallbackPermissionRequest.tsx` | Fallback générique |

### 9.4 Composants de Messages

| Composant | Fichier | Type |
|-----------|---------|------|
| **AssistantTextMessage** | `messages/AssistantTextMessage.tsx` | Texte assistant |
| **AssistantToolUseMessage** | `messages/AssistantToolUseMessage.tsx` | Utilisation outil |
| **AssistantThinkingMessage** | `messages/AssistantThinkingMessage.tsx` | Réflexion |
| **AssistantRedactedThinkingMessage** | `messages/AssistantRedactedThinkingMessage.tsx` | Réflexion redacted |
| **AssistantBashOutputMessage** | `messages/AssistantBashOutputMessage.tsx` | Sortie Bash |
| **AssistantLocalCommandOutputMessage** | `messages/AssistantLocalCommandOutputMessage.tsx` | Sortie commande |
| **UserTextMessage** | `messages/UserTextMessage.tsx` | Texte utilisateur |
| **UserCommandMessage** | `messages/UserCommandMessage.tsx` | Commande utilisateur |
| **UserBashInputMessage** | `messages/UserBashInputMessage.tsx` | Input Bash |
| **UserKodingInputMessage** | `messages/UserKodingInputMessage.tsx` | Input Koding |
| **UserPromptMessage** | `messages/UserPromptMessage.tsx` | Prompt utilisateur |
| **UserToolResultMessage** | `messages/UserToolResultMessage/UserToolResultMessage.tsx` | Résultat outil |
| **UserToolSuccessMessage** | `messages/UserToolResultMessage/UserToolSuccessMessage.tsx` | Succès outil |
| **UserToolErrorMessage** | `messages/UserToolResultMessage/UserToolErrorMessage.tsx` | Erreur outil |
| **UserToolRejectMessage** | `messages/UserToolResultMessage/UserToolRejectMessage.tsx` | Rejet outil |
| **UserToolCanceledMessage** | `messages/UserToolResultMessage/UserToolCanceledMessage.tsx` | Annulation outil |
| **TaskToolMessage** | `messages/TaskToolMessage.tsx` | Message TaskTool |
| **TaskProgressMessage** | `messages/TaskProgressMessage.tsx` | Progrès TaskTool |

### 9.5 Composants de Dialogs

| Composant | Fichier | Responsabilité |
|-----------|---------|----------------|
| **CostThresholdDialog** | `CostThresholdDialog.tsx` | Alerte de coût |
| **TrustDialog** | `TrustDialog.tsx` | Confirmation de confiance |
| **InvalidConfigDialog** | `InvalidConfigDialog.tsx` | Configuration invalide |
| **MCPServerApprovalDialog** | `MCPServerApprovalDialog.tsx` | Approbation serveur MCP |
| **MCPServerMultiselectDialog** | `MCPServerMultiselectDialog.tsx` | Sélection multi MCP |
| **ProjectOnboarding** | `ProjectOnboarding.tsx` | Onboarding projet |
| **Onboarding** | `Onboarding.tsx` | Onboarding général |

### 9.6 Composants Utilitaires

| Composant | Fichier | Responsabilité |
|-----------|---------|----------------|
| **TokenWarning** | `TokenWarning.tsx` | Avertissement tokens |
| **ToolUseLoader** | `ToolUseLoader.tsx` | Loader outil |
| **PressEnterToContinue** | `PressEnterToContinue.tsx` | Pause interactive |
| **Help** | `Help.tsx` | Affichage aide |
| **Config** | `Config.tsx` | Affichage config |
| **Bug** | `Bug.tsx` | Rapport de bug |
| **Link** | `Link.tsx` | Lien cliquable |
| **ModelConfig** | `ModelConfig.tsx` | Config modèle |
| **ModelListManager** | `ModelListManager.tsx` | Gestion liste modèles |
| **ModelStatusDisplay** | `ModelStatusDisplay.tsx` | Statut modèle |
| **LogSelector** | `LogSelector.tsx` | Sélection de logs |
| **ConsoleOAuthFlow** | `ConsoleOAuthFlow.tsx` | Flux OAuth |
| **FileEditToolUpdatedMessage** | `FileEditToolUpdatedMessage.tsx` | Message édition fichier |
| **FallbackToolUseRejectedMessage** | `FallbackToolUseRejectedMessage.tsx` | Message rejet outil |

### 9.7 Hooks Personnalisés

| Hook | Fichier | Usage |
|------|---------|-------|
| **useApiKeyVerification** | `hooks/useApiKeyVerification.ts` | Vérification clé API |
| **useArrowKeyHistory** | `hooks/useArrowKeyHistory.ts` | Historique commandes |
| **useCanUseTool** | `hooks/useCanUseTool.ts` | Vérification permissions |
| **useCancelRequest** | `hooks/useCancelRequest.ts` | Annulation requête |
| **useCostSummary** | `hooks/useCostSummary.ts` | Résumé coûts |
| **useLogMessages** | `hooks/useLogMessages.ts` | Logging messages |
| **useLogStartupTime** | `hooks/useLogStartupTime.ts` | Log temps démarrage |
| **useTerminalSize** | `hooks/useTerminalSize.ts` | Taille du terminal |
| **useUnifiedCompletion** | `hooks/useUnifiedCompletion.ts` | Autocomplétion |

---

## Annexes

### A. Fichiers Clés

| Fichier | Lignes | Description |
|---------|--------|-------------|
| `src/screens/REPL.tsx` | 809 | Composant principal |
| `src/components/PromptInput.tsx` | 600+ | Saisie utilisateur |
| `src/components/Message.tsx` | 200+ | Rendu messages |
| `src/components/Logo.tsx` | 150+ | Bannière |
| `src/components/ModeIndicator.tsx` | 89 | Indicateur mode |
| `src/components/permissions/PermissionRequest.tsx` | 300+ | Permissions |

### B. Technologies UI

| Technologie | Version | Usage |
|-------------|---------|-------|
| **React** | 18.x | Framework UI |
| **Ink** | 4.x | React renderer pour terminal |
| **chalk** | 5.x | Couleurs terminal |
| **boxen** | 7.x | Boîtes dans terminal |
| **highlight.js** | 11.x | Coloration syntaxique |
| **TypeScript** | 5.x | Type safety |

### C. Patterns React Utilisés

| Pattern | Usage dans Kode |
|---------|-----------------|
| **Hooks** | useState, useEffect, useRef, useCallback, useMemo |
| **Context API** | PermissionContext pour state global |
| **Composition** | Composants imbriqués (Message > AssistantMessage > ToolUse) |
| **Render Props** | Static items rendering |
| **Conditional Rendering** | Dialogs, permissions, modes |
| **Controlled Components** | PromptInput avec inputValue |
| **Custom Hooks** | 15+ hooks personnalisés |

---

**Navigation** :
- ← [Système d'Outils](./05-systeme-outils.md)
- → [Configuration](./07-configuration.md) *(à créer)*
- ↑ [Index](./INDEX.md)
- ⌂ [README](./README.md)

---

**Dernière mise à jour** : 16 janvier 2025
**Version de Kode** : 0.1.0
