# ADR 0001 : API en Java 21 avec Spring Boot 4.1

## Statut

Acceptée.

## Contexte

Le consortium MedHead **impose la technologie Java** pour les services du système. Il
reste à choisir le cadre applicatif et la version de Java de l'API de réservation.

L'API doit :

- exposer une API REST sécurisée par des jetons OAuth 2.0 / OpenID Connect ;
- faire une recherche géographique dans PostgreSQL / PostGIS et une réservation
  transactionnelle ;
- publier des événements dans Redis ;
- versionner son schéma de base de données ;
- exposer des contrôles de santé et des métriques ;
- s'exécuter dans un conteneur et se tester automatiquement en CI.

## Décision

L'API est écrite en **Java 21**, version LTS, avec **Spring Boot 4.1** (4.1.1 au commit de
référence).

Les briques Spring Boot utilisées couvrent chaque besoin sans code d'infrastructure :

| Besoin | Brique |
| --- | --- |
| API REST et validation | Spring Web MVC, Bean Validation |
| Persistance et données géographiques | Spring Data JPA, Hibernate Spatial, JDBC Client |
| Migrations de schéma | Flyway |
| Sécurité | Spring Security, serveur de ressources OAuth 2.0 (JWT) |
| Événements | Spring Data Redis (Redis Streams) |
| Santé et métriques | Actuator, Micrometer |
| Tests | Spring Boot Test, MockMvc, ArchUnit |

Java 21 apporte notamment les `record`, utilisés pour les commandes, les réponses et les
événements (`ReserveBedCommand`, `BedReservedEvent`…), et les API de fils modernes
(`Thread.ofPlatform()` pour le fil d'écoute de l'outbox).

## Alternatives écartées

- **Quarkus ou Micronaut** : démarrage plus rapide et compilation native possible.
  L'écosystème et la documentation de Spring Boot sont plus étendus, et la PoC n'a pas de
  contrainte de démarrage à froid qui justifierait ce choix.
- **Jakarta EE sur un serveur d'application** : plus lourd à conteneuriser et à tester,
  sans bénéfice pour un service unique.
- **Java 17** : LTS plus ancienne, sans les apports de Java 21 et avec une fin de support
  plus proche.

## Conséquences

- Le choix est aligné sur la contrainte du consortium et sur le standard de fait des
  services Java.
- Spring Boot 4.1 tire Spring Framework 7 et Jackson 3 (paquet `tools.jackson`) : certains
  exemples en ligne, écrits pour Spring Boot 3, ne s'appliquent pas tels quels.
- L'API est empaquetée en JAR exécutable et en image Docker sur une JRE 21 Alpine, exécutée
  par un utilisateur non privilégié.
- Une montée vers une LTS plus récente de Java reste possible sans changer
  l'architecture.
