# Index Complet de la Documentation Kode

> Navigation rapide vers tous les documents de la documentation

---

## 📚 Documentation Disponible

### 🎯 Documentation Principale

#### [README - Vue d'Ensemble](./README.md)
**12 KB** - **Point de départ recommandé**

- Vue d'ensemble du projet Kode
- Philosophie et vision
- Points forts et innovations
- Comparaison avec autres outils
- Table des matières générale
- Guide de navigation

**À lire si** : Vous découvrez Kode pour la première fois

---

### 🏗️ Documentation Technique

#### [Architecture Système](./02-architecture.md)
**44 KB** - **Architecture complète**

- Organisation des répertoires (272 fichiers TS)
- Les 3 couches principales (UI, Orchestration, Execution)
- Flux de données et interactions
- Points d'entrée et cycle de vie
- Interactions entre modules
- Patterns architecturaux (AsyncGenerator, Factory, Strategy, etc.)
- Points clés d'architecture

**À lire si** : Vous voulez comprendre comment Kode fonctionne en profondeur

**Fichiers clés mentionnés** :
- `src/screens/REPL.tsx` - Interface REPL (810 lignes)
- `src/query.ts` - Boucle d'orchestration
- `src/Tool.ts` - Interface de base des outils
- `src/tools/TaskTool/` - Système d'agents
- `src/services/claude.ts` - Service Anthropic (2242 lignes)
- `src/utils/model.ts` - ModelManager

---

#### [Système Multi-Modèles](./03-systeme-modeles.md)
**29 KB** - **Innovation majeure de Kode**

- Vue d'ensemble et comparaison
- Architecture globale des modèles
- ModelManager et orchestration
- Model Pointers (main, task, reasoning, quick)
- Model Capabilities et métadonnées
- Adaptateurs de modèles (ResponsesAPI, ChatCompletions)
- Guide d'ajout de nouveaux modèles
- Capacités par type (GPT-5, Claude, O1, DeepSeek)
- Exemples pratiques de configuration

**À lire si** : Vous voulez comprendre ou configurer le système multi-modèles

**Innovations clés** :
- Support illimité de modèles
- Switching dynamique sans redémarrage
- Auto-correction des erreurs de paramètres
- Support complet GPT-5 Responses API

---

### 📖 Guides Pratiques

#### [Guide Utilisateur](./10-guide-utilisateur.md)
**19 KB** - **Guide pratique complet**

- Installation et premier démarrage
- Interface et navigation (3 modes : Prompt, Bash, Koding)
- Utilisation quotidienne
- Configuration des modèles
- Création et utilisation d'agents
- Astuces et bonnes pratiques
- Dépannage

**À lire si** : Vous voulez utiliser Kode efficacement au quotidien

**Couvert** :
- Tous les raccourcis clavier
- Autocomplétion intelligente
- Commandes principales (`/help`, `/model`, `/agents`, etc.)
- Workflows typiques
- Cas d'usage pratiques
- Configuration projet
- Problèmes courants et solutions

---

## 🗂️ Par Thématique

### Pour Démarrer

