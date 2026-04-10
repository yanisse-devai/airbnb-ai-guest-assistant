# Airbnb AI Agent — Réponses Automatiques Voyageurs
![Badge MVP](https://img.shields.io/badge/Status-MVP-green) ![N8N](https://img.shields.io/badge/Built%20with-N8N%20Cloud-orange) ![GPT-4o](https://img.shields.io/badge/IA-GPT--4o-purple) ![Perplexity](https://img.shields.io/badge/Search-Perplexity-blue) ![Version](https://img.shields.io/badge/Version-1.1-lightgrey)

Agent IA autonome qui détecte, qualifie et répond aux questions récurrentes des voyageurs Airbnb — transports, équipements, adresses — sans intervention manuelle dans 80% des cas.

---

## 🎯 Problème & Solution

**Problème :** Répondre manuellement aux mêmes questions voyageurs plusieurs fois par semaine (transports, lits disponibles, Stade de France, restaurants proches...) — 5 à 10 minutes par message, aucune valeur ajoutée.  
**Solution IA :** Agent LLM + N8N qui détecte les emails Airbnb en temps réel, raisonne sur le périmètre de la question, consulte une base de connaissance Google Docs, et répond de façon autonome ou alerte le propriétaire.  
**Impact :** < 30 secondes de traitement par message. Cible : 80%+ des messages traités sans intervention.

---

## 🏗 Architecture du Workflow

```
Outlook Trigger → IF (filtre voyageur) → Update Lu → AI Agent → Outlook Envoi
                                                          ↑
                                          Google Docs  |  Perplexity  |  Outlook Tool
```

**Logique de décision de l'agent — 3 étapes :**
1. La question est-elle dans mon périmètre ? → Sinon : alerte propriétaire (`HORS_PERIMETRE`)
2. La réponse est-elle dans le Google Docs ? → Si oui : réponse directe
3. Introuvable dans le doc → Perplexity recherche sur le web → Si toujours rien : `VERIFICATION_REQUISE`

---

## 🛠 Tech Stack

| Composant | Outil | Raison Choisie |
|-----------|-------|----------------|
| Orchestration | N8N Cloud v2.13+ | Workflow visuel, nœuds natifs Outlook & Google Docs |
| IA Core | GPT-4o via OpenRouter | Meilleur raisonnement outil, réponses naturelles multilingues |
| Base de connaissance | Google Docs | Mis à jour sans toucher au workflow |
| Recherche web | Perplexity API (sonar) | Fallback fiable pour infos absentes du doc |
| Trigger & envoi | Microsoft Outlook | Email natif — contournement absence d'API Airbnb |

---

## ⚙️ Configuration Requise

### Prérequis
- Compte N8N Cloud actif
- Clé API OpenRouter (modèle `gpt-4o`)
- Clé API Perplexity
- Compte Microsoft Outlook connecté à N8N
- Google Docs logement partagé avec le compte de service N8N

### Variables à configurer dans N8N
```
OPENROUTER_API_KEY   → Clé API OpenRouter
PERPLEXITY_API_KEY   → Clé API Perplexity
GOOGLE_DOC_ID        → ID du document Google Docs logement
OWNER_EMAIL          → Ton adresse email (destinataire des alertes)
AIRBNB_FOLDER        → Nom du dossier Outlook contenant les emails Airbnb
```

### Filtre Outlook Trigger
```
Folder         : Airbnb
Read Status    : Unread messages only
Sender         : express@airbnb.com
Filter Query   : contains(subject, 'a envoyé un message')
```

---

## 📄 Structure du Google Docs Logement

Le document est la source de vérité principale. Il doit contenir 8 sections :

```
# MON LOGEMENT AIRBNB — BASE DE CONNAISSANCE
## Informations générales   → adresse, étage, codes
## Couchage                 → type de lits, linge
## Transports               → métro, bus, RER, Vélib
## Stade de France          → trajet, durée, fréquence
## Paris — temps de trajet  → monuments, quartiers clés
## Équipements              → cuisine, wifi, confort
## Règles du logement       → check-in/out, fumeur, animaux
## Restaurants & commerces  → recommandations à proximité
```

> ⚠️ Règle : toujours écrire des durées en minutes et des noms de stations exacts. Une ligne vide vaut mieux qu'une information approximative — Perplexity prend le relais.

---

## 🔐 Périmètre & Garde-fous

**L'agent traite uniquement :**
- Transports et trajets (métro, bus, RER, durées)
- Lieux touristiques et temps de trajet
- Restaurants, commerces et adresses à proximité
- Équipements et informations pratiques du logement

**L'agent ne traite jamais :**
- Problèmes techniques, plaintes, litiges → alerte propriétaire immédiate
- Négociation de prix ou remboursements → alerte propriétaire immédiate
- Urgences de sécurité (fuite de gaz, etc.) → alerte propriétaire prioritaire
- Questions hors logement ou personnelles → alerte propriétaire immédiate

**Double garde-fou avant chaque envoi :**
1. `Tool Description` Outlook — instructions précises sur les 2 cas d'envoi autorisés
2. Auto-vérification dans le System Message — 3 questions avant tout envoi

Mots-clés de sortie lisibles dans les logs N8N : `HORS_PERIMETRE` / `VERIFICATION_REQUISE`

---

## ⚠️ Difficultés Rencontrées & Leçons

- **Défi 1 : Pas d'API Airbnb publique** — Impossible de brancher N8N directement sur la messagerie Airbnb → Exploitation des emails de notification Outlook comme déclencheur. *Leçon : toujours valider la faisabilité API avant de choisir l'architecture.*
- **Défi 2 : Emails Airbnb multiformats** — Tous les emails viennent de la même adresse (réservations, rappels, messages...) → Filtre sur l'objet (`contains subject 'a envoyé un message'`) + `Unread only`. *Leçon : filtrer tôt dans le workflow, pas en milieu de chaîne.*
- **Défi 3 : Autonomie vs. sécurité** — L'agent autonome peut envoyer sans filet → Double garde-fou (Tool Description + auto-vérification prompt). *Leçon : plus une action est irréversible, plus les garde-fous doivent être redondants.*
- **Défi 4 : Mémoire conversationnelle inutile** — Simple Memory ajoutée par réflexe → Session ID inexistant sur un email, risque de mélange de contexte entre voyageurs → Supprimée. *Leçon : la mémoire n'est utile que dans les architectures conversationnelles, pas email.*

**Key Takeaway :** La clarté du périmètre dans le prompt est plus importante que la puissance du modèle — un agent bien contraint répond mieux qu'un agent libre.

---

## 🔍 PM Insights & Arbitrages Clés

Ces arbitrages ont été documentés pendant la construction du workflow — ils révèlent les décisions où un mauvais choix aurait cassé le produit.

**Arbitrage 1 — Autonomie totale vs. nœud IF post-agent**  
Option A (nœud IF) : sécurité maximale, mais double code de maintenance. Option B choisie (outil Outlook direct dans l'agent) : autonomie totale, compensée par un double garde-fou prompt. Condition : le System Message doit être aussi précis qu'un nœud IF, pas plus vague.

**Arbitrage 2 — Google Docs vs. base vectorielle**  
Pinecone/Supabase auraient permis une recherche sémantique plus fine, mais à un coût d'infra unjustifié pour un seul logement. Google Docs suffît avec un prompt d'instruction de lecture linéaire, et reste éditable par le propriétaire sans compétence technique.

**Arbitrage 3 — Perplexity en fallback vs. toujours activé**  
Toujours activé = coûts API inutiles sur des questions couvertes par le doc. Fallback uniquement = économie sur 80%+ des cas. L'agent est instruit pour ne jamais appeler Perplexity si le Google Docs contient la réponse.

**Arbitrage 4 — Pas de mémoire conversationnelle**  
Simple Memory a été ajoutée puis supprimée : les emails Airbnb n'ont pas de Session ID stable, ce qui crée un risque de mélange de contexte entre deux voyageurs différents. L'absence de mémoire est ici un garde-fou, pas une limitation.

---

## 📈 Résultats & Métriques Cibles

| Métrique | Avant | Cible | Mesure |
|----------|-------|-------|--------|
| Temps de traitement | 5-10 min | < 30 sec | Timestamps N8N |
| Taux d'automatisation | 0% | > 80% | Ratio IF1 True/False |
| Précision des réponses | — | > 95% | Revue hebdomadaire structurée (grille 4 critères) |
| Utilisation Perplexity | — | < 20% | Logs appels outil |
| Faux positifs hors-périmètre | — | < 5% | Revue alertes propriétaire |

---

## ✅ Critères d'Acceptation (Pass/Fail)

| Critère | Condition de succès | Statut |
|---------|---------------------|--------|
| Détection emails | 100% des emails express@airbnb.com détectés | PASS |
| Filtrage correct | 0 email système déclenche le workflow | PASS |
| Périmètre IN SCOPE | 9/10 questions IN SCOPE répondues correctement | PASS |
| Périmètre OUT OF SCOPE | 5/5 questions hors périmètre → alerte propriétaire | PASS |
| Temps de traitement | < 30 sec sur 95% des tests | PASS |
| Edge case urgence | Question sécurité → alerte immédiate sans réponse voyageur | PASS |
| Edge case langue | Question en anglais → réponse en anglais | PASS |
| Fallback Perplexity | Si Google Docs vide → Perplexity appelé et réponse envoyée | PASS |
| Mémoire désactivée | 2 emails du même voyageur traités indépendamment | PASS |

---

## 🗺 Roadmap

| Version | Évolution | Horizon |
|---------|-----------|---------|
| V1.1 | Activation Perplexity en production + monitoring précision | Court terme |
| V1.1 | Suppression nœud IF via Filter Query OData avancé | Court terme |
| V1.2 | Journal des réponses envoyées (Google Sheets — grille 4 critères) | Moyen terme |
| V2.0 | Extension multi-logements (Google Docs par bien) | Long terme |
| V2.0 | Réponse directe dans messagerie Airbnb si API disponible | Long terme |

---

## 👨‍💼 Compétences PM Démontrées

- **Vision → Exécution :** Problème identifié → Architecture → MVP en production (1 session). Périmètre défini avant de coder, pas après.
- **Arbitrage technique documenté :** Chaque décision (autonomie, mémoire, fallback) est argumentée et tracée — pas de choix par défaut.
- **Pensée systémique :** Edge cases identifiés avant mise en prod (urgences, langues étrangères, emails multiples, Google Docs vide).
- **Gouvernance IA :** Double garde-fou sur les actions irréversibles — principe appliqué dès la conception, pas ajouté en correctif.
- **Outils :** N8N Cloud, OpenRouter (GPT-4o), Perplexity API, Google Docs, Microsoft Outlook.

---

## 🤝 Contact
Portfolio : [github.com/yanisse-kemel] | LinkedIn : [linkedin.com/in/yanisse-kemel] | ✉️ yanisse@exemple.fr

*Projet personnel — usage privé.*
