# UML Factory --- Documentation du projet et préparation de la soutenance orale

> **Projet :** UML Factory\
> **Dépôt GitHub :** https://github.com/NayerAli/uml-factory\
> **Sujet :** transformation de diagrammes PlantUML en code source avec
> l'aide d'un modèle d'intelligence artificielle\
> **Format de soutenance conseillé :** 10 à 12 minutes, à deux
> présentateurs\
> **Public :** jury de génie logiciel / informatique

------------------------------------------------------------------------

## 1. Présentation générale

UML Factory est un outil qui transforme un diagramme de classes décrit
en **PlantUML** en fichiers de code source. L'utilisateur choisit le
diagramme, un ou plusieurs langages cibles et le dossier de sortie. Un
modèle d'intelligence artificielle interprète le diagramme, prépare un
plan de fichiers, puis rédige le code.

Le projet ne repose pas sur une simple substitution de texte, des
expressions régulières ou des gabarits de code pour produire les classes
: **la génération du code est confiée au modèle d'IA**. Le programme
encadre cependant ce travail par une architecture d'agents, des règles
propres à chaque langage et plusieurs étapes de vérification.

À la fin du traitement, UML Factory compare le code obtenu au diagramme
initial et présente un verdict : - **Conforme** : les éléments attendus
sont retrouvés ; - **Partiellement conforme** : certains éléments
manquent ; - **Non conforme** : le code ne représente pas suffisamment
le diagramme ; - **Non vérifié** : le diagramme ou la vérification n'a
pas pu être exploité.

L'objectif général est donc de réduire le travail de traduction d'un
modèle UML vers plusieurs langages, tout en donnant à l'utilisateur des
indications sur la qualité du résultat.

### Problématique

**Comment automatiser le passage d'un diagramme UML vers du code source
dans plusieurs langages, tout en conservant une architecture extensible
et en contrôlant le résultat produit par une IA ?**

### Objectifs du projet

1.  Lire un diagramme PlantUML fourni par l'utilisateur.
2.  Générer le code correspondant dans un ou plusieurs langages.
3.  Séparer le noyau du programme des spécificités de chaque langage.
4.  Suivre la génération en direct.
5.  Vérifier autant que possible la syntaxe, la compilation et la
    conformité au diagramme.
6.  Permettre l'ajout de nouveaux langages sans modifier le coordinateur
    ni la CLI.

------------------------------------------------------------------------

## 2. Fonctionnement général

Le parcours utilisateur est volontairement simple :

1.  L'utilisateur lance la commande `python -m umlfactory.cli`.
2.  Il sélectionne un fichier `.puml`.
3.  Il choisit un ou plusieurs langages cibles.
4.  Il indique le dossier dans lequel enregistrer les fichiers.
5.  Le coordinateur répartit le travail vers les agents de langage
    disponibles.
6.  Chaque agent demande au modèle d'IA de planifier et de générer les
    fichiers.
7.  Le code est contrôlé ; si des erreurs sont détectées, une boucle de
    correction peut être lancée.
8.  Les fichiers sont écrits dans le dossier choisi.
9.  Un rapport indique les avertissements et le verdict de conformité.

Le programme peut aussi recevoir les paramètres directement en ligne de
commande, sans passer par les menus interactifs.

### Schéma simplifié du processus

``` mermaid
flowchart TD
    A[Utilisateur] --> B[CLI UML Factory]
    B --> C[Lecture du diagramme PlantUML]
    C --> D[Coordinateur A2A]
    D --> E[Découverte des agents de langage]
    E --> F[Agent Java / Python / PHP / autre]
    F --> G[Modèle d'intelligence artificielle]
    G --> H[Plan des fichiers et génération du code]
    H --> I[Vérification et corrections]
    I --> J[Comparaison avec le diagramme]
    J --> K[Écriture des fichiers via MCP]
    K --> L[Rapport de conformité]
```

------------------------------------------------------------------------

## 3. Architecture technique

