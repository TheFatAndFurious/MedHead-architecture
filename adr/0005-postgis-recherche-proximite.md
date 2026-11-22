# ADR 0005 : PostGIS et index GiST pour la recherche de proximité

## Statut

Acceptée.

## Contexte

Pour chaque urgence, MedHead doit trouver **l'hôpital le plus proche** du lieu de
l'incident qui possède la spécialité demandée et au moins un lit disponible. La recherche
doit rester rapide quand le nombre d'hôpitaux augmente, et rester cohérente avec le
compteur de lits, modifié par les réservations concurrentes.

## Décision

Les données sont stockées dans **PostgreSQL avec l'extension PostGIS** :

- la position de chaque hôpital est une colonne `location` de type
  `geography(Point, 4326)` (coordonnées WGS 84, distances en mètres sur l'ellipsoïde) ;
- un **index GiST** (`idx_hospital_location`) indexe cette colonne ;
- la requête filtre sur la spécialité et `available_beds > 0`, puis **trie par l'opérateur
  de distance `<->`**, que PostGIS peut résoudre avec l'index GiST (recherche des plus
  proches voisins), et garde le premier résultat (`LIMIT 1`). L'identifiant de l'hôpital
  départage les égalités ;
- `ST_Distance` renvoie la distance en mètres, affichée à l'opérateur ;
- deux index complètent la recherche : un index partiel sur les hôpitaux ayant des lits
  libres, et un index sur `(specialty_id, hospital_id)`.

La recherche et le décrément du lit s'exécutent dans **la même base et la même
transaction** (voir [ADR 0006](0006-reserver-puis-publier.md)).

## Alternatives écartées

- **Calcul de distance en Java** (formule de haversine sur tous les hôpitaux) : parcourt
  toute la table à chaque demande, sans index.
- **Moteur de recherche externe** (Elasticsearch, Redis GEO) : bonnes fonctions
  géographiques, mais une seconde source de vérité à synchroniser avec le compteur de
  lits, avec un risque de proposer un hôpital déjà complet.
- **Service d'itinéraire externe** (temps de trajet réel) : plus juste métier, mais
  dépendance à un tiers, latence et coût à chaque demande. Écarté pour la PoC.
- **Type `geometry` ou extension `earthdistance`** : `geometry` calcule dans un plan, ce
  qui fausse les distances sans projection adaptée ; `earthdistance` est moins complet que
  PostGIS.

## Conséquences

- La recherche s'appuie sur un index, et la cohérence avec le compteur de lits est
  garantie par la transaction.
- La distance est **à vol d'oiseau**, pas un temps de trajet. C'est une simplification
  assumée de la PoC : un service d'itinéraire pourrait affiner le classement des
  hôpitaux les plus proches en production.
- La base doit disposer de PostGIS : la PoC construit sa propre image
  (`medhead-postgis:17`), dont le build dépend encore d'`apt-get`
  ([#15](https://github.com/TheFatAndFurious/MedHead/issues/15)).
- L'API utilise Hibernate Spatial pour lire les positions.
