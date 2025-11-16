# Documentation Complète de Kode

> **CLI Interactif Multi-Modèles pour le Développement Assisté par IA**

Bienvenue dans la documentation exhaustive de Kode, un outil de ligne de commande sophistiqué qui révolutionne le développement logiciel en combinant plusieurs modèles d'IA, un système d'agents spécialisés et une architecture extensible.

---

## 📚 Table des Matières

### 🎯 Introduction
- [Vue d'Ensemble du Projet](#vue-densemble-du-projet)
- [Philosophie et Vision](#philosophie-et-vision)
- [Points Forts et Innovations](#points-forts-et-innovations)

### 📖 Documentation Principale

1. **[Architecture Système](./02-architecture.md)**
   - Architecture à trois couches
   - Flux de données et interactions
   - Patterns architecturaux utilisés
   - Points d'entrée et cycle de vie

2. **[Système Multi-Modèles](./03-systeme-modeles.md)**
   - ModelManager et orchestration
   - Adaptateurs de modèles (GPT-5, Claude, etc.)
   - Model Pointers (main, task, reasoning, quick)
   - Support de 20+ providers IA
   - Switching dynamique entre modèles

3. **[Système d'Agents Dynamiques](./04-systeme-agents.md)**
   - Hiérarchie à 5 niveaux de priorité
   - Format des agents (YAML + Markdown)
   - Compatibilité Claude Code
   - Hot reload et cache
   - Création d'agents personnalisés

4. **[Système d'Outils et Permissions](./05-systeme-outils.md)**
   - 21+ outils natifs
   - Architecture des outils
   - Système de permissions granulaire
   - Intégration MCP (Model Context Protocol)
   - Création de nouveaux outils

5. **[Interface Utilisateur et REPL](./06-interface-utilisateur.md)**
   - Architecture React/Ink
   - Trois modes d'interaction (Prompt, Bash, Koding)
   - Système de commandes
   - Autocomplétion intelligente
   - Thèmes et coloration syntaxique

6. **[Configuration et Contexte](./07-configuration.md)**
   - Configuration hiérarchique (4 niveaux)
   - Gestion du contexte conversationnel
   - Message Context Manager
   - Auto-compaction intelligente
   - Système de mémoire persistante

7. **[Services et Intégrations](./08-services-integrations.md)**
   - Service Anthropic (Claude)
   - Service OpenAI et compatibles
   - Client MCP
   - Adaptateurs de modèles
   - Système de logging et debugging

### 🛠️ Guides Pratiques

8. **[Guide du Développeur](./09-guide-developpeur.md)**
   - Setup de l'environnement de développement
   - Structure du code
   - Workflows de développement
   - Tests et debugging
   - Contribution au projet

9. **[Guide de l'Utilisateur](./10-guide-utilisateur.md)**
   - Installation et démarrage rapide
   - Utilisation quotidienne
   - Configuration des modèles
   - Création d'agents
   - Astuces et bonnes pratiques

10. **[Guide de Configuration](./11-guide-configuration.md)**
    - Configuration globale et projet
    - Variables d'environnement
    - Serveurs MCP
    - Personnalisation avancée

---

## 🎯 Vue d'Ensemble du Projet

### Qu'est-ce que Kode ?

**Kode** est un assistant de développement en ligne de commande de nouvelle génération qui combine :

- **🤖 Multi-Modèles** : Support illimité de modèles IA (GPT-5, Claude, DeepSeek, Mistral, etc.)
- **🎭 Agents Spécialisés** : Système dynamique d'agents pour des tâches spécifiques
- **🔧 Outils Puissants** : 21+ outils natifs + extensibilité via MCP
- **💬 Interface Interactive** : REPL moderne avec React/Ink
- **🔐 Sécurité** : Système de permissions granulaire
- **⚡ Performance** : Gestion intelligente du contexte et cache

### Architecture en 3 Couches

```
┌─────────────────────────────────────────────┐
│  Couche 1: Interface Utilisateur (REPL)     │
│  - React/Ink pour le terminal               │
│  - 3 modes d'interaction                    │
│  - Autocomplétion intelligente              │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│  Couche 2: Orchestration (Query Loop)       │
│  - Boucle récursive LLM → Tools → LLM       │
│  - Gestion du contexte conversationnel      │
│  - Système d'agents (TaskTool)              │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│  Couche 3: Exécution des Outils             │
│  - 21 outils natifs (Fichiers, Bash, etc.)  │
│  - Outils MCP externes                      │
│  - Système de permissions                   │
└─────────────────────────────────────────────┘
```

---

## 💡 Philosophie et Vision

### Principes Fondamentaux

1. **Flexibilité Maximale**
   - Utiliser le meilleur modèle pour chaque tâche
   - Switching dynamique sans redémarrage
   - Extensibilité via agents et outils personnalisés

2. **Intelligence Contextuelle**
   - Gestion sophistiquée du contexte (200K+ tokens)
   - Auto-compaction automatique
   - Mémoire persistante entre sessions

3. **Sécurité et Contrôle**
   - Permissions granulaires par outil
   - Validation des chemins de fichiers
   - Mode safe pour opérations critiques

4. **Performance**
   - Streaming en temps réel
   - Cache multi-niveaux (agents, contexte, prompts)
   - Exécution concurrente d'outils

5. **Expérience Développeur**
   - Interface CLI moderne et réactive
   - Coloration syntaxique
   - Historique et autocomplétion
   - Debugging exhaustif

---

## 🚀 Points Forts et Innovations

### Innovation 1: Système Multi-Modèles Avancé

**Kode est l'un des rares outils à supporter vraiment plusieurs modèles IA de manière transparente.**

- **Model Pointers** : Assignez des modèles différents pour différents usages
  - `main` : Modèle principal pour conversations
  - `task` : Modèle pour sous-agents (meilleur pour outils)
  - `reasoning` : Modèle pour problèmes complexes
  - `quick` : Modèle rapide pour opérations simples

- **Switching Dynamique** : Changez de modèle en cours de session (`Shift+M`)

- **20+ Providers** : OpenAI, Anthropic, DeepSeek, Groq, Mistral, Gemini, Ollama, etc.

- **Support GPT-5 Responses API** : Premier CLI open-source à supporter complètement la nouvelle API GPT-5

### Innovation 2: Agents Dynamiques à 5 Niveaux

**Système unique de chargement d'agents avec compatibilité Claude Code.**

```
Priorité 5 (Max) : ./.kode/agents/          # Projet Kode
Priorité 4       : ./.claude/agents/        # Projet Claude Code
Priorité 3       : ~/.kode/agents/          # Utilisateur Kode
Priorité 2       : ~/.claude/agents/        # Utilisateur Claude Code
Priorité 1 (Min) : Built-in agents          # Intégrés au code
```

**Hot Reload** : Modifiez vos agents, ils sont rechargés automatiquement.

### Innovation 3: Gestion Intelligente du Contexte

**Jamais à court de contexte grâce à 4 stratégies de troncature.**

- **Auto-Compact à 92%** : Compression automatique avant saturation
- **Preserve Recent** : Garde les N derniers messages
- **Preserve Important** : Identifie et conserve les messages clés
- **Smart Compression** : Résumé IA des anciens messages

**Résultat** : Conversations de plusieurs heures sans perte d'information.

### Innovation 4: Architecture d'Adaptateurs de Modèles

**Abstraction élégante des différences entre APIs.**

```typescript
ModelAdapterFactory
  ├── ResponsesAPIAdapter      (GPT-5)
  ├── ChatCompletionsAdapter   (GPT-4, Claude via wrapper)
  └── [Extensible facilement]
```

**Auto-correction d'erreurs** : Détecte et corrige automatiquement les erreurs de paramètres API.

### Innovation 5: Intégration MCP (Model Context Protocol)

**Extensibilité illimitée via des outils externes.**

- Serveurs MCP locaux (stdio) ou distants (SSE)
- Conversion automatique en outils Kode
- Système d'approbation pour la sécurité
- Support d'images dans les réponses

---

## 📊 Comparaison avec d'Autres Outils

| Fonctionnalité | Kode | Cursor | GitHub Copilot | Claude CLI |
|---|:---:|:---:|:---:|:---:|
| **Multi-modèles** | ✅ Illimité | ⚠️ 2-3 | ❌ 1 | ❌ 1 |
| **Switching dynamique** | ✅ Oui | ❌ Non | ❌ Non | ❌ Non |
| **Model Pointers** | ✅ 4 types | ❌ Non | ❌ Non | ❌ Non |
| **Agents personnalisés** | ✅ Oui | ⚠️ Limité | ❌ Non | ❌ Non |
| **MCP Support** | ✅ Oui | ❌ Non | ❌ Non | ⚠️ Partiel |
| **GPT-5 Responses API** | ✅ Complet | ❌ Non | ❌ Non | ❌ Non |
| **Auto-compaction** | ✅ 4 stratégies | ❌ Non | ❌ Non | ❌ Non |
| **Permissions granulaires** | ✅ Oui | ⚠️ Basique | ❌ Non | ❌ Non |
| **Hot reload agents** | ✅ Oui | ❌ Non | ❌ Non | ❌ Non |
| **CLI moderne** | ✅ React/Ink | ❌ Non | ❌ Non | ⚠️ Basique |

---

## 🏗️ Architecture Technique

### Technologies Clés

- **Runtime** : Bun (avec fallback Node.js)
- **Langage** : TypeScript strict
- **UI** : Ink (React pour CLI)
- **Validation** : Zod
- **Tests** : Bun test
- **Build** : Bun + TSC

### Métriques du Projet

- **272 fichiers TypeScript**
- **21+ outils natifs**
- **20+ providers IA supportés**
- **4 stratégies de gestion du contexte**
- **5 niveaux de priorité d'agents**
- **15+ commandes intégrées**

---

## 🎓 Pour Commencer

### Installation Rapide

```bash
# Installer Kode globalement
npm install -g kode-cli

# Ou avec bun
bun install -g kode-cli

# Lancer Kode
kode
```

### Première Configuration

```bash
# Configurer votre modèle IA
kode
> /model

# Ajouter votre clé API
# Choisir votre modèle principal
```

### Premier Agent Personnalisé

```bash
# Créer votre premier agent
mkdir -p ~/.kode/agents
cat > ~/.kode/agents/mon-agent.md << 'EOF'
---
name: mon-expert
description: "Expert dans mon domaine"
tools: ["Read", "Grep", "Glob"]
---

Vous êtes un expert spécialisé...
EOF

# L'agent est immédiatement disponible !
```

---

## 📖 Navigation de la Documentation

### Pour les Débutants

1. Commencez par le **[Guide de l'Utilisateur](./10-guide-utilisateur.md)**
2. Lisez le **[Guide de Configuration](./11-guide-configuration.md)**
3. Explorez les **[Agents](./04-systeme-agents.md)** et **[Outils](./05-systeme-outils.md)**

### Pour les Développeurs

1. Comprenez l'**[Architecture](./02-architecture.md)**
2. Étudiez le **[Système Multi-Modèles](./03-systeme-modeles.md)**
3. Consultez le **[Guide du Développeur](./09-guide-developpeur.md)**

### Pour les Contributeurs

1. Lisez le **[Guide du Développeur](./09-guide-developpeur.md)**
2. Comprenez les **[Services et Intégrations](./08-services-integrations.md)**
3. Explorez le code source avec les références de la documentation

---

## 🤝 Contribution

Kode est un projet open-source. Les contributions sont les bienvenues !

- **Issues** : Rapportez des bugs ou proposez des fonctionnalités
- **Pull Requests** : Soumettez des améliorations
- **Documentation** : Aidez à améliorer cette documentation
- **Agents** : Partagez vos agents personnalisés

---

## 📄 Licence

Kode est distribué sous licence MIT.

---

## 🙏 Remerciements

Merci à tous les contributeurs, aux communautés open-source, et aux équipes d'Anthropic, OpenAI et des autres providers IA qui rendent Kode possible.

---

## 📞 Support et Communauté

- **Documentation** : Vous êtes ici !
- **Issues GitHub** : Pour les bugs et demandes de fonctionnalités
- **Discussions** : Pour les questions et partages

---

**Bonne découverte de Kode ! 🚀**

*Documentation générée le 16 janvier 2025*
