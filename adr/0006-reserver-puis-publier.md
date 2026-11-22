# ADR 0006 : réserver le lit, puis publier l'événement

## Statut

Acceptée.

## Contexte

Le scénario de la PoC demande de **publier un événement pour réserver un lit**. Lu à la
lettre, l'événement serait une commande (« réserve un lit ») traitée plus tard par un
consommateur, qui décrémenterait le compteur de l'hôpital.

Or la réservation doit respecter un **invariant de capacité** : un hôpital ne peut pas
attribuer plus de lits qu'il n'en a de libres. Lors d'un afflux d'urgences, plusieurs
régulateurs peuvent viser au même instant le dernier lit d'un même hôpital. De plus, le
régulateur, au téléphone avec l'appelant, a besoin de connaître tout de suite l'hôpital
retenu.

## Décision

Le code **réserve d'abord, puis publie** l'événement `BedReserved` :

1. Dans une seule transaction PostgreSQL, l'API cherche l'hôpital le plus proche avec la
   spécialité et un lit libre, puis **décrémente son compteur sous condition**
   (`UPDATE … SET available_beds = available_beds - 1 WHERE id = … AND available_beds > 0`).
   Si le décrément ne touche aucune ligne, une autre demande a pris le dernier lit :
   l'API relance la recherche dans la même transaction.
2. Dans la même transaction, elle enregistre la réservation et écrit `BedReserved` dans
   l'outbox (voir [ADR 0003](0003-outbox-transactionnelle-redis-streams.md)).
3. Après le `COMMIT`, l'API répond au front avec l'hôpital retenu, et l'événement est
   publié dans Redis Streams.

L'événement est donc un **fait accompli**, au passé, qui **informe** les autres systèmes ;
ce n'est pas une commande.

## Alternatives écartées

- **Publier une commande `ReserveBedRequested`, traitée par un consommateur** :
  - la réponse au régulateur devient asynchrone : il faut attendre puis interroger le
    résultat, ou ouvrir un canal de notification, alors que l'appel est en cours ;
  - l'invariant de capacité doit être tenu par le consommateur. Avec plusieurs
    consommateurs en parallèle, il faut soit sérialiser les commandes par hôpital
    (partitionnement), soit reprendre le même verrouillage en base, sans rien gagner ;
  - un refus (plus de lit) doit être renvoyé au demandeur par un autre événement, puis
    relancé vers l'hôpital suivant : la logique de choix se disperse.
- **Réserver puis publier sans outbox** : risque de double écriture, traité dans l'ADR 0003.

## Conséquences

- **Pas de double attribution du dernier lit** : le décrément conditionnel et le verrou de
  ligne qu'il pose jusqu'au `COMMIT` sérialisent les demandes concurrentes sur un même
  hôpital. La contrainte `CHECK (available_beds >= 0)` sert de filet de sécurité. Ce
  comportement est vérifié par les tests d'intégration
  `reservesTheLastBedOnlyOnceUnderConcurrentRequests` et
  `continuesWithTheNextHospitalWhenConcurrentRequestsExhaustTheNearestOne` de
  [`BedReservationIntegrationTest`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/test/java/com/example/medhead/hospital/application/BedReservationIntegrationTest.java).
- **Réponse immédiate** au régulateur, avec l'hôpital, la distance et les lits restants.
- **L'intégration événementielle est conservée** : chaque réservation produit un événement
  que d'autres systèmes peuvent consommer, sans couplage avec l'API.
- **Écart d'interprétation à expliquer** : le rapport et la soutenance doivent présenter
  ce choix comme une lecture du scénario qui privilégie la cohérence. Si un hôpital doit un
  jour confirmer ou refuser une réservation, le flux pourra s'enrichir d'événements
  `BedReservationConfirmed` ou `BedReservationCancelled` (saga), sans remettre en cause la
  réservation transactionnelle.
