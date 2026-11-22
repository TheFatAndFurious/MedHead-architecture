# MedHead-architecture

Dépôt d'architecture de la preuve de concept (PoC) **MedHead** : un système d'intervention
d'urgence qui réserve un lit dans l'hôpital disponible le plus proche possédant la
spécialité demandée.

Ce dépôt rassemble les documents d'architecture fournis par le consortium, les diagrammes
de la PoC telle qu'elle est construite, les décisions d'architecture (ADR) et le document
de reporting. Le code se trouve dans un dépôt séparé.

## Liens utiles

| Ressource | Lien |
| --- | --- |
| Dépôt de code | [TheFatAndFurious/MedHead](https://github.com/TheFatAndFurious/MedHead) |
| Intégration continue | [GitHub Actions, workflow `ci.yml`](https://github.com/TheFatAndFurious/MedHead/actions/workflows/ci.yml) |
| Releases | [Releases du dépôt de code](https://github.com/TheFatAndFurious/MedHead/releases) |
| Images Docker | [Paquets GHCR du dépôt de code](https://github.com/TheFatAndFurious?tab=packages&repo_name=MedHead) |
| Plan d'action | [Issue TheFatAndFurious/MedHead#17](https://github.com/TheFatAndFurious/MedHead/issues/17) |

## Contenu

```
README.md                 ce fichier
docs/fournis/             documents fournis par le consortium (PDF)
diagrammes/               diagrammes Mermaid, établis à partir du code
adr/                      décisions d'architecture
reporting/rapport-poc.md  document de reporting de la PoC
```

### Documents fournis

Le dossier [`docs/fournis/`](docs/fournis/) accueille les cinq documents PDF remis au
lancement de la mission, parmi lesquels « Principes de l'architecture », « Déclaration des
travaux », « Document de définition de l'architecture » et « Exigences ». Ils sont
conservés tels quels.

### Diagrammes

Les diagrammes sont écrits en [Mermaid](https://mermaid.js.org/) : GitHub les affiche
directement. Ils décrivent le code de la branche `master` du dépôt de code ; chaque fichier
indique le commit de référence et les fichiers sources dont il est tiré.

| Diagramme | Objet |
| --- | --- |
| [Contexte (C4 niveau 1)](diagrammes/contexte.md) | Les utilisateurs de MedHead et les systèmes avec lesquels il échange. |
| [Conteneurs (C4 niveau 2)](diagrammes/conteneurs.md) | Le front SvelteKit, l'API Spring Boot, PostGIS, Redis, Keycloak et les futurs consommateurs d'événements. |
| [Séquence de réservation](diagrammes/sequence-reservation.md) | Du formulaire à l'événement `BedReserved` dans Redis Streams. |
| [Déploiement](diagrammes/deploiement.md) | Docker Compose en local, la CI GitHub Actions et la publication des images. |

### Décisions d'architecture (ADR)

Chaque ADR suit le même format court : Statut, Contexte, Décision, Alternatives écartées,
Conséquences.

| N° | Décision | Statut |
| --- | --- | --- |
| [0001](adr/0001-backend-java-spring-boot.md) | API en Java 21 avec Spring Boot 4.1 | Acceptée |
| [0002](adr/0002-front-sveltekit-derogation.md) | Front SvelteKit, par dérogation à la liste imposée | Acceptée |
| [0003](adr/0003-outbox-transactionnelle-redis-streams.md) | Outbox transactionnelle publiée dans Redis Streams | Acceptée |
| [0004](adr/0004-authentification-keycloak-sessions-serveur.md) | Keycloak OIDC et sessions côté serveur | Acceptée |
| [0005](adr/0005-postgis-recherche-proximite.md) | PostGIS pour la recherche de proximité | Acceptée |
| [0006](adr/0006-reserver-puis-publier.md) | Réserver le lit, puis publier l'événement | Acceptée |

### Reporting

Le [rapport de la PoC](reporting/rapport-poc.md) présente la justification des
technologies, le respect des normes et des principes, les résultats et les enseignements
de la preuve de concept.

## Workflow

Ce dépôt suit le même workflow Git que le dépôt de code : une branche par issue, une pull
request vers `master`, des commits au format
[Conventional Commits](https://www.conventionalcommits.org/fr/) et un merge en merge
commit. Les issues de suivi restent dans le dépôt de code ; les pull requests de ce dépôt
y renvoient avec `Refs TheFatAndFurious/MedHead#<numéro>`.