UML Factory est organisé autour de trois grandes catégories de
composants : un serveur MCP, un coordinateur et des agents générateurs
spécialisés.

### 3.1 Le serveur MCP

Le serveur MCP fournit les outils d'accès aux fichiers du projet. Il ne
réalise pas lui-même l'analyse UML.

Il expose principalement deux opérations :

-   `read_puml(path)` : lit le contenu d'un fichier PlantUML ;
-   `write_sources(dest_dir, files, overwrite)` : écrit les fichiers
    générés dans le dossier de destination.

Des protections sont prévues : - seuls certains formats de diagramme
sont acceptés (`.puml`, `.plantuml`, `.pu`, `.uml`) ; - le fichier
PlantUML doit contenir une déclaration `@startuml` ; - les chemins de
sortie sont contrôlés pour empêcher une écriture en dehors du dossier
choisi ; - les conflits avec des fichiers existants sont refusés, sauf
si l'utilisateur autorise explicitement leur remplacement ; - des
limites de taille et de nombre de fichiers sont appliquées.

**À retenir :** MCP donne aux composants une manière contrôlée de lire
le diagramme et d'écrire les sources produites.

### 3.2 Le coordinateur A2A

Le coordinateur reçoit la demande de génération et orchestre le travail.

Ses responsabilités sont notamment : - récupérer le diagramme,
directement ou via le serveur MCP ; - découvrir les agents de langage
disponibles ; - vérifier que les langages demandés sont accessibles ; -
transmettre le travail aux agents concernés ; - suivre les événements de
progression ; - récupérer les fichiers produits et les rapports ; -
demander au serveur MCP d'écrire les fichiers ; - regrouper les
résultats et les erreurs dans un rapport.

Le coordinateur ne contient pas une liste figée de tous les langages. Il
découvre les agents disponibles à partir de leurs **Agent Cards** et de
leurs compétences déclarées. Cela permet d'ajouter un langage sans
devoir réécrire le coordinateur.

### 3.3 Les agents de langage

Il existe un agent par langage cible, par exemple Java, Python ou PHP.

Chaque agent connaît les règles spécifiques de son langage : - son
identifiant et son nom ; - les extensions des fichiers ; - les
conventions de style ; - les règles de découpage du code en fichiers ; -
les outils nécessaires à la vérification ; - les règles utilisées pour
examiner la conformité.

L'agent s'appuie sur un modèle d'IA pour comprendre le diagramme,
planifier les fichiers et rédiger leur contenu. Il effectue ensuite les
contrôles prévus pour son langage et peut renvoyer les erreurs au modèle
afin qu'il corrige les fichiers.

### 3.4 Rôle de MCP et A2A

Ces deux mécanismes ne remplissent pas la même fonction :

  -----------------------------------------------------------------------
  Mécanisme                           Rôle dans UML Factory
  ----------------------------------- -----------------------------------
  **MCP**                             Donner accès aux outils de fichiers
                                      : lecture du PlantUML et écriture
                                      des sources

  **A2A**                             Faire communiquer les agents,
                                      transmettre les tâches et
                                      coordonner leur exécution
  -----------------------------------------------------------------------

On peut retenir une formule simple : **MCP donne accès aux outils ; A2A
organise la coopération entre agents.**

### Vue d'ensemble de l'architecture

``` mermaid
flowchart LR
    U[Utilisateur] --> CLI[CLI]
    CLI --> C[Coordinateur A2A]
    C --> MCP[Serveur MCP]
    MCP -->|Lit le .puml| C
    C --> JAVA[Agent Java]
    C --> PY[Agent Python]
    C --> PHP[Agent PHP]
    JAVA --> IA[API du modèle IA]
    PY --> IA
    PHP --> IA
    JAVA --> C
    PY --> C
    PHP --> C
    C -->|Fichiers générés| MCP
    MCP --> OUT[Dossier de sortie]
```

------------------------------------------------------------------------

## 4. La génération du code

La génération ne se limite pas à demander à l'IA de « convertir » le
diagramme. L'agent suit plusieurs étapes.

