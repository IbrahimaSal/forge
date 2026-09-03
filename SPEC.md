# Spécification des Exigences Logicielles (SRS)
## Fork "IA4dev Code" — basé sur Continue.dev

| | |
|---|---|
| **Document** | SRS-IA4DEV-CODE-001 |
| **Version** | 0.3 (draft) |
| **Statut** | Brouillon — aucune implémentation ne démarre avant approbation |
| **Norme de référence** | ISO/IEC/IEEE 29148:2018 (succède à IEEE 830-1998) |

**Changement v0.3** : pivot d'architecture — Phase 1 = CLI/TUI (fork de `extensions/cli` + `core/`), Phase 2 (différée) = extension VS Code. Décision motivée par l'investigation du 2026-09-01 : `extensions/cli` réutilise déjà le moteur `multiEdit` de `core/` sans aucune dépendance à `ApplyManager`/`extensions/vscode/`, éliminant structurellement la source de fragilité identifiée plutôt que de la contourner.

Convention de mots-clés (RFC 2119) : **DOIT/SHALL** = exigence obligatoire ; **DEVRAIT/SHOULD** = recommandé, dérogation justifiable ; **PEUT/MAY** = optionnel.

---

## 1. Introduction

### 1.1 Objet (Purpose)
Ce document spécifie les exigences fonctionnelles et non-fonctionnelles d'un outil CLI/TUI, fork de Continue.dev (`extensions/cli` + `core/`), destiné à remplacer l'usage de Mistral Code Enterprise pour l'utilisateur. Il sert de base contractuelle avant le démarrage de toute implémentation.

### 1.2 Périmètre (Scope)
Le système spécifié couvre : (a) la fiabilisation du mécanisme d'édition de fichiers, (b) l'activation d'un système de skills, (c) une couche d'orchestration de workflows, (d) l'intégration de l'authentification IA4dev. 

**Phasage** :
- **Phase 1 (spécifiée ici)** : outil CLI/TUI (fork de `extensions/cli` + `core/`), utilisable en parallèle de VS Code (l'utilisateur édite dans son IDE habituel pendant que l'agent CLI tourne dans un terminal), sur le modèle de Claude Code CLI.
- **Phase 2 (différée, hors périmètre de ce document)** : extension VS Code, réutilisant `core/` et les mêmes domaines EDIT/SKILL/WKF/AUTH, avec une couche d'intégration éditeur à spécifier séparément le moment venu.

Aucun autre IDE n'est couvert. La distribution publique (marketplace) est hors périmètre.

### 1.3 Définitions, acronymes, abréviations

| Terme | Définition |
|---|---|
| SRS | Software Requirements Specification |
| REQ | Identifiant d'exigence individuelle |
| Apply | Mécanisme Continue qui applique a posteriori un bloc de code généré en chat à un fichier |
| `multiEdit` | Outil agent Continue de type search/replace exact (`old_string`/`new_string`) |
| Skill | Unité d'instructions chargée à la demande par le modèle (fichier `SKILL.md`) |
| Workflow | Séquence orchestrée d'étapes (skills et/ou tools) avec état partagé |
| IA4dev | Infrastructure/offre de l'utilisateur, backend d'authentification existant |
| TBD | To Be Determined — exigence non encore figée, trackée en Annexe A |

### 1.4 Références
- Rapport d'investigation du code source `continuedev/continue` — domaine EDIT/AUTH/SKILL/WKF, version VS Code (session du 2026-08-28/29).
- Rapport d'investigation `extensions/cli` — maturité, édition de fichiers, auth, couplage à `core/` (session du 2026-09-01).
- `SPEC.md` v0.1 (Design Doc) et v0.2 (première version SRS, base VS Code), remplacées par le présent document.
- Documentation Continue : télémétrie, indexation codebase.

### 1.5 Vue d'ensemble du document
Section 2 décrit le produit globalement. Section 3 liste les exigences vérifiables individuellement, numérotées par domaine (EDIT, SKILL, WKF, AUTH, NFR). Section 4 fournit la matrice de traçabilité exigence → critère de vérification. Annexe A liste les points TBD bloquants.

