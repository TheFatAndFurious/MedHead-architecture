# Séquence d'une réservation de lit

Ce diagramme suit une demande de réservation, du formulaire du régulateur jusqu'à
l'événement `BedReserved` dans Redis Streams.

> Établi à partir du commit
> [`b31cfc3`](https://github.com/TheFatAndFurious/MedHead/tree/b31cfc3) de `master`, puis mis à jour
> pour la publication par lots ([#40](https://github.com/TheFatAndFurious/MedHead/issues/40)) et le
> réveil en mémoire du publieur ([#41](https://github.com/TheFatAndFurious/MedHead/issues/41)).
> Sources : [`+page.server.ts`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/src/routes/%2Bpage.server.ts),
> [`hooks.server.ts`](https://github.com/TheFatAndFurious/MedHead/blob/master/frontend/src/hooks.server.ts),
> [`HospitalReservationController.java`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/api/HospitalReservationController.java),
> [`ReserveNearestAvailableBedService.java`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/application/ReserveNearestAvailableBedService.java),
> [`HospitalRepository.java`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/persistence/HospitalRepository.java),
> [`ReservationRequestRepository.java`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/java/com/example/medhead/hospital/persistence/ReservationRequestRepository.java),
> [`V7__drop_outbox_notify_trigger.sql`](https://github.com/TheFatAndFurious/MedHead/blob/master/src/main/resources/db/migration/V7__drop_outbox_notify_trigger.sql)
> et le paquet [`outbox`](https://github.com/TheFatAndFurious/MedHead/tree/master/src/main/java/com/example/medhead/outbox).

## Partie synchrone : réserver

Le lit est réservé et l'événement est écrit dans l'outbox **dans la même transaction**.
La réponse est envoyée dès le `COMMIT`, sans attendre Redis.

```mermaid
sequenceDiagram
    autonumber
    actor R as Régulateur
    participant N as Navigateur
    participant F as Front SvelteKit
    participant S as Redis base 1 (sessions)
    participant A as API Spring Boot
    participant P as PostgreSQL + PostGIS

    R->>N: Choisit le lieu et la spécialité
    N->>F: POST / (lieu, spécialité, idempotencyKey) + cookie de session
    F->>F: Vérifie l'origine de la requête
    F->>S: Lit la session (jeton d'accès, rôles)
    S-->>F: Session de l'opérateur
    F->>F: Vérifie le rôle REGULATEUR, le lieu et la clé
    F->>A: POST /api/v1/hospitals/reservations<br/>Authorization: Bearer, Idempotency-Key<br/>(latitude, longitude, specialtyCode)
    A->>A: Valide le JWT (signature, iss, aud medhead-api, rôle REGULATEUR)

    A->>P: BEGIN
    A->>P: pg_advisory_xact_lock(sub + Idempotency-Key)
    A->>P: SELECT reservation_request (sub, clé)

    alt Demande déjà acceptée avec cette clé
        P-->>A: Réponse enregistrée
        A->>P: COMMIT
        A-->>F: 201 Created + Idempotent-Replayed: true<br/>(même réservation, aucun lit consommé)
        Note over A,F: Une clé réutilisée pour une autre demande renvoie 422.
    else Nouvelle demande
        loop Jusqu'à ce que le décrément touche une ligne
            A->>P: Hôpital le plus proche avec la spécialité et un lit libre<br/>(tri KNN sur l'index GiST, LIMIT 1)
            P-->>A: hospitalId, distance en mètres
            A->>P: UPDATE hospital SET available_beds = available_beds - 1<br/>WHERE id = hospitalId AND available_beds > 0
            P-->>A: 1 ligne (verrou gardé jusqu'au COMMIT)<br/>ou 0 ligne si une autre demande a pris le dernier lit
        end
        A->>P: INSERT bed_reservation
        A->>P: INSERT event_outbox (BedReserved, published_at NULL)
        A->>P: INSERT reservation_request (empreinte SHA-256, réponse JSON)
        A->>P: COMMIT
        Note right of A: Après le COMMIT seulement : réveil en mémoire<br/>du publieur de la même instance (aucun NOTIFY).
        A-->>F: 201 Created (hôpital, distance, lits restants)
    end

    F-->>N: Page de confirmation
    N-->>R: Hôpital, distance et lits restants affichés
```

### Cas d'échec

| Situation | Réponse de l'API | Comportement |
| --- | --- | --- |
| Aucun hôpital n'a de lit pour cette spécialité | `404 Not Found` | La transaction est annulée : aucun lit, aucune demande, aucun événement. La même clé peut être renvoyée plus tard. |
| Spécialité inconnue, coordonnées ou en-tête invalides | `400 Bad Request` | Idem. |
| Clé `Idempotency-Key` déjà utilisée pour une autre demande | `422 Unprocessable Content` | Le front invite à recharger la page pour obtenir une nouvelle clé. |
| Jeton absent, expiré ou sans le rôle `REGULATEUR` | `401` ou `403` | Le front invite à se reconnecter. |
| Pas de réponse de l'API (délai de 10 s dépassé, coupure) | aucune | Le front garde la **même** clé et propose de renvoyer la demande : si la réservation a été enregistrée, elle est renvoyée sans consommer de second lit. |

## Partie asynchrone : publier

Le publieur tourne dans l'API, sur un fil dédié. Il est réveillé en mémoire après chaque
réservation validée par la même instance ; sans réveil, il lit l'outbox au plus toutes les
secondes, pour les événements d'une autre instance arrêtée avant de les publier.

```mermaid
sequenceDiagram
    autonumber
    participant P as PostgreSQL + PostGIS
    participant L as Publieur de l'outbox<br/>(fil dans l'API)
    participant X as Redis base 0<br/>(flux medhead:reservations)
    participant C as Consommateurs<br/>(non construits)

    Note over L,P: Au démarrage : rattrapage des événements en attente.<br/>Ensuite : réveil en mémoire après chaque réservation validée, ou au plus tard après 1 s.

    loop Tant que le lot est complet : une transaction par lot de 100 événements au plus
        L->>P: BEGIN
        L->>P: SELECT event_outbox WHERE published_at IS NULL<br/>ORDER BY occurred_at, id LIMIT 100 FOR UPDATE SKIP LOCKED
        P-->>L: Événements non publiés les plus anciens
        Note over L,X: Groupes créés une seule fois (XGROUP CREATE … MKSTREAM),<br/>puis seulement si le flux a disparu.
        L->>X: Pipeline : un XADD par événement, NOMKSTREAM MAXLEN ~ 10000 ACKED<br/>(eventId, type, aggregateId, occurredAt, payload)
        X-->>L: Un identifiant par entrée
        L->>P: UPDATE event_outbox SET published_at = now() WHERE id IN (lot)
        L->>P: COMMIT
    end

    Note over L: Lot incomplet : attente du prochain réveil, 1 s au plus.<br/>Erreur Redis ou base : tout le lot est annulé, nouvel essai après 1 s.

    C->>X: XREADGROUP GROUP reservation-consumers
    X-->>C: BedReserved
    C->>C: Ignore un eventId déjà traité
    C->>X: XACK
```

### Garanties

- **Pas de double attribution** : le décrément conditionnel (`available_beds > 0`) et le
  verrou de ligne qu'il pose empêchent deux demandes simultanées de prendre le même
  dernier lit. La contrainte `CHECK (available_beds >= 0)` en est le filet de sécurité.
- **Pas d'événement sans réservation, ni de réservation sans événement** : les deux
  écritures sont dans la même transaction (voir [ADR 0003](../adr/0003-outbox-transactionnelle-redis-streams.md)
  et [ADR 0006](../adr/0006-reserver-puis-publier.md)).
- **Livraison au moins une fois** : si Redis accepte tout ou partie du lot mais que le
  `COMMIT` qui le marque publié échoue, tout le lot sera renvoyé. Les consommateurs
  dédoublonnent par `eventId`.
- **Plusieurs instances de l'API** : `FOR UPDATE SKIP LOCKED` permet à plusieurs
  publieurs de se partager l'outbox sans publier deux fois le même événement en temps
  normal.
- **Données personnelles** : la table `reservation_request` ne garde qu'une empreinte
  SHA-256 des coordonnées de l'urgence, et l'événement ne contient ni position ni identité.