### Étape 1 --- Comprendre la demande

L'agent reçoit le diagramme PlantUML, le langage cible, le nom de
package éventuel et les consignes complémentaires.

### Étape 2 --- Planifier les fichiers

Le modèle prépare un plan qui indique quels fichiers doivent être créés
et quels éléments UML ils doivent représenter. Cette étape permet
notamment de répartir les classes dans plusieurs fichiers.

### Étape 3 --- Générer les fichiers

Le modèle rédige le contenu des fichiers. Plusieurs fichiers peuvent
être traités en parallèle, dans la limite de la concurrence configurée.

### Étape 4 --- Contrôler le résultat

L'agent effectue des contrôles propres au langage. Selon le langage et
les outils installés, il peut également compiler ou charger le code dans
un processus de vérification.

### Étape 5 --- Corriger les erreurs

Quand les contrôles détectent des erreurs, les informations
correspondantes sont renvoyées au modèle. Le projet prévoit au maximum
deux tours de correction dans le pipeline documenté.

### Étape 6 --- Évaluer la conformité UML

Le système compare les éléments du code au diagramme : classes,
interfaces, énumérations, héritage, réalisation d'interfaces, attributs,
opérations et relations.

Le verdict final est produit par le modèle à partir du diagramme, du
code et des constats de vérification.

------------------------------------------------------------------------

## 5. Vérification : ce que le projet contrôle réellement

La vérification est un point important à présenter au jury, car elle
distingue UML Factory d'un simple générateur de code.

### Contrôles documentés

  -----------------------------------------------------------------------
  Contrôle                            Exemple / objectif
  ----------------------------------- -----------------------------------
  Contrôles statiques                 Vérifier certains éléments de
                                      syntaxe et de cohérence propres au
                                      langage

  Compilation ou chargement           Java : `javac`, puis exécution d'un
                                      petit programme de test ; Python :
                                      import des modules ; les langages
                                      ajoutés peuvent fournir leur
                                      commande de vérification

  Comparaison UML/code                Comparer les noms des classes,
                                      interfaces, énumérations, attributs
                                      et opérations, ainsi que les
                                      relations

  Correction                          Renvoyer les erreurs au modèle pour
                                      tenter de corriger les fichiers

  Verdict                             Résumer le niveau de conformité au
                                      diagramme
  -----------------------------------------------------------------------

Pour Java et Python, la comparaison utilise notamment la réflexion et
tente d'instancier les classes concrètes avec des valeurs par défaut.
Pour un langage ajouté, le mécanisme décrit dans le README recherche les
noms dans les fichiers.

### Limites à connaître et à expliquer

La vérification n'est pas une preuve complète de correction.

Le README précise que le système ne vérifie pas : - le comportement
métier des méthodes : elles ne sont pas appelées pour tester leur
logique ; - les types exacts des attributs ; - les paramètres et les
valeurs de retour ; - la visibilité des éléments ; - le sens et le type
exact de chaque relation.

Une relation peut notamment être repérée parce qu'une classe désigne une
autre classe par un champ ou une propriété, sans que tous les détails
sémantiques soient contrôlés.

Le verdict de conformité est rendu par le modèle : il s'appuie sur les
contrôles automatiques, mais reste un avis et non une preuve
mathématique.

### Sécurité de l'exécution du code généré

Le code peut être exécuté pendant les contrôles. Le projet prévoit
plusieurs précautions : - utilisation d'un dossier temporaire ; -
environnement nettoyé, sans clé API ni variables du fichier `.env` ; -
analyse préalable de certains appels sensibles liés aux processus, au
réseau, aux fichiers ou à l'environnement ; - délai maximal de
vérification documenté de 10 secondes ; - si la vérification ne peut pas
être effectuée, le résultat est signalé comme non vérifié.

**Limite importante :** le README indique explicitement qu'il n'y a pas
d'isolation système de type Docker. Il ne faut donc pas présenter ce
mécanisme comme un bac à sable totalement isolé.

