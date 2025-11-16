# Guide de l'Utilisateur Kode

> Guide pratique pour bien démarrer et maîtriser Kode au quotidien

[← Retour à l'index](./README.md)

---

## Table des Matières

1. [Installation et Premier Démarrage](#1-installation-et-premier-démarrage)
2. [Interface et Navigation](#2-interface-et-navigation)
3. [Utilisation Quotidienne](#3-utilisation-quotidienne)
4. [Configuration des Modèles](#4-configuration-des-modèles)
5. [Création et Utilisation d'Agents](#5-création-et-utilisation-dagents)
6. [Astuces et Bonnes Pratiques](#6-astuces-et-bonnes-pratiques)
7. [Dépannage](#7-dépannage)

---

## 1. Installation et Premier Démarrage

### Installation

**Via npm** :
```bash
npm install -g kode-cli
```

**Via bun** (recommandé) :
```bash
bun install -g kode-cli
```

**Depuis les sources** :
```bash
git clone https://github.com/Pipikeen/Kode
cd Kode
bun install
bun run build
bun link
```

### Vérification

```bash
kode --version
# → kode version 0.1.0

kode --help
# Affiche l'aide
```

### Premier Lancement

```bash
kode
```

**Ce qui se passe** :
1. Kode crée `~/.kode/` pour la configuration globale
2. Vous verrez l'écran d'accueil
3. Si aucun modèle n'est configuré, vous serez guidé pour en ajouter un

### Configuration Initiale du Modèle

Au premier lancement :

```
┌───────────────────────────────────────────┐
│ ✻ Welcome to Kode!                        │
│   No model configured yet.                │
│   Type /model to add a model              │
└───────────────────────────────────────────┘

>
```

**Étapes** :

1. Tapez `/model`
2. Choisissez "Add new model"
3. Sélectionnez un provider (OpenAI, Anthropic, DeepSeek, etc.)
4. Entrez votre clé API
5. Choisissez le modèle spécifique
6. Configurez les paramètres (optionnel)
7. Le modèle est maintenant actif !

**Exemple avec Claude** :

```
> /model

[Interface s'affiche]

Provider: Anthropic
API Key: sk-ant-api03-... [votre clé]
Model: claude-sonnet-4-20250514
Name (optional): Claude Main

✅ Model added successfully!
```

---

## 2. Interface et Navigation

### L'Interface REPL

**REPL** = Read-Eval-Print Loop (boucle interactive)

```
┌───────────────────────────────────────────┐
│ ✻ Kode Research Preview                  │
│   Model: claude-sonnet-4                  │
│   Directory: /home/user/my-project        │
└───────────────────────────────────────────┘

[Messages précédents affichés ici]

> _  [curseur]
```

### Les Trois Modes d'Interaction

#### 1. Mode Prompt (par défaut)

**Préfixe** : `>` (vert)

**Usage** : Communication avec l'IA

```
> Explique-moi comment fonctionne ce code
> Crée une fonction pour parser du JSON
> Analyse les performances de cette fonction
```

**Raccourci** : Aucun, c'est le mode par défaut

#### 2. Mode Bash

**Préfixe** : `!` (rouge/orange)

**Usage** : Exécution de commandes shell directes

```
! git status
! npm test
! ls -la src/
```

**Raccourci** : Tapez `!` au début de votre commande

**Note** : Commandes sûres exécutées sans permission :
- `git status`, `git diff`, `git log`
- `pwd`, `ls`, `tree`
- `npm list`, etc.

#### 3. Mode Koding

**Préfixe** : `#` (jaune)

**Usage** : Annotations et notes pour documentation

```
# Architecture: utilise le pattern MVC
# TODO: ajouter la validation des entrées
# Note: cette fonction est appelée depuis 3 endroits
```

**Raccourci** : Tapez `#` au début

**Utilité** : Ces notes sont capturées et peuvent être formatées automatiquement pour `AGENTS.md`

### Raccourcis Clavier

| Raccourci | Action |
|-----------|--------|
| `↑` / `↓` | Naviguer dans l'historique |
| `Tab` | Autocomplétion |
| `→` | Accepter suggestion |
| `Esc` | Sélecteur de messages / Annuler |
| `Shift+M` | Changer de modèle rapide |
| `Shift+Tab` | Cycle modes de permission |
| `Ctrl+C` | Interrompre requête en cours |
| `Ctrl+D` ou `:q` | Quitter Kode |

### Autocomplétion Intelligente

Kode propose automatiquement :

**1. Commandes slash** :
```
> /h[Tab]
→ /help
```

**2. Agents** :
```
> @doc[Tab]
→ @docs-expert
```

**3. Fichiers** :
```
> Read src/[Tab]
→ src/app.ts
→ src/utils.ts
→ ...
```

**4. Modèles** :
```
> Ask [Tab]
→ gpt-5
→ claude-sonnet-4
→ ...
```

### Commandes Principales

| Commande | Description |
|----------|-------------|
| `/help` | Aide interactive |
| `/model` | Gestion des modèles IA |
| `/agents` | Liste et gestion des agents |
| `/config` | Configuration de Kode |
| `/clear` | Effacer l'écran |
| `/compact` | Compacter la conversation |
| `/cost` | Afficher le coût cumulé |
| `/history` | Voir l'historique des sessions |
| `/resume` | Reprendre une session |

---

## 3. Utilisation Quotidienne

### Cas d'Usage Typiques

#### 1. Analyse de Code

```
> Analyse le fichier src/app.ts et explique son architecture
```

**Kode va** :
1. Lire le fichier avec `FileRead`
2. Analyser le code
3. Expliquer l'architecture

#### 2. Création de Fonction

```
> Crée une fonction qui valide les emails dans utils/validation.ts
```

**Kode va** :
1. Lire le fichier existant
2. Générer la fonction
3. Demander permission pour éditer
4. Insérer la fonction au bon endroit

**Dialogue de permission** :
```
┌─ Permission Request ─────────────────┐
│ FileEdit wants to modify:            │
│ utils/validation.ts                  │
│                                      │
│ [a] Allow  [r] Reject  [p] Permanent│
└──────────────────────────────────────┘
```

Tapez `a` pour autoriser une fois, ou `p` pour autoriser de manière permanente.

#### 3. Débogage

```
> Ce code plante, peux-tu trouver le problème ?

[Collez le code ou indiquez le fichier]
```

**Kode va** :
1. Analyser le code
2. Identifier les bugs potentiels
3. Proposer des corrections
4. Demander si vous voulez appliquer les corrections

#### 4. Recherche dans le Code

```
> Trouve toutes les fonctions qui utilisent l'API de paiement
```

**Kode va** :
1. Utiliser `Grep` pour chercher dans le code
2. Lister les résultats
3. Vous pouvez demander plus de détails sur chaque résultat

#### 5. Tests

```
> Écris des tests pour la fonction validateEmail
```

**Kode va** :
1. Lire la fonction
2. Générer des tests unitaires
3. Créer/éditer le fichier de test
4. Vous pouvez ensuite lancer les tests avec `! npm test`

### Workflow Typique

**1. Démarrer une session** :
```bash
cd /path/to/project
kode
```

**2. Demander une analyse** :
```
> Analyse la structure du projet et identifie les points d'amélioration
```

**3. Implémenter les changements** :
```
> Implémente une validation des entrées pour le formulaire de contact
```

**4. Vérifier avec des tests** :
```
! npm test
```

**5. Commit** :
```
! git add .
! git commit -m "Add input validation"
```

**6. Notes pour la documentation** :
```
# Ajouté validation avec zod pour les formulaires
# Pattern réutilisable dans src/validation/
```

### Gestion des Permissions

**Modes de permission** :

1. **Default** : Demande confirmation pour chaque outil
2. **Safe** : Mode strict, demande pour TOUT
3. **Auto** : Exécution automatique (si configuré)

**Changer de mode** :
```
Shift+Tab
```

**Permissions permanentes** :

Quand vous choisissez `[p] Permanent`, Kode sauvegarde dans `.kode.json` :

```json
{
  "allowedTools": [
    "FileEdit(/home/user/project/**)",
    "Bash(npm:*)",
    "Bash(git:*)"
  ]
}
```

### Gestion du Contexte

**Compaction automatique** :

Quand votre conversation atteint 92% de la limite de contexte, Kode compacte automatiquement :

```
✨ Context auto-compacted...
Previous conversation (150 messages) summarized.
```

**Compaction manuelle** :
```
> /compact
```

**Reprendre une conversation** :

Kode sauvegarde automatiquement chaque session dans `~/.kode/messages/`

```
> /history
# Liste des sessions

> /resume 0
# Reprend la session #0
```

---

## 4. Configuration des Modèles

### Ajouter un Modèle

```
> /model
```

**Interface interactive** :
1. "Add new model"
2. Provider: `openai` / `anthropic` / `deepseek` / etc.
3. API Key: votre clé
4. Modèle spécifique: choisir dans la liste
5. Configuration (optionnel) :
   - Context length
   - Max tokens
   - Reasoning effort (GPT-5)

### Configurer les Model Pointers

**Concept** : Assigner différents modèles pour différents usages

```
> /model
→ "Configure model pointers"

Main model: gpt-5
Task model: claude-sonnet-4
Reasoning model: deepseek-reasoner
Quick model: gpt-4o-mini
```

**Avantages** :
- **Main** : Conversations principales (équilibre performance/coût)
- **Task** : Sub-agents (meilleur pour outils)
- **Reasoning** : Problèmes complexes (spécialisé)
- **Quick** : Opérations rapides (économique)

### Changer de Modèle Rapidement

**Méthode 1** : Raccourci clavier
```
Shift+M
```

Kode cycle entre tous les modèles configurés :
```
✅ Switched to Claude Sonnet 4 (2/4) [anthropic]
```

**Méthode 2** : Commande
```
> /model
→ "Switch active model"
→ Choisir dans la liste
```

### Providers Supportés

- **OpenAI** : GPT-4o, GPT-5, GPT-4o-mini, O1
- **Anthropic** : Claude Sonnet 4, Claude Opus, Claude Haiku
- **DeepSeek** : DeepSeek Reasoner, DeepSeek Chat
- **Mistral** : Mistral Large, Mistral Medium
- **Groq** : Modèles rapides
- **Gemini** : Google Gemini Pro
- **Ollama** : Modèles locaux
- **Azure OpenAI** : Via endpoints Azure
- **Custom** : N'importe quel endpoint OpenAI-compatible

### Configuration Avancée

**Fichier** : `~/.kode.json`

```json
{
  "modelProfiles": [
    {
      "name": "GPT-5 Main",
      "modelName": "gpt-5",
      "provider": "openai",
      "apiKey": "sk-...",
      "contextLength": 200000,
      "maxTokens": 32768,
      "reasoningEffort": "medium",
      "isActive": true
    }
  ],
  "modelPointers": {
    "main": "gpt-5",
    "task": "claude-sonnet-4-20250514",
    "reasoning": "deepseek-reasoner",
    "quick": "gpt-4o-mini"
  },
  "defaultModelName": "gpt-5"
}
```

---

## 5. Création et Utilisation d'Agents

### Qu'est-ce qu'un Agent ?

Un **agent** est une configuration spécialisée de Kode pour un type de tâche particulier.

**Avantages** :
- Prompt système personnalisé
- Outils limités (sécurité)
- Modèle spécifique
- Réutilisable et partageable

### Voir les Agents Disponibles

```
> /agents
```

**Affiche** :
- Agents built-in
- Agents utilisateur (`~/.kode/agents/`)
- Agents projet (`./.kode/agents/`)

### Utiliser un Agent

**Méthode 1** : Mention
```
> @docs-expert Crée la documentation pour ce module
```

**Méthode 2** : Commande Task
```
> Utilise l'agent security-auditor pour analyser le code
```

### Créer votre Premier Agent

#### Étape 1 : Créer le Fichier

**Global** (disponible dans tous les projets) :
```bash
mkdir -p ~/.kode/agents
nano ~/.kode/agents/mon-expert.md
```

**Projet** (spécifique au projet actuel) :
```bash
mkdir -p ./.kode/agents
nano ./.kode/agents/mon-expert.md
```

#### Étape 2 : Définir l'Agent

```markdown
---
name: mon-expert
description: "Expert spécialisé dans mon domaine"
tools: ["Read", "Grep", "Glob"]
model_name: main
color: blue
---

Vous êtes un expert spécialisé dans [domaine].

Votre mission :
1. Analyser le code avec attention
2. Proposer des améliorations
3. Suivre les meilleures pratiques

Guidelines :
- Toujours commencer par lire les fichiers pertinents
- Proposer des solutions complètes
- Expliquer vos choix
```

#### Étape 3 : Utiliser l'Agent

L'agent est **immédiatement disponible** (hot reload) :

```
> @mon-expert Analyse le module auth
```

**Ou via la commande /agents** :
```
> /agents
→ Choisir "mon-expert"
→ Entrer votre requête
```

### Exemples d'Agents Utiles

#### Agent de Documentation

```markdown
---
name: docs-writer
description: "Spécialisé dans la création de documentation"
tools: ["Read", "Write", "Edit", "Grep", "Glob"]
color: purple
---

Vous êtes un expert en documentation technique.

Mission : Créer une documentation claire, complète et bien structurée.

Format :
- Markdown professionnel
- Exemples de code
- Sections bien organisées
- Table des matières

Toujours :
1. Lire le code existant
2. Identifier les points clés
3. Rédiger de manière pédagogique
```

#### Agent de Tests

```markdown
---
name: test-writer
description: "Spécialisé dans l'écriture de tests"
tools: ["Read", "Write", "Edit", "Bash"]
model_name: task
---

Vous êtes un expert en testing.

Créez des tests :
- Unitaires complets
- Cas edge couverts
- Assertions claires
- Setup/teardown appropriés

Framework : Détectez automatiquement (Jest, Vitest, etc.)
```

#### Agent de Sécurité

```markdown
---
name: security-auditor
description: "Audit de sécurité du code"
tools: ["Read", "Grep", "Glob"]
color: red
---

Vous êtes un expert en sécurité.

Recherchez :
- Injections SQL, XSS, CSRF
- Secrets hardcodés
- Dépendances vulnérables
- Mauvaises pratiques crypto
- Problèmes d'authentification

Fournissez :
- Liste priorisée des vulnérabilités
- Solutions concrètes
- Code de correction
```

### Hiérarchie des Agents

Kode charge les agents dans cet ordre (du moins au plus prioritaire) :

```
1. Built-in (code intégré)
2. ~/.claude/agents/  (compatibilité Claude Code - utilisateur)
3. ~/.kode/agents/    (utilisateur Kode)
4. ./.claude/agents/  (projet Claude Code)
5. ./.kode/agents/    (projet Kode - PRIORITÉ MAX)
```

**Conseil** : Mettez les agents génériques dans `~/.kode/agents/` et les agents spécifiques au projet dans `./.kode/agents/`

---

## 6. Astuces et Bonnes Pratiques

### Astuces d'Utilisation

#### 1. Autocomplétion Maximale

Utilisez Tab partout :
- Commandes : `/h` + Tab → `/help`
- Fichiers : `Read src/` + Tab → liste
- Agents : `@` + Tab → liste d'agents

#### 2. Historique Intelligent

Naviguer avec `↑` / `↓` :
- Kode se souvient de vos commandes
- Historique sauvegardé par projet
- Peut contenir vos requêtes complexes

#### 3. Copier-Coller Efficace

```
> [Collez un long code]
Kode détecte automatiquement les grandes quantités de texte

> Analyse ce code et trouve les bugs
```

#### 4. Sessions Multiples

Vous pouvez avoir plusieurs sessions Kode :
```bash
# Terminal 1
cd /project-a
kode

# Terminal 2
cd /project-b
kode
```

Chaque session a son propre contexte et configuration.

#### 5. Mode Non-Interactif

Pour scripts ou CI/CD :
```bash
kode -p "Analyse le code et génère un rapport" > report.md
```

### Bonnes Pratiques

#### 1. Configuration Projet

Créez un `.kode.json` dans votre projet :

```json
{
  "allowedTools": [
    "FileEdit(/path/to/project/**)",
    "Bash(npm:*)",
    "Bash(git:*)"
  ],
  "contextFiles": [
    "AGENTS.md",
    "ARCHITECTURE.md"
  ],
  "dontCrawlDirectory": false,
  "enableArchitectTool": true
}
```

#### 2. Documentation Projet

Créez `AGENTS.md` ou `CLAUDE.md` :

```markdown
# Instructions pour les Agents Kode

## Architecture
- Pattern: Clean Architecture
- Framework: React + TypeScript
- Tests: Vitest

## Guidelines
- Utiliser des fonctions pures
- Tests obligatoires pour nouvelle logique
- Pas de any en TypeScript

## Structure
src/
├── components/
├── services/
├── utils/
└── types/
```

Kode chargera automatiquement ce contexte !

#### 3. Agents Projet

Créez des agents spécifiques à votre stack :

```bash
mkdir -p .kode/agents

# Agent React
cat > .kode/agents/react-expert.md << 'EOF'
---
name: react-expert
description: "Expert React pour ce projet"
tools: ["Read", "Write", "Edit"]
---

Expert React utilisant nos guidelines :
- Hooks (pas de class components)
- TypeScript strict
- Tests avec React Testing Library
EOF
```

#### 4. Git Integration

Configurez les permissions git une fois pour toutes :

```
> /model
→ "Configure permissions"
→ Ajouter : Bash(git:*)

Maintenant Kode peut :
! git status
! git diff
! git add .
! git commit -m "..."
```

#### 5. Coûts Optimisés

Utilisez les Model Pointers :
- `quick` pour questions simples
- `main` pour développement
- `reasoning` pour problèmes complexes

```
> /model
→ Configure model pointers
→ quick: gpt-4o-mini (économique)
→ main: claude-sonnet-4 (équilibré)
→ reasoning: gpt-5 (puissant)
```

### Personnalisation

#### Thèmes

```
> /config
→ "Theme"
→ dark / light / light-daltonized / dark-daltonized
```

#### Verbose Mode

Pour voir plus de détails :
```bash
kode --verbose
```

Affiche :
- Appels d'outils
- Temps d'exécution
- Tokens utilisés
- Logs de debug

---

## 7. Dépannage

### Problèmes Courants

#### "No model configured"

**Solution** :
```
> /model
→ Add new model
→ Suivre les étapes
```

#### "API key invalid"

**Solutions** :
1. Vérifier la clé API :
```
> /model
→ Edit model
→ Mettre à jour l'API key
```

2. Vérifier les variables d'environnement :
```bash
echo $ANTHROPIC_API_KEY
echo $OPENAI_API_KEY
```

#### "Context window full"

Kode compacte automatiquement à 92%.

**Manuel** :
```
> /compact
```

**Ou reprendre avec contexte frais** :
```
> /clear
```

#### "Permission denied"

Si en mode safe :
```
> /config
→ "Safe mode"
→ Disable (si approprié)
```

**Ou ajouter permissions** :
```
> Quand demandé, choisir [p] Permanent
```

#### "Tool not found"

Vérifier que l'outil est activé :
```
> /help
→ Voir liste des outils disponibles
```

Certains outils nécessitent des variables d'environnement.

### Réinitialisation

**Configuration globale** :
```bash
rm -rf ~/.kode/
kode
# Reconfigurer
```

**Configuration projet** :
```bash
rm .kode.json
# Kode utilisera config globale
```

### Debugging

**Mode verbose** :
```bash
kode --verbose
```

**Logs** :
```bash
ls ~/.kode/*/debug/
cat ~/.kode/*/debug/*-detailed.log
```

**Diagnostic système** :
```
> /doctor
# Affiche l'état du système
```

### Support

- **Documentation** : Consultez cette doc !
- **Issues** : https://github.com/Pipikeen/Kode/issues
- **Discussions** : Pour questions générales

---

## Résumé des Commandes Essentielles

| Action | Commande |
|--------|----------|
| Aide | `/help` |
| Configurer modèle | `/model` |
| Lister agents | `/agents` |
| Configuration | `/config` |
| Compacter contexte | `/compact` |
| Historique | `/history` |
| Reprendre session | `/resume N` |
| Coût total | `/cost` |
| Quitter | `Ctrl+D` ou `:q` |

## Raccourcis Essentiels

| Raccourci | Action |
|-----------|--------|
| `Tab` | Autocomplétion |
| `↑` / `↓` | Historique |
| `Shift+M` | Changer modèle |
| `Shift+Tab` | Mode permission |
| `Esc` | Sélecteur de messages |
| `Ctrl+C` | Interrompre |

---

**Bon développement avec Kode ! 🚀**

[← Retour à l'index](./README.md)
