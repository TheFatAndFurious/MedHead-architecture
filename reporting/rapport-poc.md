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

Le document d'exigences cite deux exemples de normes et de principes à justifier : la
norme RGPD et le principe d'architecture microservice. Le Document de définition de
l'architecture (p. 3) ajoute la communication par événements comme objectif principal
de la plateforme. Ces trois points sont traités ci-dessous, puis la PoC est confrontée
à chacun des principes du document « Principes de l'architecture ».

### 3.1 RGPD

Le Document de définition de l'architecture (p. 4) impose que les données réelles des
patients restent conformes au RGPD, et que les prototypes non conformes à la production
utilisent des données anonymisées ou factices. Le principe C4 des Principes de
l'architecture demande la même isolation.

#### Données traitées

| Donnée | Où elle circule | Où elle est conservée |
| --- | --- | --- |
| **Position de l'urgence** (latitude, longitude) | Navigateur → front → API, dans la demande de réservation | Nulle part en clair. `reservation_request` n'en garde qu'une empreinte SHA-256, mêlée à la spécialité, pour reconnaître une demande rejouée ([`V6__create_reservation_request.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/resources/db/migration/V6__create_reservation_request.sql)). La réponse conservée contient la distance à l'hôpital, pas la position. |
| **Spécialité demandée** | Même chemin | `bed_reservation.specialty_code`, l'outbox et l'événement `BedReserved` ([`V4__create_bed_reservation_and_event_outbox.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/resources/db/migration/V4__create_bed_reservation_and_event_outbox.sql)) |
| **Identité de l'opérateur** (régulateur ou lecteur) | Keycloak, puis jeton d'accès et session serveur du front | Compte dans la base de Keycloak ; identifiant `sub` du jeton dans `reservation_request.requested_by`, pour tracer l'auteur de chaque réservation ; session Redis du front ([guide d'authentification](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/authentification.md#session-web)) |
| Hôpitaux, lits, spécialités | API, front, événements | Tables `hospital`, `specialty`, `hospital_specialty` : données d'établissements, pas de données personnelles |

Prise isolément, aucune de ces données n'identifie un patient. Mais le lieu et l'heure
d'une urgence, associés à une spécialité médicale, peuvent concerner une personne
identifiable et révéler une information de santé. La PoC les traite donc comme des
données personnelles sensibles, et la production devra les traiter comme des données de
santé.

#### Minimisation et finalité

- **Aucune donnée nominative du patient** : la demande ne contient qu'un lieu et un code
  de spécialité ([`HospitalReservationRequest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/api/HospitalReservationRequest.java)).
  Ni nom, ni numéro d'identification, ni symptôme ne sont demandés.
- **Position non stockée** : elle sert au calcul puis n'est conservée que sous forme
  d'empreinte, comme décrit ci-dessus.
- **Finalité unique** : orienter une urgence vers l'hôpital compétent le plus proche et y
  réserver un lit. L'identifiant de l'opérateur n'est conservé que pour l'idempotence et
  la traçabilité de la réservation.
- **Données fictives** : les 12 hôpitaux de démonstration sont fictifs, les comptes sont
  des comptes de démonstration, et les tests créent leurs propres données. La PoC
  n'utilise aucune donnée réelle de patient ni d'établissement.

#### Contrôle d'accès par rôles

Keycloak attribue l'un des deux rôles `LECTEUR` ou `REGULATEUR` ; l'API vérifie la
signature, l'émetteur et l'audience du jeton, puis applique les rôles
([`SecurityConfiguration`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/security/SecurityConfiguration.java),
[ADR 0004](../adr/0004-authentification-keycloak-sessions-serveur.md)) :

| Accès | Rôle exigé |
| --- | --- |
| Santé de l'API (`/actuator/health`, `liveness`, `readiness`) et `/actuator/info` | Public |
| Consulter les hôpitaux et les spécialités | `LECTEUR` ou `REGULATEUR` |
| Réserver un lit | `REGULATEUR` |
| Tout autre point d'accès | Refusé |