------------------------------------------------------------------------

## 6. L'extensibilité : ajouter un nouveau langage

L'évolutivité est l'un des principes importants du projet.

Pour ajouter un langage, UML Factory peut préparer une spécification de
langage, vérifier les outils disponibles, tester un agent et créer un
fichier dédié dans `umlfactory/generators/`.

Le processus documenté comprend les étapes suivantes :

1.  **Brouillon :** le modèle rédige les règles du langage sous forme de
    données, sans écrire de code Python.
2.  **Vérification des outils :** le programme recherche les
    compilateurs ou outils nécessaires sur la machine.
3.  **Validation de la commande :** la commande de vérification est
    présentée et doit être autorisée.
4.  **Test de l'agent :** l'agent convertit des diagrammes de référence,
    puis le code est contrôlé.
5.  **Création de l'agent :** un fichier
    `umlfactory/generators/<langage>.py` est créé.

Le seuil de validation documenté est une moyenne de **75/100** sur le
corpus de test. En dessous, le langage est classé comme expérimental ;
l'utilisateur décide s'il souhaite le conserver.

Pour une intégration manuelle, le principe est de créer un seul fichier
qui décrit les règles du langage et les outils nécessaires. Le
coordinateur et la CLI n'ont pas à être modifiés.

### Pourquoi est-ce intéressant ?

Cette organisation évite de mélanger les règles de Java, Python, PHP ou
d'un autre langage dans un seul gros programme. Le noyau reste commun et
chaque agent porte les particularités de sa cible.

------------------------------------------------------------------------

## 7. Technologies et outils

  -----------------------------------------------------------------------
  Technologie / outil                 Utilisation dans le projet
  ----------------------------------- -----------------------------------
  **Python 3.13**                     Langage principal du projet,
                                      version indiquée dans le README

  **PlantUML**                        Description textuelle du diagramme
                                      UML d'entrée

  **MCP**                             Accès aux outils de lecture et
                                      d'écriture de fichiers

  **A2A**                             Communication et orchestration des
                                      agents

  **OpenAI / API compatible**         Fournisseur possible du modèle d'IA

  **Anthropic**                       Autre fournisseur possible du
                                      modèle d'IA

  **Java / Python / PHP**             Langages générateurs présents dans
                                      le dépôt décrit par le README

  **`javac` / `java`**                Compilation et test du code Java,
                                      si le JDK est installé
  -----------------------------------------------------------------------

Le README précise qu'un serveur local compatible avec l'API OpenAI, par
exemple Ollama ou LM Studio, peut aussi être configuré à l'aide de
`OPENAI_API_URL` et `OPENAI_MODEL`, sans clé API si le serveur local
n'en exige pas.

------------------------------------------------------------------------

## 8. Installation et lancement

### Prérequis

-   Python 3.13, version testée et indiquée par le README ;
-   Git ;
-   une configuration de modèle d'IA : OpenAI, Anthropic ou serveur
    local compatible ;
-   pour vérifier le Java : un JDK contenant `javac` et `java`.

### Récupérer le projet

``` bash
git clone https://github.com/NayerAli/uml-factory.git
cd uml-factory
```

### Installer les dépendances

``` bash
python -m pip install -r requirements.txt
```

### Configurer le modèle

Sous Windows :

``` bat
copy .env.example .env
```

Sous Linux ou macOS :

``` bash
cp .env.example .env
```

Ouvrir ensuite `.env` et renseigner la configuration choisie.

Exemple pour une API OpenAI ou compatible :

``` env
OPENAI_API_URL=https://api.openai.com/v1
OPENAI_API_KEY=votre_cle_api
OPENAI_MODEL=nom_du_modele
```

Exemple pour Anthropic :

``` env
ANTHROPIC_API_KEY=votre_cle_api
```

Ne jamais publier le fichier `.env` ni sa clé API.

### Lancer l'application

``` bash
python -m umlfactory.cli
```

La CLI démarre automatiquement les composants locaux nécessaires lorsque
la configuration le permet, puis guide l'utilisateur dans le choix du
diagramme, des langages et du dossier de sortie.

