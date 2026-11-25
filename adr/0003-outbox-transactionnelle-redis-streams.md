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
   à partir du début du flux. Le publieur ne recrée pas les groupes à chaque événement :
   ses `XADD` utilisent `NOMKSTREAM`, et seule la disparition du flux provoque la
   recréation du flux et de ses groupes (voir [Évolutions](#évolutions)).

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
  marque l'événement publié échoue, l'événement est renvoyé. Depuis la publication par
  lots (voir [Évolutions](#évolutions)), c'est tout le lot qui est renvoyé. Les **consommateurs doivent
  être idempotents** et dédoublonner par `eventId`.
- **Changer de courtier sans toucher au domaine** : seule la classe
  `RedisStreamPublisher` dépend de Redis. Passer à Kafka ou à RabbitMQ revient à écrire un
  autre publieur ; le domaine et le service de réservation n'écrivent qu'une ligne
  d'outbox et ne changent pas.
- **Plusieurs instances** : `SKIP LOCKED` permet à plusieurs publieurs de se partager
  l'outbox sans publier deux fois le même événement en temps normal.
- **Points ouverts**, suivis dans le dépôt de code :
  - débit : une transaction par événement
    ([#6](https://github.com/TheFatAndFurious/MedHead/issues/6)) ; publication par lots
    en cours ([#40](https://github.com/TheFatAndFurious/MedHead/issues/40)), effet à
    mesurer ;
  - écoute `LISTEN` sans délai maximal
    ([#7](https://github.com/TheFatAndFurious/MedHead/issues/7)) ;
  - un événement impossible à publier bloque les suivants
    ([#8](https://github.com/TheFatAndFurious/MedHead/issues/8)) ;
  - un flux dont un groupe n'est plus lu (consommateur arrêté) peut remplir Redis
    ([#2](https://github.com/TheFatAndFurious/MedHead/issues/2)) ;
  - les événements publiés restent dans la table : une purge sera nécessaire ;
  - le `NOTIFY` fait passer un par un les commits des réservations, ce qui rend la
    latence de l'API sensible à la vitesse du disque (voir [Évolutions](#évolutions)).

## Évolutions

### 25 novembre 2026 : groupes créés une fois, consommateur de démonstration

Les campagnes de charge du 22 novembre 2026 ont montré que le publieur plafonne entre 140
et 240 événements par seconde, et que le flux atteignait 172 802 entrées malgré
`MAXLEN ~ 10000` ([rapport, 5.2](../reporting/rapport-poc.md#52-loutbox-sous-charge)).
Deux corrections, sans changer la décision :

- **Groupes créés une seule fois.** Le publieur envoyait un `XGROUP CREATE` par groupe et
  par événement, refusé en `BUSYGROUP` sauf la première fois : un aller-retour Redis
  inutile à chaque publication. Les groupes sont désormais créés avant le premier envoi.
  Les `XADD` portent `NOMKSTREAM` : si le flux a été supprimé, Redis refuse l'ajout au
  lieu de recréer un flux sans groupe, ce qui rendrait la rétention `ACKED` inopérante.
  Le publieur recrée alors le flux et ses groupes, puis renvoie l'événement. Limite :
  la suppression explicite d'un groupe (`XGROUP DESTROY`) sur un flux existant n'est
  détectée qu'au redémarrage de l'API.
- **Rétention `ACKED` conservée, avec un consommateur de démonstration.** Plutôt que de
  supprimer des entrées non acquittées (politique `KEEPREF`), la PoC garde la protection
  `ACKED` et ajoute le service `event-consumer`
  ([`consumer.py`](https://github.com/TheFatAndFurious/MedHead/blob/master/docker/event-consumer/consumer.py)) : un script Python
  qui lit le groupe `reservation-consumers` (`XREADGROUP`), journalise chaque événement
  puis l'acquitte (`XACK`). Il représente un consommateur métier minimal, démarre avec la
  PoC et participe aux campagnes de charge, ce qui permet de vérifier que le flux reste
  borné quand les événements sont acquittés. Le risque d'un consommateur arrêté reste
  ouvert ([#2](https://github.com/TheFatAndFurious/MedHead/issues/2)).

**Mesure.** Les campagnes du 25 novembre 2026, à 400 réservations/s
([résultats](https://github.com/TheFatAndFurious/MedHead/blob/master/docs/performance/resultats-2026-11-25.md)),
montrent que le flux reste borné à 10 000 entrées et que le publieur passe de 206 à
près de 300 événements/s au pic (277 et 308 dans les deux campagnes). Elles ont aussi révélé une conséquence de la décision : le
`NOTIFY` émis dans la transaction de réservation fait prendre à PostgreSQL, au `COMMIT`,
un verrou exclusif global gardé jusqu'à la fin de l'écriture du journal sur disque. Ces
commits passent donc un par un. Quand le disque ralentit, ici à cause des
synchronisations AOF de Redis (`appendfsync always`) sur le même disque virtuel, la file
d'attente se reporte sur l'API : p99 du pic à 854 ms au lieu de 15 ms le 22 novembre.
Sans ces synchronisations, le publieur suit les 400 événements/s et l'API retrouve un p99
de 16 ms ; les réglages utilisés pour le montrer acceptent une perte de données en cas
de panne et ne sont pas adoptés. Ce verrou devient un point ouvert.

### 25 novembre 2026 : publication par lots

Pour réduire les écritures disque à l'origine de ce ralentissement
([#40](https://github.com/TheFatAndFurious/MedHead/issues/40)), le publieur traite
désormais un lot de 100 événements au plus dans **une seule transaction** :
une sélection `FOR UPDATE SKIP LOCKED`, un **pipeline** de `XADD` dans l'ordre de
l'outbox, une seule requête `UPDATE` pour marquer le lot publié, puis un commit. Un lot
coûte un aller-retour Redis et une écriture du journal PostgreSQL au lieu d'un par
événement ; avec `appendfsync always`, Redis ne synchronise en principe son fichier AOF
qu'une fois pour les commandes reçues ensemble. La décision ne change pas :

- **Doublons plus nombreux après une panne** : si Redis accepte une partie du lot puis
  qu'une erreur survient, ou si le commit échoue, aucun événement du lot n'est marqué
  publié et tout le lot est renvoyé avec les mêmes `eventId`. Le dédoublonnage par les
  consommateurs, déjà exigé, couvre ce cas. La taille du lot
  (`MEDHEAD_OUTBOX_BATCH_SIZE`) arbitre entre coût par événement et nombre de doublons
  possibles.
- **Verrous des lignes gardés pendant tout le lot** : un second publieur prend les lignes
  suivantes grâce à `SKIP LOCKED`, sans attendre.

Le même travail corrige le **retard d'environ une seconde** observé au pic. Quand des
événements restaient en attente après un lot incomplet, le publieur dormait pendant tout
le délai de nouvelle tentative (1 s), pour le cas où un autre publieur les verrouillerait.
Sous charge, c'était le cas le plus courant, car des réservations étaient validées
pendant le lot. Il attend désormais leur notification, déjà arrivée, et n'utilise ce
délai que comme borne. L'effet des deux changements reste à mesurer par une campagne en
configuration par défaut.
