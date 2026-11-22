# Diagramme de contexte (C4 niveau 1)

Ce diagramme situe MedHead parmi ses utilisateurs et les systèmes avec lesquels il échange.

> Établi à partir du commit
> [`b31cfc3`](https://github.com/TheFatAndFurious/MedHead/tree/b31cfc3) de `master`.
> Sources : [`SecurityConfiguration.java`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/security/SecurityConfiguration.java),
> [`medhead-realm.json`](https://github.com/TheFatAndFurious/MedHead/blob/master/docker/keycloak/medhead-realm.json),
> [`RedisStreamPublisher.java`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/outbox/RedisStreamPublisher.java),
> [`seed-demo.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/docker/seed/seed-demo.sql).

```mermaid
flowchart LR
    regulateur["👤 Régulateur<br/><i>Opérateur du centre d'appels d'urgence</i><br/>rôle REGULATEUR"]
    lecteur["👤 Lecteur<br/><i>Consulte les hôpitaux et les spécialités</i><br/>rôle LECTEUR"]

    medhead["<b>MedHead</b><br/>Système de réservation de lits d'urgence<br/><i>Trouve l'hôpital disponible le plus proche<br/>ayant la spécialité demandée et y réserve un lit</i>"]

    idp["<b>Fournisseur d'identité</b><br/>Keycloak, protocole OpenID Connect<br/><i>Authentifie les opérateurs et porte leurs rôles</i>"]
    hopitaux["<b>Référentiel hospitalier</b><br/><i>Hôpitaux, spécialités, lits disponibles</i><br/>Simulé dans la PoC par un jeu de données"]
    consommateurs["<b>Systèmes consommateurs</b><br/><i>Hôpital receveur, tableau de bord,<br/>notification des équipes</i><br/>Non construits dans la PoC"]

    regulateur -- "Demande un lit pour une urgence<br/>(lieu, spécialité)" --> medhead
    lecteur -- "Consulte la carte des hôpitaux" --> medhead
    regulateur -- "S'authentifie" --> idp
    lecteur -- "S'authentifie" --> idp
    medhead -- "Délègue l'authentification,<br/>vérifie les jetons" --> idp
    hopitaux -. "Alimente les hôpitaux,<br/>spécialités et lits" .-> medhead
    medhead -. "Publie l'événement BedReserved<br/>(Redis Streams)" .-> consommateurs

    classDef personne fill:#08427b,stroke:#052e56,color:#ffffff
    classDef systeme fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef externe fill:#999999,stroke:#6b6b6b,color:#ffffff
    classDef futur fill:#ffffff,stroke:#6b6b6b,color:#333333,stroke-dasharray:5 5
    class regulateur,lecteur personne
    class medhead systeme
    class idp,hopitaux externe
    class consommateurs futur
```

## Lecture

- **Régulateur** : seul rôle autorisé à réserver un lit
  (`POST /api/v1/hospitals/reservations`). Il peut aussi consulter les hôpitaux.
- **Lecteur** : consulte la liste des hôpitaux et des spécialités, sans réserver.
- **Fournisseur d'identité** : Keycloak est déployé avec la PoC (realm `medhead`), mais il
  joue le rôle d'un fournisseur d'identité externe. En production, il pourrait être remplacé
  par l'annuaire du consortium sans modifier l'API, qui ne vérifie que des jetons JWT.
- **Référentiel hospitalier** : la PoC ne se connecte à aucun système hospitalier réel. Les
  hôpitaux, leurs spécialités et leurs lits disponibles viennent d'un jeu de données de
  démonstration chargé au démarrage (`seed-demo.sql`). Les spécialités suivent la
  nomenclature du NHS, chargée par une migration Flyway.
- **Systèmes consommateurs** (pointillés) : chaque réservation publie un événement
  `BedReserved` dans un flux Redis, avec un groupe de consommateurs déjà créé
  (`reservation-consumers`). Aucun consommateur métier n'est construit dans la PoC.

Le niveau suivant est décrit dans le [diagramme de conteneurs](conteneurs.md).