### Exemple sans menus interactifs

``` bash
python -m umlfactory.cli examples/model.puml --lang java -o sortie
```

Générer dans deux langages :

``` bash
python -m umlfactory.cli examples/model.puml --lang java python -o sortie
```

Lister les langages disponibles :

``` bash
python -m umlfactory.cli --list-languages
```

Afficher l'aide :

``` bash
python -m umlfactory.cli --help
```

### Tests du projet

``` bash
python -m unittest discover -s tests
```

Les tests Java sont ignorés si `javac` et `java` ne sont pas
disponibles.

------------------------------------------------------------------------

## 9. Structure du dépôt

Voici les principaux éléments de l'organisation du code, d'après le
README :

``` text
uml-factory/
├── .env.example
├── requirements.txt
├── examples/
│   ├── model.puml
│   └── banque_grand.puml
├── tests/
└── umlfactory/
    ├── cli.py
    ├── deploy.py
    ├── coordinator.py
    ├── skills.py
    ├── providers.py
    ├── config.py
    ├── verify.py
    ├── umlmodel.py
    ├── languages.py
    ├── eval_corpus.py
    ├── mcp_server.py
    ├── generators/
    │   ├── java.py
    │   ├── python.py
    │   ├── php.py
    │   ├── base.py
    │   ├── planning.py
    │   ├── prompts.py
    │   ├── repair.py
    │   └── _declarative.py
    ├── ui/
    │   ├── term.py
    │   ├── progress.py
    │   └── prompts.py
    ├── llm.py
    ├── a2a_common.py
    ├── mcp_client.py
    ├── paths.py
    └── textutil.py
```

  -----------------------------------------------------------------------
  Fichier / dossier                   Responsabilité
  ----------------------------------- -----------------------------------
  `cli.py`                            Point d'entrée utilisateur, menus,
                                      options et progression

  `deploy.py`                         Démarrage et arrêt des composants
                                      de la pile

  `coordinator.py`                    Coordination des agents et
                                      distribution des demandes

  `mcp_server.py`                     Lecture du PlantUML et écriture des
                                      sources

  `generators/`                       Agents spécialisés par langage et
                                      logique commune de génération

  `verify.py`                         Contrôles du code généré

  `umlmodel.py`                       Lecture minimale du modèle UML pour
                                      les contrôles de conformité

  `providers.py` et `llm.py`          Configuration et appels aux modèles
                                      d'IA

  `skills.py` et `languages.py`       Création, contrôle et validation de
                                      nouveaux langages

  `tests/`                            Tests automatisés
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 10. Préparation de la soutenance orale à deux

## Répartition proposée

Durée indicative : **10 à 12 minutes**, suivies des questions du jury.

  Partie                                Présentateur 1   Présentateur 2
  ------------------------------------- ---------------- ----------------
  Introduction et problématique         Oui              
  Objectifs et démonstration générale   Oui              
  Architecture MCP / A2A                Oui              
  Pipeline de génération                                 Oui
  Vérification et sécurité                               Oui
  Ajout d'un langage                                     Oui
  Installation / démonstration          À deux           À deux
  Bilan et conclusion                                    Oui

L'idée est que le premier présentateur explique **ce qu'est le projet et
comment ses composants communiquent**, tandis que le deuxième explique
**comment le code est produit, contrôlé et comment le système peut
évoluer**.

------------------------------------------------------------------------

## 11. Script oral --- Présentateur 1

### Diapositive 1 --- Introduction

> Bonjour, aujourd'hui nous allons vous présenter notre projet UML
> Factory.
>
> UML Factory est un outil qui permet de transformer un diagramme UML
> écrit en PlantUML en code source dans différents langages de
> programmation.
>
> Le principe est de partir d'un modèle de conception et de laisser un
> modèle d'intelligence artificielle produire les fichiers
> correspondants.
>
> Nous allons vous présenter le fonctionnement général, l'architecture
> utilisée, puis les mécanismes de génération et de vérification.