---

## 2. Description générale

### 2.1 Perspective du produit
Le produit est un fork indépendant de Continue.dev (Apache 2.0) — plus précisément de `extensions/cli` (package `@continuedev/cli`, TUI Ink/React) et de `core/` dont il dépend — et non un patch du binaire propriétaire Mistral Code Enterprise. C'est un outil terminal (TUI interactif + mode headless scriptable), pas une extension d'éditeur, sur le modèle de Claude Code CLI. Il remplace Mistral Code Enterprise dans l'usage quotidien de l'utilisateur.

### 2.2 Fonctions du produit
1. Édition de code assistée par IA, fiable (via `core/edit/searchAndReplace/`, sans couche `ApplyManager` — celle-ci n'existe pas dans cette base).
2. Chargement à la demande de skills réutilisables.
3. Exécution de workflows multi-étapes.
4. Authentification via l'infrastructure IA4dev (à construire : le flux hub/WorkOS d'origine a été retiré de `extensions/cli` par l'upstream en juillet 2026, remplacé par une saisie directe de clé API).
5. Autocomplete et chat sur modèles Mistral (fonctions héritées de Continue, non respécifiées ici — cf. 2.4 hors périmètre de modification).

### 2.3 Caractéristiques utilisateur
Utilisateur unique identifié : développeur, opérateur du fork, propriétaire de l'infrastructure IA4dev. Niveau technique : capable de lire du TypeScript, de builder un outil CLI Node.js, d'opérer un backend d'auth. Usage prévu : terminal, en parallèle d'un IDE (VS Code ou autre) ouvert séparément pour la lecture/relecture visuelle du code.

### 2.4 Contraintes
- C-1 : Le code de base DOIT rester un fork rebasable de `extensions/cli` + `core/` du dépôt `continuedev/continue` (pas de réécriture complète, pas de dépendance à `extensions/vscode/`).
- C-2 : Aucune modification n'est apportée aux fonctions qui ne sont pas listées en 2.2 (autocomplete, indexation locale @codebase) sauf régression constatée.
- C-3 : Le fichier `.vsix` Mistral Code existant ne DOIT PAS être utilisé comme base de code (corrompu, minifié, CGU restrictives non vérifiées).
- C-4 : La Phase 2 (extension VS Code) NE DOIT PAS être entamée avant que les domaines EDIT/AUTH/SKILL/WKF de la Phase 1 soient livrés et validés.

### 2.5 Hypothèses et dépendances
- A-1 : Le backend IA4dev (page de login web + mécanisme de retour de token) existe déjà et est opérationnel (confirmé par l'utilisateur).
- A-2 : L'accès aux modèles Mistral (Codestral, Devstral/Large) via API est disponible et fonctionnel indépendamment de ce projet.
- A-3 : Le dépôt `continuedev/continue` reste accessible publiquement sous licence Apache 2.0 pendant la durée du projet.

---

## 3. Exigences spécifiques

### 3.1 Exigences d'interface externe

| ID | Exigence | Priorité |
|---|---|---|
| REQ-IF-010 | Le système DOIT exposer une commande (`login`) qui ouvre le navigateur par défaut de l'utilisateur vers l'URL de login IA4dev. | Haute |
| REQ-IF-020 | Le système DOIT démarrer un serveur HTTP local temporaire pour recevoir le bearer token transmis par IA4dev lors de la redirection post-login (mécanisme précis en TBD-6, cf. Annexe A). | Haute |
| REQ-IF-030 | Le système DOIT communiquer avec les endpoints API Mistral (Codestral/Devstral) pour les fonctions chat, edit et autocomplete. | Haute |

### 3.2 Exigences fonctionnelles

#### 3.2.1 Domaine EDIT — Fiabilisation de l'édition de fichiers

Contexte historique : dans la version VS Code de Continue, le mécanisme « chat Apply » (`extensions/vscode/src/apply/ApplyManager.ts` → `streamLazyApply` → `core/diff/streamDiff.ts`) est la cause identifiée des échecs récurrents d'édition (matching heuristique ligne-par-ligne non déterministe). **Ce risque est structurellement absent en Phase 1** : `extensions/cli` ne dépend d'aucun de ces fichiers (confirmé par investigation du 2026-09-01, grep exhaustif sans résultat) et édite déjà les fichiers via `core/edit/searchAndReplace/executeMultiFindAndReplace` (`extensions/cli/src/tools/multiEdit.ts`). Les exigences ci-dessous portent donc sur le renforcement de ce chemin existant, pas sur le contournement d'un mécanisme défaillant.

| ID | Exigence | Priorité |
|---|---|---|
| REQ-EDIT-010 | Toute édition de fichier initiée par le modèle DOIT être réalisée via l'outil `multiEdit` (`extensions/cli/src/tools/multiEdit.ts`, search/replace exact `old_string`/`new_string`). | Haute |
| REQ-EDIT-020 | L'outil `edit.ts` single-edit (`extensions/cli/src/tools/edit.ts`) DEVRAIT être conservé pour les remplacements uniques simples, mais NE DOIT PAS reposer sur un mécanisme de validation différent de `multiEdit`. | Moyenne |
| REQ-EDIT-030 | *(Sans objet en Phase 1 — `ApplyManager` n'existe pas dans `extensions/cli`. Exigence conservée pour rappel lors de la Phase 2.)* | — |
| REQ-EDIT-040 | *(Sans objet en Phase 1 — le raccourci "Quick Edit" est un concept d'éditeur VS Code inline. Reporté à la Phase 2.)* | — |
| REQ-EDIT-050 | Si `multiEdit` échoue (`FindAndReplaceOldStringNotFound` ou `MultipleOccurrences`), le système DOIT renvoyer l'erreur au modèle avec un extrait du fichier actuel suffisant pour permettre une correction. | Haute |
| REQ-EDIT-060 | Le nombre de tentatives automatiques de correction après échec DOIT être plafonné (valeur par défaut : 2 ; configurable). | Moyenne |
| REQ-EDIT-070 | Un échec définitif (après épuisement des tentatives) DOIT produire un message d'erreur explicite affiché à l'utilisateur ; le système NE DOIT JAMAIS laisser un fichier dans un état silencieusement corrompu ou partiellement modifié sans notification. | Haute |
| REQ-EDIT-080 | Le système NE DOIT PAS s'appuyer sur du matching approximatif (fuzzy, ex. Jaro-Winkler) pour `multiEdit`. La fiabilité DOIT reposer sur la fraîcheur du contexte (REQ-EDIT-081) plutôt que sur la tolérance de l'algorithme de recherche — principe aligné sur l'outil Edit de Claude Code (matching strict, discipline de lecture). | Haute |
| REQ-EDIT-081 | Le système DOIT rejeter tout appel `multiEdit` portant sur un fichier qui n'a pas été lu (via l'outil de lecture) au moins une fois dans la session en cours, avec un message demandant explicitement de relire le fichier avant de réessayer. | Haute |
| REQ-EDIT-090 | *(Sans objet en Phase 1 — le CLI est nativement agentique, il n'existe pas de "chat libre sans outils" distinct. Le contrôle de ce que le modèle peut faire passe par `src/permissions/` du CLI, hors périmètre de cette exigence. Reporté à la Phase 2 si un mode chat séparé y est introduit.)* | — |

#### 3.2.2 Domaine SKILL — Skills réutilisables

| ID | Exigence | Priorité |
|---|---|---|
| REQ-SKILL-010 | Le système DOIT charger les fichiers `SKILL.md` (frontmatter YAML avec `name` et `description` obligatoires) présents dans `.continue/skills/` (portée projet) et son équivalent global, via `loadMarkdownSkills` (`extensions/cli/src/util/loadMarkdownSkills.ts`, déjà partagé avec `core/config/markdown/`). | Haute |
| REQ-SKILL-020 | Le contenu complet d'un skill DOIT être chargé à la demande via l'outil `readSkill` (`extensions/cli/src/tools/skills.ts`), le modèle ne recevant par défaut que `name` et `description`. | Haute |
| REQ-SKILL-030 | Un skill ajouté ou modifié DOIT être détecté au démarrage d'une nouvelle session CLI sans nécessiter de reconstruction (`build`) de l'outil. | Moyenne |
| REQ-SKILL-040 | Le système DOIT être livré avec au moins deux skills d'exemple fonctionnels. | Basse |

#### 3.2.3 Domaine WKF — Workflows multi-étapes

| ID | Exigence | Priorité |
|---|---|---|
| REQ-WKF-010 | Le système DOIT fournir un outil (`runWorkflow`) capable d'exécuter une séquence ordonnée d'étapes, chaque étape pouvant être un skill ou un tool existant. | Haute |
| REQ-WKF-020 | Le système DOIT transmettre un état partagé entre les étapes d'un même workflow. | Haute |
| REQ-WKF-030 | Un workflow DOIT être défini par un fichier Markdown avec frontmatter YAML, cohérent avec le format `SKILL.md` déjà existant (liste d'étapes, chacune référençant un skill ou un tool). | Haute |
| REQ-WKF-040 | En cas d'échec d'une étape, le système DOIT arrêter l'exécution du workflow à cette étape et produire un rapport indiquant l'étape en échec et son statut. Les étapes suivantes NE DOIVENT PAS s'exécuter (pas de continuation, pas de rollback automatique). | Haute |
| REQ-WKF-050 | Un workflow d'exemple ("lint → edit → test → commit") DOIT s'exécuter de bout en bout sur un projet de test, avec rapport par étape visible dans le chat. | Moyenne |

#### 3.2.4 Domaine AUTH — Authentification IA4dev

| ID | Exigence | Priorité |
|---|---|---|
Contexte : `extensions/cli` n'a plus de flux de login navigateur — le hub/WorkOS a été retiré par l'upstream en juillet 2026 (commit *"login flow retired after acquisition"*), remplacé par une saisie manuelle de clé API dans `src/onboarding.ts` (`~/.continue/config.yaml`). Il n'y a donc rien à démonter côté auth ; le flux IA4dev est une addition neuve, pas un remplacement.

| ID | Exigence | Priorité |
|---|---|---|
| REQ-AUTH-010 | Le système DOIT proposer, en plus de la saisie manuelle de clé API existante, un mode de connexion « IA4dev » déclenché par une commande dédiée (ex. `cn login --ia4dev`). | Haute |
| REQ-AUTH-020 | Le système DOIT ouvrir le navigateur par défaut vers l'URL de login IA4dev lors du déclenchement de cette commande. | Haute |
| REQ-AUTH-030 | Le système DOIT démarrer un serveur HTTP local (port dynamique ou fixe, cf. TBD-6) avant l'ouverture du navigateur, recevoir le bearer token en paramètre de la requête de callback, puis arrêter le serveur — sans intervention manuelle de copier-coller. | Haute |
| REQ-AUTH-031 | Le serveur local DOIT expirer (timeout) après un délai défini (ex. 5 minutes) si aucun callback n'est reçu, avec message d'erreur explicite dans le TUI. | Moyenne |
| REQ-AUTH-040 | Le système DOIT stocker le bearer token de façon sécurisée sur disque (permissions restrictives, ex. `0600`, dans le répertoire de config utilisateur — pas de `SecretStorage` VS Code, non applicable en CLI). | Haute |
| REQ-AUTH-050 | Une commande `logout` DOIT supprimer le bearer token stocké localement. | Moyenne |
| REQ-AUTH-060 | Après authentification IA4dev, le système NE DOIT PLUS émettre d'appel réseau résiduel vers un domaine `*.continue.dev`. | Haute |

### 3.3 Exigences non-fonctionnelles

| ID | Exigence | Catégorie |
|---|---|---|
| REQ-NFR-010 | La télémétrie PostHog vers continue.dev DOIT être désactivée par défaut (`allowAnonymousTelemetry: false`). | Confidentialité |
| REQ-NFR-020 | Une télémétrie propre à IA4dev, si souhaitée, DOIT faire l'objet d'une exigence séparée avant implémentation. | Confidentialité |
| REQ-NFR-030 | Le modèle par défaut pour chat/edit DOIT être un modèle Mistral (Devstral ou Large) ; l'autocomplete DOIT utiliser Codestral. | Fonctionnel/Config |
| REQ-NFR-040 | L'indexation du codebase DOIT rester locale par défaut (comportement hérité, non modifié). | Confidentialité |
| REQ-NFR-050 | Le code du fork DEVRAIT minimiser les modifications invasives du cœur de Continue au profit des points d'extension/configuration existants, afin de rester rebasable sur les mises à jour upstream. | Maintenabilité |
| REQ-NFR-060 | Le scénario de test EDIT (fichier de +300 lignes, zones non contiguës modifiées) DOIT réussir de façon reproductible sur au moins 20 exécutions consécutives. | Fiabilité |
| REQ-NFR-070 | Le fork DOIT être développé sur une version figée (commit/tag) de `continuedev/continue`, choisie au démarrage de l'implémentation. Un rebase sur l'upstream DOIT être effectué de façon planifiée et volontaire (indicativement tous les 2 à 3 mois), jamais en continu automatique. | Maintenabilité |

---

## 4. Matrice de traçabilité (extrait — à compléter en phase de vérification)

| Exigence | Méthode de vérification | Statut |
|---|---|---|
| REQ-EDIT-010, 020 | Revue de code : `multiEdit`/`edit` restent les seuls chemins d'écriture de fichier déclenchables par le modèle | Non vérifié |
| REQ-EDIT-050, 060, 070 | Test d'intégration : injection d'un `old_string` inexistant, vérification du message renvoyé au modèle et à l'utilisateur | Non vérifié |
| REQ-EDIT-080, 081 | Test d'intégration : appel `multiEdit` sur un fichier non lu dans la session → rejet attendu ; absence de tout chemin de code utilisant une correspondance approximative | Non vérifié |
| REQ-NFR-060 | Test automatisé répété 20x sur fichier de test dédié | Non vérifié |
| REQ-SKILL-010, 020, 030 | Test manuel : dépôt d'un `SKILL.md`, vérification de détection sur nouvelle session | Non vérifié |
| REQ-AUTH-010, 020, 030, 031, 060 | Test manuel du flux `cn login --ia4dev` + capture réseau (absence de trafic `*.continue.dev`) + test du timeout serveur local | Non vérifié |

---

## 5. Annexe A — Points TBD

### A.1 Résolus (sessions du 2026-08-30 et 2026-08-31)

| ID | Décision actée | Exigences mises à jour |
|---|---|---|
| ~~TBD-2~~ | Format workflow = Markdown/YAML façon `SKILL.md` ; échec d'étape = arrêt + rapport, pas de rollback | REQ-WKF-030, REQ-WKF-040 |
| ~~TBD-3~~ | Chat libre conservé, en lecture seule uniquement (pas d'Apply) — *sans objet en Phase 1, cf. REQ-EDIT-090 révisé* | REQ-EDIT-090 |
| ~~TBD-5~~ | Base figée sur un commit/tag `continuedev/continue`, rebase manuel planifié (~2-3 mois) | REQ-NFR-070 |
| ~~TBD-4~~ | Décision (2026-08-31) : on n'investigue pas le bug du fuzzy matching Jaro-Winkler de Continue. On aligne `multiEdit` sur le principe de l'outil Edit de Claude Code — matching strict, pas de tolérance approximative — et on compense par une discipline de lecture obligatoire avant édition. | REQ-EDIT-080, REQ-EDIT-081 |

### A.2 Point rouvert par le pivot CLI (2026-09-01)

TBD-1 (résolu le 2026-08-31 pour un contexte extension VS Code, mécanisme `vscode://`) ne s'applique plus tel quel : un CLI n'a pas d'URI handler `vscode://`. Le pattern standard pour un outil CLI (`gh auth login`, `aws sso login`, etc.) est un **serveur HTTP local** auquel le navigateur redirige après login. Ceci ouvre un nouveau point :

| ID | Description | Exigences affectées | Lot bloqué | Action requise |
|---|---|---|---|---|
| TBD-6 | Le backend IA4dev peut-il rediriger vers une URL `http://127.0.0.1:<port>/callback` après login (au lieu de/en plus de `vscode://`) ? Si oui : le port est-il choisi dynamiquement par le CLI au démarrage (nécessite que IA4dev accepte un port variable transmis en paramètre `redirect_uri`), ou faut-il enregistrer un port fixe à l'avance côté IA4dev ? | REQ-IF-020, REQ-AUTH-030 | AUTH | Utilisateur : vérifier côté infrastructure IA4dev si un `redirect_uri` `http://127.0.0.1:<port>/...` paramétrable est supporté |

### A.3 Encore ouverts (bloquants avant démarrage du lot concerné)

| ID | Bloque |
|---|---|
| TBD-6 | Domaine AUTH uniquement |

Domaines EDIT, SKILL, WKF : aucun blocant, peuvent démarrer immédiatement.

---

## 6. Ordre de livraison proposé

1. Domaine EDIT — impact utilisateur immédiat, débloque la confiance dans l'outil. Prêt à démarrer.
2. Domaine SKILL — activation quasi gratuite de l'existant. Prêt à démarrer.
3. Domaine WKF — le plus structurant. Prêt à démarrer.
4. Domaine AUTH — en attente de la résolution de TBD-6 ; peut être mené en parallèle des trois autres dès que résolu.

---

## 7. Vision Phase 3 (roadmap — non spécifié, à détailler dans une prochaine révision du SRS)

Orientation stratégique actée le 2026-09-03. **Ceci n'est pas une exigence opposable** — à formaliser en REQ vérifiables dans une future version (v0.4+) une fois les Phases 1/2 livrées.

### 7.1 Différenciateur visé : orientation "Harness"
La valeur du produit ne doit pas être positionnée comme "un assistant de code de plus branché sur un modèle", mais comme la qualité du **harnais** autour du modèle : contexte fourni, outils exposés, garde-fous, boucles de vérification. Cohérent avec le domaine EDIT de la Phase 1 — la fiabilité de l'édition était déjà un problème de harnais (discipline de lecture, matching strict, retry informé), pas un problème de modèle. Ce constat devient le fil conducteur produit.

### 7.2 Agnosticisme au modèle
Comme Continue, le système DEVRA accepter n'importe quel modèle/provider, pas uniquement Mistral. `core/llm/` est déjà multi-provider par conception — probablement en grande partie hérité du fork plutôt qu'à construire. La config par défaut Mistral (REQ-NFR-030) doit rester un choix de configuration, jamais une contrainte en dur dans le code.

### 7.3 Écosystème de construction de harnais
Au-delà de livrer UN harnais fixe, envisager un écosystème permettant à l'utilisateur de **construire/personnaliser son propre harnais** : composition de skills/workflows/permissions/outils comme des briques réutilisables et partageables, plutôt qu'une configuration figée par l'éditeur du produit.

### 7.4 Questions ouvertes pour la prochaine révision
- Qu'est-ce qui rend un harnais mesurablement meilleur qu'un autre (métriques, evals) ?
- L'agnosticisme au modèle doit-il être tiré dans la Phase 1 dès maintenant (probablement peu coûteux via `core/llm/`), ou rester strictement Phase 3 ?
- À quoi ressemble concrètement "aider l'utilisateur à construire son harnais" — un outil de scaffolding (`cn harness init`), un format de composition déclaratif, un marketplace de briques (skills/tools/permissions) ?