Ces règles sont vérifiées par les 10 tests de
[`SecurityConfigurationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/security/SecurityConfigurationTest.java)
et, de bout en bout, par le test E2E du lecteur dont la réservation envoyée directement
au serveur est refusée en `403`
([`access.spec.ts`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/e2e/access.spec.ts)).
Une clé d'idempotence est rattachée à l'opérateur : la même clé envoyée par un autre
compte ne renvoie jamais la réservation d'un tiers
([README, Idempotence des demandes](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#idempotence-des-demandes)).

#### Chiffrement en transit

- **PostgreSQL et Redis n'acceptent que des connexions TLS**, et l'API vérifie le
  certificat et le nom d'hôte (`sslmode=verify-full` pour PostgreSQL). Le test
  [`DatabaseTlsIntegrationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/DatabaseTlsIntegrationTest.java)
  vérifie que le pool utilise TLS, que le serveur refuse une connexion en clair et que
  `verify-full` refuse un certificat signé par une autre autorité. Les tests de
  [`OutboxPublicationIntegrationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/outbox/OutboxPublicationIntegrationTest.java)
  font de même pour Redis.
- **Compromis assumé** : entre le front SvelteKit et l'API, et vers Keycloak, les
  échanges passent en HTTP sur le réseau Docker interne. Les ports ne sont publiés que
  sur `127.0.0.1` ([diagramme de déploiement](../diagrammes/deploiement.md)). Ce
  compromis est acceptable pour une démonstration locale, pas pour un déploiement ; il
  est suivi dans [#16](https://github.com/TheFatAndFurious/MedHead/issues/16).

#### Jetons et journaux

- **Jetons conservés côté serveur** : le navigateur ne reçoit qu'un identifiant de
  session aléatoire de 256 bits dans un cookie `HttpOnly`, `SameSite=Lax`. Les jetons
  OAuth restent dans la session Redis du front, sous une clé hachée
  ([guide d'authentification](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/authentification.md#session-web)).
- **Journaux sans donnée sensible** : l'API ne journalise ni les demandes ni les
  réponses. Ses seuls messages applicatifs concernent l'outbox (échec de publication,
  reconnexion de l'écoute) et ne reprennent que le message de l'exception technique
  ([`OutboxPublisher`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/outbox/OutboxPublisher.java),
  [`OutboxNotificationListener`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/outbox/OutboxNotificationListener.java)).
  Le front n'écrit qu'un message générique quand le stockage des sessions est
  indisponible.

#### Durée de conservation

Durées déjà appliquées par la PoC :

| Donnée | Durée |
| --- | --- |
| Transaction de connexion OIDC | 10 minutes |
| Session web du front (jetons et identité) | 8 heures au plus ; inactivité SSO de 30 minutes, jeton d'accès de 5 minutes |
| Événements dans le flux Redis | Flux borné à environ 10 000 entrées acquittées (`MAXLEN ~ 10000`, rétention `ACKED`) |

Source : [guide d'authentification](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/authentification.md#session-web)
et [README, Rétention du stream](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#rétention-du-stream-et-maxlen).
En revanche, les tables `bed_reservation`, `reservation_request` et `event_outbox` ne
sont pas purgées dans la PoC.

**Proposition pour la production**, à valider avec le délégué à la protection des
données :

| Donnée | Durée proposée | Justification |
| --- | --- | --- |
| Événements publiés de l'outbox | 7 jours après publication, puis suppression | Une fois publié, l'événement vit dans le bus et le lac de données ; la copie locale ne sert qu'au diagnostic |
| `reservation_request` (lien opérateur ↔ réservation, réponse rejouable) | 30 jours, puis suppression | Couvre largement la fenêtre de rejeu d'une demande et l'analyse d'un incident récent |
| `bed_reservation` (hôpital, spécialité, date) | 12 mois, puis agrégation anonyme | Sert à mesurer les objectifs métier (voir [5.4](#54-objectifs-métier-et-méthode-de-mesure)) |
| Journaux techniques | 6 mois | Diagnostic et sécurité, sans donnée de demande |

#### Ce qui manque pour la production

- **Analyse d'impact relative à la protection des données (AIPD)** : obligatoire pour un
  traitement à grande échelle de données de santé, à mener avant la mise en service.
- **Registre des traitements** : déclarer le traitement « orientation des urgences et
  réservation de lits », ses finalités, ses destinataires (hôpitaux, consommateurs
  d'événements) et ses durées de conservation.
- **HTTPS de bout en bout**, y compris entre le front, l'API et Keycloak, et Keycloak en
  mode production ([#16](https://github.com/TheFatAndFurious/MedHead/issues/16)).
- **Chiffrement au repos** des volumes PostgreSQL et Redis, et de leurs sauvegardes.
- **Purge automatique** selon les durées retenues, y compris l'outbox
  ([#9](https://github.com/TheFatAndFurious/MedHead/issues/9) pour les événements
  orphelins).
- **Moindre privilège** sur les comptes techniques
  ([#1](https://github.com/TheFatAndFurious/MedHead/issues/1),
  [#3](https://github.com/TheFatAndFurious/MedHead/issues/3)).
- **Fonds de carte** chargés depuis un service tiers, qui reçoit l'adresse IP du poste
  et la zone consultée ([#14](https://github.com/TheFatAndFurious/MedHead/issues/14)).

### 3.2 Architecture microservice

Le document d'exigences (p. 2) demande une API qui puisse « s'inscrire à terme dans une
architecture microservice ». Le Document de définition de l'architecture (p. 13-14)
précise la cible : services découplés, une base par service (*database per service*),
architecture hexagonale. La PoC en applique les règles à son service unique :

| Propriété | Mise en œuvre dans la PoC | Preuve |
| --- | --- | --- |
| **Contexte borné** | Un seul service, responsable de la réservation de lits ; il correspond à la capacité « Bed Availability » de l'architecture cible. Les hôpitaux et spécialités y sont un modèle de lecture local. | [Diagramme de contexte](../diagrammes/contexte.md) |
| **API REST versionnée** | Toutes les routes métier sont sous `/api/v1` ; erreurs au format `application/problem+json` | [`HospitalReservationController`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/api/HospitalReservationController.java), [`ApiExceptionHandler`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/api/ApiExceptionHandler.java) |
| **Base dédiée** | La base de réservation n'appartient qu'à l'API ; Keycloak a sa propre base | [Diagramme de conteneurs](../diagrammes/conteneurs.md) |
| **Service sans état** | L'API ne garde aucune session (`STATELESS`) ; le front stocke ses sessions dans Redis. Les deux peuvent être répliqués derrière un répartiteur. Plusieurs instances peuvent publier l'outbox sans doublon grâce à `FOR UPDATE SKIP LOCKED`. | [`SecurityConfiguration`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/security/SecurityConfiguration.java), [ADR 0003](../adr/0003-outbox-transactionnelle-redis-streams.md) |
| **Conteneurisé** | Images de l'API (JRE 21 Alpine) et du front (Node 24), exécutées par un utilisateur non privilégié, publiées sur GHCR avec le commit dans le label OCI `revision` | [`Dockerfile`](https://github.com/TheFatAndFurious/MedHead/blob/master/Dockerfile), [`frontend/Dockerfile`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/Dockerfile), [diagramme de déploiement](../diagrammes/deploiement.md) |
| **Contrôles de santé** | `/actuator/health/liveness` et `/actuator/health/readiness` ; le `HEALTHCHECK` de l'image de l'API interroge `readiness`, et Compose attend ces contrôles avant de démarrer les services dépendants | [README, Réservation d'un lit et outbox](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#réservation-dun-lit-et-outbox), [`Dockerfile`](https://github.com/TheFatAndFurious/MedHead/blob/master/Dockerfile), [`docker-compose.yaml`](https://github.com/TheFatAndFurious/MedHead/blob/master/docker-compose.yaml) |
| **Couches vérifiées** | Dépendances à sens unique `api` → `application` → `persistence` → `domain`, vérifiées par ArchUnit à chaque build ; la couche `application` ignore les DTO HTTP | [`ArchitectureTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/ArchitectureTest.java) |
| **Contrat d'événement** | `BedReserved` : `eventId`, `type`, `aggregateId`, `occurredAt`, `payload` JSON | [README, Transaction](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#transaction), [ADR 0003](../adr/0003-outbox-transactionnelle-redis-streams.md) |
| **Identité externalisée** | L'API ne connaît que des jetons OIDC standard : changer de fournisseur d'identité ne demande qu'une configuration | [ADR 0004](../adr/0004-authentification-keycloak-sessions-serveur.md) |