### Diapositive 2 --- Problématique et objectifs

> Dans un projet logiciel, les diagrammes UML servent à représenter la
> structure d'une application avant ou pendant son développement.
>
> Cependant, passer du diagramme au code demande du temps et peut
> entraîner des oublis ou des différences entre la conception et
> l'implémentation.
>
> Notre problématique est donc la suivante : comment automatiser cette
> transformation, prendre en charge plusieurs langages et vérifier que
> le résultat reste cohérent avec le diagramme initial ?
>
> Pour répondre à cette problématique, UML Factory utilise une
> architecture modulaire composée de plusieurs agents spécialisés.

### Diapositive 3 --- Fonctionnement général

> L'utilisateur commence par sélectionner un fichier PlantUML, puis
> choisit un ou plusieurs langages et un dossier de destination.
>
> Le programme transmet ensuite la demande à un coordinateur. Celui-ci
> découvre les agents disponibles et délègue la génération à l'agent
> correspondant au langage choisi.
>
> Les agents utilisent un modèle d'intelligence artificielle pour
> préparer les fichiers et écrire le code.
>
> Enfin, le résultat est contrôlé et les fichiers sont enregistrés dans
> le dossier demandé.

### Diapositive 4 --- Architecture MCP et A2A

> L'architecture repose sur trois composants principaux.
>
> Le premier est le serveur MCP. Son rôle est de fournir des outils pour
> lire le diagramme PlantUML et écrire les fichiers générés.
>
> Le deuxième est le coordinateur A2A. Il reçoit la demande, découvre
> les agents disponibles et distribue le travail.
>
> Le troisième composant correspond aux agents de langage. Chaque agent
> est spécialisé dans une cible comme Java, Python ou PHP.
>
> Il est important de distinguer MCP et A2A : MCP permet d'accéder aux
> outils, tandis qu'A2A permet aux agents de communiquer et de se
> répartir les tâches.

### Transition vers le présentateur 2

> Nous avons vu comment l'utilisateur lance une génération et comment
> les différents composants communiquent. Je vais maintenant laisser la
> parole à mon binôme, qui va expliquer plus précisément comment le code
> est généré, vérifié et comment il est possible d'ajouter un nouveau
> langage.

------------------------------------------------------------------------

## 12. Script oral --- Présentateur 2

### Diapositive 5 --- Pipeline de génération

> Chaque agent suit plusieurs étapes.
>
> Tout d'abord, le modèle analyse le diagramme et prépare un plan des
> fichiers à créer.
>
> Ensuite, il génère le contenu de chaque fichier. La génération de
> plusieurs fichiers peut être réalisée en parallèle.
>
> Une fois le code produit, l'agent lance les contrôles disponibles pour
> le langage concerné.
>
> Si des erreurs sont détectées, elles peuvent être renvoyées au modèle
> pour qu'il corrige les fichiers. Le projet prévoit jusqu'à deux tours
> de correction.
>
> L'objectif est donc de ne pas accepter directement la première réponse
> du modèle, mais de contrôler le résultat.

### Diapositive 6 --- Vérification et conformité

> La vérification est un élément essentiel de UML Factory.
>
> Le système contrôle notamment certains aspects de la syntaxe, la
> compilation ou le chargement du code lorsque les outils nécessaires
> sont disponibles.
>
> Il compare également les éléments du code avec le diagramme : les
> classes, les attributs, les opérations, l'héritage et les relations.
>
> À la fin, le modèle rend un verdict : conforme, partiellement
> conforme, non conforme ou non vérifié.
>
> Il faut cependant préciser que cette vérification a des limites. Elle
> ne teste pas le comportement métier des méthodes et ne contrôle pas
> tous les détails des types ou des relations.
>
> Le verdict est donc une indication de conformité, pas une garantie
> absolue que le programme fonctionne correctement.

### Diapositive 7 --- Extensibilité

