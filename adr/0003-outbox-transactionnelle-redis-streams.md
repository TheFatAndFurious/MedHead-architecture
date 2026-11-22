# ADR 0003 : outbox transactionnelle publiée dans Redis Streams

## Statut

Acceptée.

## Contexte

Chaque réservation doit être portée à la connaissance d'autres systèmes (hôpital receveur,
tableau de bord, notifications), dans l'esprit d'une architecture basée sur les
événements. La réservation est écrite dans PostgreSQL ; l'événement doit partir vers un
courtier de messages.

Écrire dans deux systèmes sans transaction commune pose le problème de la **double
écriture** :

- publier **avant** le `COMMIT` peut annoncer une réservation qui sera finalement annulée ;
- publier **après** le `COMMIT` perd l'événement si l'application s'arrête entre les deux.

## Décision

L'API applique le motif de l'**outbox transactionnelle** :

1. Le service de réservation écrit l'événement `BedReserved` dans la table `event_outbox`,
   **dans la même transaction** que la réservation. Les deux existent, ou aucun.
2. Un déclencheur sur `event_outbox` appelle `pg_notify('medhead_outbox')`. PostgreSQL ne
   délivre la notification qu'au `COMMIT`.
3. Un publieur, dans l'API, écoute ce canal (`LISTEN`) sur une connexion dédiée, hors du
   pool. Au démarrage et à chaque notification, il publie les événements en attente.
4. Chaque événement est publié dans sa propre transaction : sélection avec
   `FOR UPDATE SKIP LOCKED`, `XADD` dans le flux Redis `medhead:reservations`, puis
   `published_at` renseigné et `COMMIT`.
5. Le flux est borné (`MAXLEN ~ 10000`, en ne supprimant que les entrées acquittées) et
   le groupe de consommateurs `reservation-consumers` est créé dès la première publication,
   à partir du début du flux.

Le contrat de l'événement est le suivant : `eventId`, `type`, `aggregateId` (identifiant de
la réservation), `occurredAt` et `payload` (JSON : réservation, hôpital, spécialité, date).

## Alternatives écartées

- **Publier directement dans Redis depuis la transaction** : Redis ne participe pas à la
  transaction PostgreSQL. Double écriture, voir ci-dessus.
- **Sondage périodique de l'outbox** : c'était la première version. Elle impose un
  compromis entre la latence (intervalle long) et la charge sur la base (intervalle
  court). `LISTEN/NOTIFY` réveille le publieur dès le `COMMIT`, sans requête quand il n'y
  a rien à publier.
- **Capture de changements (Debezium)** : publication fiable sans code applicatif, mais
  il faut Kafka Connect et un connecteur, trop lourd pour la PoC.
- **Kafka** : journal distribué, rejouable et partitionné, adapté à la production. Il
  ajoute un cluster à exploiter. Redis Streams offre les mêmes notions utiles à la PoC
  (journal persistant, groupes de consommateurs, acquittements) avec un seul conteneur,
  déjà présent pour les sessions.
- **RabbitMQ** : bon courtier de files, mais pas de relecture de l'historique par défaut.
- **Redis Pub/Sub** : aucun stockage ; un consommateur absent perd les messages.

## Conséquences

- **Livraison au moins une fois** : si Redis accepte l'entrée mais que le `COMMIT` qui
  marque l'événement publié échoue, l'événement est renvoyé. Les **consommateurs doivent
  être idempotents** et dédoublonner par `eventId`.
- **Changer de courtier sans toucher au domaine** : seule la classe
  `RedisStreamPublisher` dépend de Redis. Passer à Kafka ou à RabbitMQ revient à écrire un
  autre publieur ; le domaine et le service de réservation n'écrivent qu'une ligne
  d'outbox et ne changent pas.
- **Plusieurs instances** : `SKIP LOCKED` permet à plusieurs publieurs de se partager
  l'outbox sans publier deux fois le même événement en temps normal.
- **Points ouverts**, suivis dans le dépôt de code :
  - débit : une transaction et un `XGROUP CREATE` par événement
    ([#6](https://github.com/TheFatAndFurious/MedHead/issues/6)) ;
  - écoute `LISTEN` sans délai maximal
    ([#7](https://github.com/TheFatAndFurious/MedHead/issues/7)) ;
  - un événement impossible à publier bloque les suivants
    ([#8](https://github.com/TheFatAndFurious/MedHead/issues/8)) ;
  - un flux sans consommateur peut remplir Redis
    ([#2](https://github.com/TheFatAndFurious/MedHead/issues/2)) ;
  - les événements publiés restent dans la table : une purge sera nécessaire.