Limites : la PoC ne contient qu'un service métier ; il n'y a ni passerelle d'API, ni
découverte de services, ni maillage de services. Le contrat REST est documenté dans le
README, mais aucune spécification OpenAPI n'est publiée, alors que le principe C3 la
recommande. L'architecture en couches isole déjà Redis derrière un seul publieur, mais
n'est pas encore une architecture hexagonale complète (ports explicites pour la
persistance). Ces points sont repris dans le plan de remédiation
([5.3](#53-compromis-assumés-écarts-avec-la-production-et-plan-de-remédiation)).

### 3.3 Intégration avec une architecture basée sur les événements

L'architecture cible prévoit que tous les services publient leurs événements métier
sur un bus d'événements commun, agrégé dans un lac de données (Document de définition
de l'architecture, p. 13). La PoC publie chaque réservation sous la forme d'un événement
`BedReserved`, sans coupler l'API à ses futurs consommateurs. Le déroulé complet est
représenté dans le [diagramme de séquence](../diagrammes/sequence-reservation.md).

1. **Outbox transactionnelle.** L'événement est écrit dans la table `event_outbox`
   **dans la même transaction** que le décrément du lit et la réservation : les deux
   existent, ou aucun. Il n'y a donc ni réservation sans événement, ni événement d'une
   réservation annulée ([ADR 0003](../adr/0003-outbox-transactionnelle-redis-streams.md)).
2. **Réveil par `LISTEN/NOTIFY`.** Un déclencheur PostgreSQL émet `pg_notify` au `COMMIT`.
   Le publieur, dans l'API, écoute ce canal sur une connexion dédiée et publie
   aussitôt, sans interroger la table à intervalle régulier
   ([`V5__notify_outbox_insertions.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/resources/db/migration/V5__notify_outbox_insertions.sql),
   [`OutboxNotificationListener`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/outbox/OutboxNotificationListener.java)).
3. **Redis Streams et groupes de consommateurs.** L'événement est ajouté au flux
   `medhead:reservations` par `XADD`. Le groupe `reservation-consumers` est créé dès la
   première publication : chaque futur système (hôpital receveur, tableau de bord,
   notifications) lira le flux avec son propre groupe, à son rythme, et acquittera ses
   lectures (`XREADGROUP`, `XACK`).
4. **Livraison au moins une fois.** Si Redis accepte l'événement mais que le `COMMIT` qui
   le marque publié échoue, l'événement est renvoyé **avec le même `eventId`**. Ce cas
   est testé dans
   [`OutboxPublicationIntegrationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/outbox/OutboxPublicationIntegrationTest.java)
   (15 tests : publication TLS, reprise, rollback, publications concurrentes, doublon
   après échec du commit, rétention).
5. **Consommateurs idempotents.** Chaque consommateur devra dédoublonner par `eventId`.
   Cette règle fait partie du contrat publié
   ([README, Réservation d'un lit et outbox](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#réservation-dun-lit-et-outbox)).
6. **Changement de courtier.** Seule la classe
   [`RedisStreamPublisher`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/outbox/RedisStreamPublisher.java)
   dépend de Redis. Passer à **Kafka** (journal partitionné et rejouable, adapté au lac de
   données de la cible) ou à **RabbitMQ** revient à écrire un autre publieur ; le domaine
   et le service de réservation, qui n'écrivent qu'une ligne d'outbox, ne changent pas.

L'événement est un fait accompli (« un lit a été réservé ») et non une commande : ce
choix, qui s'écarte de la lettre du scénario, est expliqué en
[5.1](#51-réserver-puis-publier-ou-publier-pour-réserver). Sous forte charge, le débit du
publieur est la limite principale de la PoC (voir [5.2](#52-loutbox-sous-charge)).

### 3.4 Matrice des principes d'architecture

La matrice reprend les quatorze principes du document « Principes de l'architecture ».
Le principe C4 (IV) demande que la PoC vise spécifiquement B1, B2, B3 et B4.

| Principe | Comment la PoC le respecte | Preuve | Écart restant |
| --- | --- | --- | --- |
| **A1** Primauté des principes | La PoC est confrontée à chaque principe dans cette matrice ; ses écarts sont justifiés et tracés. | Cette section, [ADR 0002](../adr/0002-front-sveltekit-derogation.md) (dérogation front) | — |
| **A2** Maximiser les avantages pour l'entreprise | La PoC fournit des modules réutilisables par d'autres services : pipeline CI/CD, motif outbox, authentification OIDC à jetons côté serveur, contrat REST indépendant du front. | [`ci.yml`](https://github.com/TheFatAndFurious/MedHead/blob/master/.github/workflows/ci.yml), ADR [0003](../adr/0003-outbox-transactionnelle-redis-streams.md) et [0004](../adr/0004-authentification-keycloak-sessions-serveur.md) | — |
| **A3** Conformité aux lois et aux règlements | Minimisation, contrôle d'accès, TLS vers les bases, données fictives ; référentiel des spécialités du NHS. | [3.1](#31-rgpd), [`V3__seed_nhs_specialties.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/resources/db/migration/V3__seed_nhs_specialties.sql) | AIPD, registre, chiffrement au repos, purge (voir 3.1) |
| **A4** Serment d'Hippocrate | « D'abord ne pas nuire » : jamais deux patients sur le même dernier lit, jamais de second lit consommé par une demande rejouée, refus explicite quand aucun lit n'est disponible plutôt qu'une orientation hasardeuse. | Tests de concurrence et d'idempotence de [`BedReservationIntegrationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/hospital/application/BedReservationIntegrationTest.java) ; aucune surréservation sous charge ([4.2](#42-performance-de-lapi)) | Distance à vol d'oiseau ; message « aucun lit » affiché pour tout `404` ([#10](https://github.com/TheFatAndFurious/MedHead/issues/10)) |
| **B1** Continuité des activités | Services sans état et réplicables ; publication de l'outbox reprise automatiquement après panne de Redis, sans bloquer les réservations ; idempotence pour rejouer une demande sans risque ; santé et métriques Actuator ; tests de charge et de stress. | [README, Réservation d'un lit et outbox](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#réservation-dun-lit-et-outbox), [résultats de charge](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/performance/resultats-2026-11-22.md) | Une seule instance de chaque conteneur ; pas de plan de reprise d'activité ni de SLI suivis en continu ; outbox en retard sous forte charge ([5.2](#52-loutbox-sous-charge)) ; [#2](https://github.com/TheFatAndFurious/MedHead/issues/2), [#7](https://github.com/TheFatAndFurious/MedHead/issues/7), [#8](https://github.com/TheFatAndFurious/MedHead/issues/8) |
| **B2** Séparation fine des préoccupations | Un contexte borné par service ; couches vérifiées par ArchUnit ; front, API, identité et bus séparés ; dépendances documentées par les diagrammes C4. | [`ArchitectureTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/ArchitectureTest.java), [diagramme de conteneurs](../diagrammes/conteneurs.md) | Redis partagé par l'API et les sessions du front ([#3](https://github.com/TheFatAndFurious/MedHead/issues/3)) |
| **B3** Intégration et livraison continues | Pipeline déclenché par chaque pull request, push sur `master` et tag `v*` ; artefacts nommés par SHA ; images GHCR étiquetées par commit ; release automatique sur tag ; petites pull requests liées à une issue. | [`ci.yml`](https://github.com/TheFatAndFurious/MedHead/blob/master/.github/workflows/ci.yml), [README, Intégration continue](https://github.com/TheFatAndFurious/MedHead/blob/master/README.md#intégration-continue), [run de référence](https://github.com/TheFatAndFurious/MedHead/actions/runs/45066204052) | Aucune release taguée à la date du rapport ; actions référencées par tag ([#12](https://github.com/TheFatAndFurious/MedHead/issues/12)) |
| **B4** Tests automatisés précoces et complets | Pyramide complète : 28 tests unitaires et de composant, 35 tests d'intégration, 3 scénarios d'acceptation BDD, 22 tests unitaires du front, 5 tests E2E, plus les campagnes de charge k6 ; rapports publiés par la CI. | [4.1](#41-résultats-fonctionnels) | Couverture de code non mesurée ; pas de tests de contrat consommateur (aucun consommateur construit) |
| **B5** Sécurité « shift-left » | Sécurité intégrée dès les premiers lots : OIDC, rôles, TLS `verify-full`, jetons hors du navigateur, contrôle d'origine, idempotence par utilisateur ; tests de sécurité automatisés ; risques de sécurité tracés en issues. | [`SecurityConfigurationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/security/SecurityConfigurationTest.java), [`DatabaseTlsIntegrationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/DatabaseTlsIntegrationTest.java), [ADR 0004](../adr/0004-authentification-keycloak-sessions-serveur.md) | Pas d'analyse automatique du code ni des dépendances en CI ; [#1](https://github.com/TheFatAndFurious/MedHead/issues/1), [#4](https://github.com/TheFatAndFurious/MedHead/issues/4), [#5](https://github.com/TheFatAndFurious/MedHead/issues/5), [#16](https://github.com/TheFatAndFurious/MedHead/issues/16) |
| **B6** Extension par les événements | Chaque réservation publie `BedReserved` ; de nouvelles fonctions s'y abonnent sans modifier l'API. | [3.3](#33-intégration-avec-une-architecture-basée-sur-les-événements), [ADR 0003](../adr/0003-outbox-transactionnelle-redis-streams.md) | Débit du publieur ([#6](https://github.com/TheFatAndFurious/MedHead/issues/6)) |
| **C1** ADM TOGAF 10 | Les livrables suivent les phases de la Déclaration des travaux : diagrammes (phases B à D), ADR (phase H), ce rapport (phases F et G). | [README du dépôt d'architecture](../README.md) | — |
| **C2** Référentiel d'architecture centralisé | Le dépôt `MedHead-architecture` rassemble documents fournis, diagrammes établis à partir du code, ADR et rapport, versionnés par pull request. | [README du dépôt d'architecture](../README.md) | Documents fournis à déposer (voir l'en-tête) |
| **C3** Normes reconnues | OpenID Connect ; déploiement conteneurisé, immuable et reproductible ; conception par domaine (contexte borné, langage métier dans les scénarios) ; développement guidé par le comportement (Cucumber) ; langage JVM ; documentation en Markdown. | [`intervention-urgence.feature`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/resources/features/intervention-urgence.feature), [diagramme de déploiement](../diagrammes/deploiement.md) | Pas de spécification OpenAPI ; pas de maillage ni de découverte de services ; Javadoc présente sur quelques classes seulement ; pas d'*event sourcing* (l'outbox publie des faits, elle n'est pas la source de l'état) |
| **C4** Culture de l'apprentissage | Hypothèse formulée ([1.1](#11-hypothèse-à-valider)) ; PoC isolée, sur un poste, avec des données factices ; plans de test autodocumentés en BDD ; rapports d'exécution en CI ; tests de performance sur l'algorithme clé (recherche et réservation) ; principes d'ingénierie, de livraison et de test non assouplis. | [4](#4-résultats-de-la-poc), [5](#5-enseignements) | — |

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