1. **[README](./README.md)** - Vue d'ensemble
2. **[Guide Utilisateur](./10-guide-utilisateur.md)** - Installation et premiers pas
3. **[Guide Utilisateur § Configuration](./10-guide-utilisateur.md#4-configuration-des-modèles)** - Configurer votre premier modèle

### Pour Comprendre Kode

1. **[README § Architecture](./README.md#architecture-en-3-couches)** - Architecture simplifiée
2. **[Architecture](./02-architecture.md)** - Architecture complète
3. **[Système Multi-Modèles](./03-systeme-modeles.md)** - Gestion des modèles IA
4. **[Système d'Agents](./04-systeme-agents.md)** - Agents dynamiques
5. **[Système d'Outils](./05-systeme-outils.md)** - 21+ outils natifs

### Pour Utiliser Kode

1. **[Guide Utilisateur](./10-guide-utilisateur.md)** - Guide complet
2. **[Guide Utilisateur § Utilisation Quotidienne](./10-guide-utilisateur.md#3-utilisation-quotidienne)** - Workflows
3. **[Guide Utilisateur § Agents](./10-guide-utilisateur.md#5-création-et-utilisation-dagents)** - Agents personnalisés

### Pour Configurer

1. **[Guide Utilisateur § Modèles](./10-guide-utilisateur.md#4-configuration-des-modèles)** - Configuration modèles
2. **[Système Multi-Modèles § Ajouter un Modèle](./03-systeme-modeles.md#6-ajouter-un-nouveau-modèle)** - Ajout modèles personnalisés
3. **[Guide Utilisateur § Bonnes Pratiques](./10-guide-utilisateur.md#bonnes-pratiques)** - Configuration projet

---

## 📊 Statistiques de la Documentation

### Documents Créés

| Document | Taille | Lignes | Description |
|----------|--------|--------|-------------|
| **README.md** | 12 KB | 369 | Vue d'ensemble et navigation |
| **02-architecture.md** | 44 KB | 1191 | Architecture système complète |
| **03-systeme-modeles.md** | 29 KB | 1065 | Système multi-modèles |
| **04-systeme-agents.md** | 25 KB | 811 | Système d'agents dynamiques |
| **05-systeme-outils.md** | 31 KB | 1068 | Catalogue 21+ outils, permissions |
| **06-interface-utilisateur.md** | 29 KB | 1024 | REPL Ink/React, 70+ composants |
| **07-configuration.md** | 33 KB | 1050 | Hiérarchie config, auto-compaction |
| **08-services-integrations.md** | 28 KB | 920 | Services Claude, OpenAI, MCP, logging |
| **10-guide-utilisateur.md** | 19 KB | 576 | Guide pratique complet |
| **INDEX.md** | Ce fichier | - | Index de navigation |
| **TOTAL** | **296 KB** | **10,573** | Documentation exhaustive complète |

### Couverture

**Architecture** :
- ✅ Vue d'ensemble système
- ✅ Les 3 couches (UI, Orchestration, Execution)
- ✅ Flux de données
- ✅ Patterns architecturaux
- ✅ Points d'entrée et cycle de vie

**Système Multi-Modèles** :
- ✅ ModelManager et orchestration
- ✅ Model Pointers (innovation)
- ✅ Adaptateurs (GPT-5 Responses API, etc.)
- ✅ 20+ providers supportés
- ✅ Guide d'ajout de modèles

**Guide Utilisateur** :
- ✅ Installation et configuration
- ✅ Interface et navigation
- ✅ Utilisation quotidienne
- ✅ Workflows pratiques
- ✅ Agents personnalisés
- ✅ Astuces et dépannage

**Documentation Technique Complète** :
- ✅ Système d'agents (détails techniques) - [04-systeme-agents.md](./04-systeme-agents.md)
- ✅ Système d'outils (21+ outils) - [05-systeme-outils.md](./05-systeme-outils.md)
- ✅ Interface utilisateur (Ink/React) - [06-interface-utilisateur.md](./06-interface-utilisateur.md)
- ✅ Configuration avancée - [07-configuration.md](./07-configuration.md)
- ✅ Services et intégrations - [08-services-integrations.md](./08-services-integrations.md)

**Documentation Future** (optionnelle) :
- ⏳ Guide développeur avancé
- ⏳ Guide de contribution au code
- ⏳ Architecture interne des composants React

---

## 🎯 Parcours Recommandés

### Nouveau Utilisateur

1. **[README](./README.md)** - Découvrir Kode
2. **[Guide Utilisateur § Installation](./10-guide-utilisateur.md#1-installation-et-premier-démarrage)** - Installer
3. **[Guide Utilisateur § Interface](./10-guide-utilisateur.md#2-interface-et-navigation)** - Apprendre l'interface
4. **[Guide Utilisateur § Utilisation](./10-guide-utilisateur.md#3-utilisation-quotidienne)** - Commencer à utiliser

### Utilisateur Avancé

1. **[Système Multi-Modèles](./03-systeme-modeles.md)** - Maîtriser les modèles
2. **[Guide Utilisateur § Agents](./10-guide-utilisateur.md#5-création-et-utilisation-dagents)** - Créer agents
3. **[Guide Utilisateur § Bonnes Pratiques](./10-guide-utilisateur.md#bonnes-pratiques)** - Optimiser workflow
4. **[Architecture](./02-architecture.md)** - Comprendre le système

### Développeur/Contributeur

1. **[Architecture](./02-architecture.md)** - Architecture complète
2. **[Système Multi-Modèles](./03-systeme-modeles.md)** - Système de modèles
3. **[Système d'Agents](./04-systeme-agents.md)** - Agents et chargement dynamique
4. **[Système d'Outils](./05-systeme-outils.md)** - Outils et permissions
5. **[Interface Utilisateur](./06-interface-utilisateur.md)** - REPL et composants React
6. **[Configuration](./07-configuration.md)** - Config hiérarchique et auto-compaction
7. **[Services](./08-services-integrations.md)** - Claude, OpenAI, MCP, logging
8. **[Architecture § Patterns](./02-architecture.md#7-patterns-architecturaux)** - Patterns utilisés

---

## 🔍 Recherche Rapide

### Par Concept

- **Multi-modèles** → [Système Multi-Modèles](./03-systeme-modeles.md)
- **Model Pointers** → [Système Multi-Modèles § Model Pointers](./03-systeme-modeles.md#model-pointers-innovation)
- **Agents** → [Système d'Agents](./04-systeme-agents.md)
- **Outils** → [Système d'Outils](./05-systeme-outils.md)
- **Interface** → [Interface Utilisateur](./06-interface-utilisateur.md)
- **Configuration** → [Configuration](./07-configuration.md)
- **Services** → [Services et Intégrations](./08-services-integrations.md)
- **Architecture** → [Architecture Système](./02-architecture.md)
- **Installation** → [Guide Utilisateur § Installation](./10-guide-utilisateur.md#installation)

### Par Action

- **Installer Kode** → [Guide Utilisateur § Installation](./10-guide-utilisateur.md#installation)
- **Ajouter un modèle** → [Système Multi-Modèles § Ajouter un Modèle](./03-systeme-modeles.md#6-ajouter-un-nouveau-modèle)
- **Créer un agent** → [Guide Utilisateur § Créer Agent](./10-guide-utilisateur.md#créer-votre-premier-agent)
- **Changer de modèle** → [Guide Utilisateur § Changer Modèle](./10-guide-utilisateur.md#changer-de-modèle-rapidement)
- **Comprendre l'architecture** → [Architecture](./02-architecture.md)

### Par Problème

- **Problème d'installation** → [Guide Utilisateur § Dépannage](./10-guide-utilisateur.md#7-dépannage)
- **Erreur API key** → [Guide Utilisateur § Dépannage § API key](./10-guide-utilisateur.md#api-key-invalid)
- **Contexte plein** → [Guide Utilisateur § Dépannage § Context](./10-guide-utilisateur.md#context-window-full)
- **Permission refusée** → [Guide Utilisateur § Dépannage § Permission](./10-guide-utilisateur.md#permission-denied)

---

## 📝 Notes

Cette documentation est maintenant **complète** !

Les documents couvrent tous les aspects essentiels de Kode :
- ✅ Démarrage et utilisation quotidienne (README, Guide Utilisateur)
- ✅ Architecture système complète (Architecture, Agents, Outils, UI)
- ✅ Innovation majeure multi-modèles (Système Multi-Modèles)
- ✅ Configuration avancée (Configuration, Auto-compaction)
- ✅ Services et intégrations (Claude, OpenAI, MCP, Logging)

Pour toute question ou suggestion d'amélioration, n'hésitez pas à ouvrir une issue sur GitHub.

---

**Documentation générée le 16 janvier 2025**

**Version de Kode** : 0.1.0

**Contributeurs** : Documentation exhaustive créée par analyse approfondie du code source

---

[← Retour au README principal](./README.md)