> Un autre point important du projet est la possibilité d'ajouter de
> nouveaux langages.
>
> Chaque langage possède son propre fichier d'agent dans le dossier
> `generators`.
>
> Pour ajouter un langage, le système prépare ses règles, vérifie les
> outils nécessaires et teste l'agent sur des diagrammes de référence.
>
> Le seuil de validation indiqué dans le projet est de 75 sur 100 en
> moyenne.
>
> Cette organisation permet d'ajouter un langage sans modifier le
> coordinateur ni la ligne de commande principale.
>
> C'est un avantage important pour faire évoluer le projet.

### Diapositive 8 --- Démonstration

> Pour illustrer le fonctionnement, nous pouvons prendre un diagramme de
> classes représentant, par exemple, une petite application bancaire.
>
> Nous lançons la commande, sélectionnons le diagramme, puis demandons
> une génération en Java ou en Python.
>
> Le programme affiche la progression, génère les fichiers, effectue les
> contrôles disponibles et indique le résultat de la vérification.
>
> Nous pouvons ensuite ouvrir les fichiers produits et les comparer au
> diagramme initial.

### Diapositive 9 --- Bilan et conclusion

> Pour conclure, UML Factory propose une automatisation du passage du
> diagramme UML au code source.
>
> Ses principaux points forts sont la prise en charge de plusieurs
> langages, l'architecture distribuée avec MCP et A2A, la génération
> assistée par IA et les contrôles de conformité.
>
> Le projet présente aussi des limites, notamment parce que les
> vérifications ne garantissent pas le comportement fonctionnel du
> programme.
>
> Des améliorations pourraient consister à approfondir les tests
> fonctionnels, à renforcer les contrôles sémantiques et à intégrer
> davantage de langages.
>
> Merci pour votre attention. Nous sommes maintenant prêts à répondre à
> vos questions.

------------------------------------------------------------------------

# 13. Démonstration pratique à préparer

La démonstration doit être testée avant la soutenance, notamment parce
que la génération dépend du fournisseur de modèle et de la configuration
de la machine.

### Scénario conseillé

1.  Ouvrir un diagramme d'exemple du dossier `examples/`.
2.  Montrer rapidement les classes et les relations représentées.
3.  Lancer la CLI.
4.  Choisir Java ou Python.
5.  Choisir le dossier `sortie/`.
6.  Observer la progression de la génération.
7.  Montrer les fichiers générés.
8.  Présenter le résultat de vérification et le verdict de conformité.

### Commande de démonstration

``` bash
python -m umlfactory.cli examples/model.puml --lang java -o sortie
```

Pour générer deux langages :

``` bash
python -m umlfactory.cli examples/model.puml --lang java python -o sortie
```

### Plan de secours

Avant le passage devant le jury : - vérifier que les dépendances sont
installées ; - vérifier le fichier `.env` et la disponibilité du modèle
; - vérifier que les ports locaux nécessaires sont libres ; - lancer une
génération complète à l'avance ; - conserver un exemple déjà généré dans
un dossier de secours ; - préparer des captures d'écran de la
progression et du résultat.

Ne pas présenter un résultat de démonstration préparé à l'avance comme
une génération qui vient d'être exécutée en direct.

------------------------------------------------------------------------

# 14. Questions possibles du jury et réponses

### 1. Pourquoi utiliser une IA plutôt qu'un générateur basé sur des templates ?

L'IA peut interpréter le diagramme et produire du code adapté au langage
et aux consignes. Le projet ne se limite pas à remplir des modèles de
fichiers prédéfinis. En contrepartie, il faut contrôler les résultats,
car une IA peut produire des erreurs.

### 2. Quelle est la différence entre MCP et A2A ?

MCP fournit des outils accessibles aux composants, notamment pour lire
et écrire les fichiers. A2A sert à faire communiquer les agents et à
leur transmettre des tâches.

### 3. Pourquoi avoir un agent par langage ?

Les langages ont des syntaxes, des conventions et des outils de
compilation différents. Un agent spécialisé permet de conserver ces
règles séparées et de faire évoluer un langage sans modifier tout le
système.

