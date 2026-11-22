# Rapport de la preuve de concept MedHead

Ce document est le rapport de la preuve de concept (PoC) du système d'intervention
d'urgence en temps réel MedHead. Il répond au livrable « document de reporting » du
document d'exigences : il justifie les technologies retenues, montre comment la PoC
respecte les normes et les principes d'architecture, présente ses résultats et en tire
les enseignements. Dans la démarche TOGAF décrite par la Déclaration des travaux
d'architecture, il sert à l'évaluation de la conformité (phase G) et à l'analyse de
l'écart entre la PoC et la production (phase F).

| Élément | Référence |
| --- | --- |
| Dépôt de code | [TheFatAndFurious/MedHead](https://github.com/TheFatAndFurious/MedHead) |
| Commit de référence | [`ce8dbf4`](https://github.com/TheFatAndFurious/MedHead/commit/ce8dbf4de37e7868b64ddca32df5210ce0705fe8) de `master`, version `0.1.0-SNAPSHOT` |
| Exécution de CI de référence | [Run 45066204052](https://github.com/TheFatAndFurious/MedHead/actions/runs/45066204052) du 22 novembre 2026 sur ce commit : `frontend-ci`, `backend-ci`, `e2e` et `publish` réussis |
| Campagnes de charge | [Résultats du 22 novembre 2026](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/performance/resultats-2026-11-22.md) |
| Décisions d'architecture | [ADR 0001 à 0006](../adr/) |
| Diagrammes | [Contexte](../diagrammes/contexte.md), [conteneurs](../diagrammes/conteneurs.md), [séquence de réservation](../diagrammes/sequence-reservation.md), [déploiement](../diagrammes/deploiement.md) |
| Documents fournis | Exigences pour le développement de la PoC, Principes de l'architecture, Déclaration des travaux d'architecture, Document de définition de l'architecture ([`docs/fournis/`](../docs/fournis/)) |

<!-- TODO Matt : déposer les quatre PDF fournis dans docs/fournis/ et vérifier que le lien ci-dessus les atteint. -->

## Correspondance avec la grille d'évaluation

| Attendu | Section |
| --- | --- |
| Justification des technologies | [2. Justification des technologies](#2-justification-des-technologies) |
| Java imposé par le consortium | [2.1](#21-api--java-imposé-spring-boot-41-et-java-21) |
| Front choisi parmi les possibilités imposées, avec un critère de choix | [2.2](#22-front--sveltekit-par-dérogation-critère-de-choix) |
| Respect des normes et des principes | [3. Respect des normes et des principes](#3-respect-des-normes-et-des-principes) |
| Respect du RGPD | [3.1](#31-rgpd) |
| Respect de l'architecture microservice | [3.2](#32-architecture-microservice) |
| Intégration avec une architecture basée sur les événements | [3.3](#33-intégration-avec-une-architecture-basée-sur-les-événements) |
| Résultats de la PoC | [4. Résultats de la PoC](#4-résultats-de-la-poc) |
| Succès d'un point de vue fonctionnel | [4.1](#41-résultats-fonctionnels) |
| Métriques sur le temps d'exécution de l'API | [4.2](#42-performance-de-lapi) |
| Enseignements de la PoC | [5. Enseignements](#5-enseignements) |

## 1. Contexte, objectifs et périmètre

Le consortium MedHead réunit quatre organisations qui veulent bâtir une plateforme de
soins commune, faite de services à responsabilité unique, découplés, tolérants aux
pannes et communiquant par événements (Document de définition de l'architecture,
p. 3). Le système d'intervention d'urgence en temps réel est la principale
préoccupation des parties prenantes : sa capacité à attribuer à temps un lit dans le
service hospitalier approprié est un risque à lever par une preuve de concept
(Document de définition de l'architecture, p. 16). La feuille de route du consortium
place l'évaluation de cette PoC en tête de ses jalons (même document, p. 11).

### 1.1 Hypothèse à valider

Le document d'exigences (p. 1) formule l'hypothèse suivante : une PoC du système
d'intervention d'urgence en temps réel doit permettre d'améliorer la qualité des
traitements d'urgence, donc de sauver plus de vies, et de gagner la confiance des
utilisateurs quant à la simplicité d'un tel système.

Pour la partie technique, la PoC doit montrer qu'un service peut, pour une urgence
localisée et une spécialité donnée :

- proposer l'hôpital le plus proche qui possède cette spécialité et un lit disponible ;
- y réserver ce lit sans double attribution, même sous des demandes concurrentes ;
- publier un événement exploitable par d'autres systèmes de la plateforme ;
- répondre assez vite pour un usage en temps réel, sous la charge attendue ;
- respecter les normes et les principes d'architecture du consortium.

Le scénario de référence du document d'exigences (p. 1-2) en donne un exemple : trois
hôpitaux (Fred Brooks, Julia Crusher, Beverly Bashir), une urgence en cardiologie près
de l'hôpital Fred Brooks ; l'hôpital Fred Brooks doit être proposé et un événement
publié pour réserver un lit.

### 1.2 Critères de succès

Le document d'exigences (p. 1) fixe cinq critères de succès. Le tableau indique ce que
la PoC en mesure et où le rapport en traite.

| Critère du document d'exigences | Mesuré par la PoC ? | Section |
| --- | --- | --- |
| Plus de 90 % des cas d'urgence acheminés vers l'hôpital compétent le plus proche du réseau | Non : objectif métier, mesurable seulement en exploitation. La PoC vérifie la règle de sélection elle-même. | [4.1](#41-résultats-fonctionnels), [5.4](#54-objectifs-métier-et-méthode-de-mesure) |
| Temps moyen de traitement d'une urgence ramené de 18,25 à 12 minutes | Non : objectif métier. La PoC ne mesure que la part de l'API dans ce temps. | [5.4](#54-objectifs-métier-et-méthode-de-mesure) |
| Temps de réponse inférieur à 200 ms avec une charge allant jusqu'à 800 requêtes par seconde, par instance de service | Oui : campagnes k6 du 22 novembre 2026 | [4.2](#42-performance-de-lapi) |
| Mise en œuvre respectant les normes imposées | Oui : justification et matrice des principes | [3](#3-respect-des-normes-et-des-principes) |
| Mise en œuvre terminée dans le délai imparti | Non mesurable dans le dépôt : le document d'exigences ne donne pas d'échéance | <!-- TODO Matt : échéance de la PoC et date de livraison effective --> |

Les exigences de réalisation (p. 2-3) complètent ces critères : API REST en Java,
inscrite à terme dans une architecture microservice ; interface graphique avec l'un
des frameworks Angular, React ou VueJS ; protection des données du patient ; pyramide
de tests unitaires, d'intégration et E2E, avec des tests de stress ; CI/CD, workflow
Git et README documentés ; ce document de reporting.

### 1.3 Ce qui est construit et ce qui est simulé

La Déclaration des travaux d'architecture (p. 4) autorise la PoC à utiliser des
implémentations factices des systèmes auxiliaires, à simplifier leurs contrats et à
documenter ses lacunes avec un plan de remédiation. La PoC applique ces
simplifications ainsi :

| Élément | Construit | Simulé ou hors périmètre |
| --- | --- | --- |
| API de réservation | Service Spring Boot complet : recherche de l'hôpital le plus proche (PostGIS), réservation transactionnelle, idempotence des demandes, outbox, sécurité JWT | — |
| Interface graphique | Front SvelteKit : connexion, carte des hôpitaux, formulaire lieu et spécialité, affichage de l'hôpital retenu | La localisation se choisit parmi huit lieux de démonstration du Grand Londres, pas par saisie libre ([README du front](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/README.md#carte-des-hôpitaux-et-lieux-durgence)) |
| Référentiel des spécialités | Les 80 spécialités du référentiel NHS, chargées par une migration Flyway ([README](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#démarrage-avec-docker)) | — |
| Référentiel hospitalier | Tables `hospital` et `hospital_specialty`, compteur de lits mis à jour à chaque réservation | Données fictives : 12 hôpitaux de démonstration dans le Grand Londres ([`seed-demo.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/docker/seed/seed-demo.sql)) ; aucune alimentation par les systèmes hospitaliers réels |
| Événements | Publication de `BedReserved` dans Redis Streams, avec un groupe de consommateurs créé | Aucun consommateur métier (hôpital receveur, tableau de bord) n'est construit |
| Identité | Keycloak, OpenID Connect, rôles `LECTEUR` et `REGULATEUR` | Comptes de démonstration ; le fournisseur d'identité du consortium n'est pas intégré |
| Distance | Distance à vol d'oiseau sur l'ellipsoïde WGS 84 | Pas de temps de trajet réel ([ADR 0005](../adr/0005-postgis-recherche-proximite.md)) |
| Données patient | Aucune : la demande ne contient qu'un lieu et une spécialité | Pas de dossier patient, pas de tri selon les symptômes (hors projet selon le Document de définition de l'architecture, p. 16) |
| Déploiement | Docker Compose sur un poste, images publiées sur GHCR par la CI | Pas d'environnement de production, pas d'orchestrateur |

Les contextes et conteneurs sont décrits dans les diagrammes de
[contexte](../diagrammes/contexte.md) et de [conteneurs](../diagrammes/conteneurs.md).

## 2. Justification des technologies

Deux technologies sont encadrées par le consortium : le langage de l'API (Java) et le
framework du front (Angular, React ou VueJS). Les autres choix sont libres ; chacun a
été retenu pour un besoin précis de la PoC, face à une alternative écartée. Les
décisions structurantes sont détaillées dans les ADR du dépôt d'architecture.

### 2.1 API : Java imposé, Spring Boot 4.1 et Java 21

**La technologie Java a été imposée par le consortium.** Le document d'exigences
(p. 2) le dit explicitement pour l'API RESTful, et le principe C3 des Principes de
l'architecture demande de favoriser les langages exécutables sur JVM. L'API est donc
écrite en Java ; le choix ne portait que sur la version de Java et sur le cadre
applicatif ([ADR 0001](../adr/0001-backend-java-spring-boot.md)).

**Java 21** est la version LTS retenue
([`pom.xml`](https://github.com/TheFatAndFurious/MedHead/blob/master/pom.xml)) :

- support à long terme, avec une fin de support plus lointaine que Java 17 ;
- `record` pour les commandes, réponses et événements immuables (`ReserveBedCommand`,
  `HospitalReservationResponse`, `BedReservedEvent`) ;
- API de fils modernes, utilisées pour le fil d'écoute de l'outbox.

**Spring Boot 4.1** (4.1.1 au commit de référence) couvre chaque besoin de l'API par
une brique standard, sans code d'infrastructure à écrire :

| Besoin | Brique Spring Boot |
| --- | --- |
| API REST versionnée et validation des entrées | Spring Web MVC, Bean Validation |
| Persistance et données géographiques | Spring Data JPA, Hibernate Spatial, JDBC Client |
| Migrations de schéma | Flyway |
| Sécurité OAuth 2.0 / OpenID Connect | Spring Security, serveur de ressources JWT |
| Publication d'événements | Spring Data Redis (Redis Streams) |
| Santé et métriques | Actuator, Micrometer |
| Tests | Spring Boot Test, MockMvc |

Alternatives écartées : Quarkus et Micronaut (démarrage plus rapide, mais écosystème
plus réduit, et la PoC n'a pas de contrainte de démarrage à froid), Jakarta EE sur un
serveur d'application (plus lourd à conteneuriser et à tester pour un service unique).

### 2.2 Front : SvelteKit par dérogation, critère de choix

Le document d'exigences (p. 2) impose l'un des frameworks JavaScript / TypeScript
courants du marché : **Angular, React ou VueJS**. Le front de la PoC est écrit avec
**SvelteKit** (Svelte 5, TypeScript, adaptateur Node), **par dérogation validée par le
mentor** le <!-- TODO Matt : date de validation de la dérogation SvelteKit par le mentor -->.
La dérogation et ses alternatives sont tracées dans
l'[ADR 0002](../adr/0002-front-sveltekit-derogation.md).

**Critère de choix : la sécurité des jetons d'authentification.** L'API est un serveur
de ressources OAuth 2.0 qui attend un jeton d'accès JWT. Si le navigateur détient ce
jeton, une faille XSS suffit à le voler. SvelteKit fournit **nativement des routes et
des actions exécutées côté serveur** (`hooks.server.ts`, `+page.server.ts`) : le front
mène lui-même la connexion OpenID Connect en client confidentiel et appelle l'API
depuis son serveur. **Les jetons OAuth ne sont jamais exposés au navigateur**, qui ne
reçoit qu'un cookie de session opaque, sans service « Backend for Frontend » séparé à
écrire et à déployer
([`hooks.server.ts`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/src/hooks.server.ts),
[`+page.server.ts`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/src/routes/%2Bpage.server.ts)).

Deux critères secondaires ont pesé :

- **la légèreté** : Svelte compile les composants en JavaScript, sans moteur de DOM
  virtuel livré au navigateur ; le front n'a que trois dépendances d'exécution
  (`openid-client`, `redis`, `maplibre-gl`,
  [`package.json`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/package.json)) ;
- **la simplicité** : un seul projet porte les pages, les formulaires et la partie
  serveur, et les formulaires HTML fonctionnent sans JavaScript côté client.

Comparaison avec les frameworks de la liste imposée :

| Framework | Jetons hors du navigateur | Conclusion |
| --- | --- | --- |
| Angular | Application monopage : il faut un BFF séparé | Écarté : un service de plus |
| React | Idem ; Next.js offrirait des routes serveur, avec un cadre plus lourd | Écarté pour une page unique |
| VueJS | Avec Nuxt, routes serveur comparables à SvelteKit | **Alternative de repli** si la dérogation était retirée |

Le choix du front n'engage pas le reste du système : le contrat entre le front et
l'API est une **API REST versionnée (`/api/v1`), indépendante du framework**.
Remplacer le front par une application Vue (Nuxt) ou Angular ne demanderait aucune
modification de l'API.

### 2.3 Autres technologies

| Technologie | Rôle dans la PoC | Pourquoi | Alternative écartée |
| --- | --- | --- | --- |
| **PostgreSQL 17 + PostGIS** | Base de réservation, recherche de l'hôpital le plus proche | Le type `geography` calcule les distances en mètres, et un index GiST résout le tri par distance (`<->`). La recherche et le décrément du lit restent dans la même transaction ([ADR 0005](../adr/0005-postgis-recherche-proximite.md)). | Calcul de distance en Java sur toute la table ; Elasticsearch ou Redis GEO, qui ajouteraient une seconde source de vérité à synchroniser avec le compteur de lits |
| **Redis 8.2 Streams** | Bus d'événements de la PoC (`BedReserved`) et stockage des sessions du front | Journal persistant, groupes de consommateurs et acquittements dans un seul conteneur, déjà nécessaire pour les sessions ([ADR 0003](../adr/0003-outbox-transactionnelle-redis-streams.md)) | Kafka (adapté à la production, mais un cluster à exploiter) ; Redis Pub/Sub (aucun stockage) ; RabbitMQ (pas de relecture par défaut) |
| **Keycloak 26** | Fournisseur d'identité OpenID Connect, rôles `LECTEUR` et `REGULATEUR` | Standard OIDC, open source, hébergeable sur place : l'API ne dépend que de jetons standard ([ADR 0004](../adr/0004-authentification-keycloak-sessions-serveur.md)) | Authentification maison dans l'API ; fournisseur SaaS (données d'identité hors de l'infrastructure) |
| **Flyway** | Migrations versionnées du schéma (`V1` à `V6`) et chargement du référentiel des spécialités | Schéma reproductible, appliqué au démarrage de l'API, intégré à Spring Boot ([migrations](https://github.com/TheFatAndFurious/MedHead/tree/master/src/main/resources/db/migration)) | Génération du schéma par Hibernate (non maîtrisée en production) ; Liquibase (équivalent, plus verbeux pour du SQL simple) |
| **k6** | Tests de charge et de stress de l'API | Scénarios versionnés en JavaScript, débit d'arrivée imposé et seuils automatisables ([stratégie](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/performance/strategie-tests-charge.md)) | JMeter (scénarios XML peu lisibles en revue) ; Gatling (scénarios en Scala ou Java, plus lourds à maintenir pour un seul point d'accès) |
| **ArchUnit** | Test d'architecture : sens des dépendances `api` → `application` → `persistence` → `domain` | La règle fait échouer le build si une couche dépend d'une couche supérieure ([`ArchitectureTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/ArchitectureTest.java)) | Découpage en modules Maven (plus lourd pour un seul service) ; revue manuelle seule |
| **Cucumber** | Test d'acceptation du scénario du document d'exigences, en français | Le scénario du PDF est exécuté tel quel, en langage métier, comme le demandent les principes B4 et C3 (BDD) ([`intervention-urgence.feature`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/resources/features/intervention-urgence.feature)) | Test JUnit nommé d'après le scénario : moins lisible par un interlocuteur métier |
| **Playwright** | Tests E2E dans Chromium sur la pile complète | Pilote un vrai navigateur à travers la connexion Keycloak, avec traces et rapport HTML en CI ([`frontend/e2e`](https://github.com/TheFatAndFurious/MedHead/tree/master/frontend/e2e)) | Cypress (gestion plus contrainte des redirections entre origines, nécessaires pour Keycloak) ; Selenium (plus verbeux, sans attente automatique) |
| **Docker et Docker Compose** | Images de l'API et du front ; pile complète en une commande | Environnement reproductible, utilisateurs non privilégiés, limites de ressources pour les tests de charge ([`docker-compose.yaml`](https://github.com/TheFatAndFurious/MedHead/blob/master/docker-compose.yaml)) | Installation manuelle des services ; Kubernetes, disproportionné pour une PoC sur un poste |
| **GitHub Actions** | CI/CD : tests, rapports, images GHCR, releases | Intégré au dépôt, déclenché par les pull requests, les push sur `master` et les tags ([`ci.yml`](https://github.com/TheFatAndFurious/MedHead/blob/master/.github/workflows/ci.yml)) | Jenkins (serveur à héberger) ; GitLab CI (dépôt hébergé sur GitHub) |

## 3. Respect des normes et des principes

<!-- À rédiger (#26) -->

### 3.1 RGPD

<!-- À rédiger (#26) -->

### 3.2 Architecture microservice

<!-- À rédiger (#26) -->

### 3.3 Intégration avec une architecture basée sur les événements

<!-- À rédiger (#26) -->

### 3.4 Matrice des principes d'architecture

<!-- À rédiger (#26) -->

## 4. Résultats de la PoC

<!-- À rédiger (#26) -->

### 4.1 Résultats fonctionnels

<!-- À rédiger (#26) -->

### 4.2 Performance de l'API

<!-- À rédiger (#26) -->

## 5. Enseignements

<!-- À rédiger (#26) -->

### 5.1 Réserver puis publier, ou publier pour réserver

<!-- À rédiger (#26) -->

### 5.2 L'outbox sous charge

<!-- À rédiger (#26) -->

### 5.3 Compromis assumés, écarts avec la production et plan de remédiation

<!-- À rédiger (#26) -->

### 5.4 Objectifs métier et méthode de mesure

<!-- À rédiger (#26) -->

### 5.5 Retour sur la démarche

<!-- À rédiger (#26) -->

## 6. Annexes

<!-- À rédiger (#26) -->