### 4. Comment le système sait-il quels langages sont disponibles ?

Le coordinateur découvre les agents joignables et consulte leurs Agent
Cards, qui décrivent leurs compétences et les langages qu'ils prennent
en charge.

### 5. Est-ce que le code généré est forcément correct ?

Non. Le projet effectue plusieurs contrôles et peut demander des
corrections au modèle, mais cela ne garantit pas le bon fonctionnement
métier du programme.

### 6. Que se passe-t-il si le compilateur n'est pas installé ?

Le code peut tout de même être généré, mais la compilation ne peut pas
être vérifiée. Le programme signale cette situation au lieu de présenter
le contrôle comme réussi.

### 7. Comment ajouter un nouveau langage ?

Il faut créer ou faire préparer un fichier d'agent dans
`umlfactory/generators/`, définir ses règles et ses outils, puis valider
l'agent avec le corpus de diagrammes prévu.

### 8. Pourquoi utiliser un coordinateur ?

Il évite que la CLI connaisse les détails de chaque langage. Il
centralise la découverte des agents, la distribution des tâches et la
récupération des résultats.

### 9. Le système fonctionne-t-il sans Internet ?

Cela dépend du fournisseur de modèle. Le README indique qu'un serveur
local compatible avec l'API OpenAI, comme Ollama ou LM Studio, peut être
utilisé. Avec un modèle local correctement configuré, les appels au
modèle peuvent rester sur la machine. Il faut toutefois vérifier les
dépendances et la configuration utilisées.

### 10. Quelle est la principale limite du projet ?

La principale limite est que la conformité structurelle ne signifie pas
que le comportement fonctionnel est correct. Le projet compare certains
éléments du code au diagramme, mais ne teste pas les règles métier des
méthodes.

------------------------------------------------------------------------

# 15. Conseils pour réussir la présentation à deux

-   Ne lisez pas les diapositives : expliquez le fonctionnement avec vos
    propres mots.
-   Le premier présentateur doit maîtriser la problématique, le parcours
    utilisateur et l'architecture.
-   Le deuxième doit maîtriser le pipeline de génération, les
    vérifications et l'extensibilité.
-   Utilisez le même exemple PlantUML pendant toute la présentation pour
    garder un fil conducteur.
-   Pendant la démonstration, montrez au moins un fichier généré et le
    rapport de conformité.
-   Soyez transparents sur les limites : cela montre que vous comprenez
    les choix techniques du projet.
-   Entraînez-vous à faire la transition entre les deux intervenants
    sans rupture.

## Répartition temporelle indicative

  Séquence                                                  Durée
  -------------------------------------------------- ------------
  Introduction et problématique --- présentateur 1       1 min 30
  Objectifs et fonctionnement --- présentateur 1         1 min 30
  Architecture MCP / A2A --- présentateur 1                 2 min
  Génération et vérification --- présentateur 2             2 min
  Extensibilité --- présentateur 2                          1 min
  Démonstration --- à deux                                  2 min
  Bilan et conclusion --- présentateur 2                    1 min
  **Total**                                            **11 min**

------------------------------------------------------------------------

## 16. Sources et périmètre de cette documentation

Cette documentation a été construite à partir du dépôt GitHub et de ses
fichiers accessibles, en particulier :

-   [README.md](https://github.com/NayerAli/uml-factory/blob/main/README.md)
    --- présentation, installation, commandes, configuration et
    structure du dépôt ;
-   [coordinator.py](https://github.com/NayerAli/uml-factory/blob/main/umlfactory/coordinator.py)
    --- orchestration A2A, découverte des agents et délégation ;
-   [mcp_server.py](https://github.com/NayerAli/uml-factory/blob/main/umlfactory/mcp_server.py)
    --- lecture PlantUML et écriture des fichiers ;
-   [generators/base.py](https://github.com/NayerAli/uml-factory/blob/main/umlfactory/generators/base.py)
    --- pipeline de génération, vérification et correction.


