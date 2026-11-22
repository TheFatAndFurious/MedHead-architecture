# ADR 0004 : Keycloak OIDC, sessions côté serveur et API en serveur de ressources JWT

## Statut

Acceptée.

## Contexte

Réserver un lit engage un hôpital : seuls des opérateurs authentifiés et habilités doivent
pouvoir le faire. La PoC distingue deux profils :

- **Lecteur** : consulte les hôpitaux et les spécialités ;
- **Régulateur** : consulte et réserve.

L'authentification doit reposer sur un standard, pour qu'un fournisseur d'identité du
consortium puisse remplacer celui de la PoC. Les jetons ne doivent pas être exposés au
navigateur, et les services doivent rester sans état pour pouvoir être répliqués.

## Décision

- **Fournisseur d'identité : Keycloak**, protocole **OpenID Connect**. Le realm `medhead`
  définit les rôles `LECTEUR` et `REGULATEUR`, placés dans la revendication `roles` du
  jeton, et ajoute l'audience `medhead-api` au jeton d'accès.
- **Front SvelteKit : client confidentiel `medhead-web`** avec le flux *Authorization Code*,
  **PKCE (S256)**, `state` et `nonce`. L'échange du code, le rafraîchissement et la
  révocation des jetons se font côté serveur.
- **Sessions côté serveur dans Redis** (base 1). Le navigateur ne reçoit qu'un identifiant
  de session aléatoire de 256 bits, dans un cookie `httpOnly`, `SameSite=Lax`, préfixé
  `__Host-` en HTTPS. Redis stocke la session sous une clé hachée en SHA-256. La session
  dure 8 heures au plus ; le jeton d'accès est rafraîchi sous un verrou Redis pour éviter
  les rafraîchissements concurrents. La déconnexion révoque le jeton de rafraîchissement.
- **API : serveur de ressources JWT**, sans session (`STATELESS`). Elle vérifie la
  signature (clés JWKS de Keycloak), l'émetteur et l'audience `medhead-api`, puis
  applique les rôles :

| Point d'accès | Accès |
| --- | --- |
| `GET /actuator/health`, `/actuator/health/liveness`, `/actuator/health/readiness` | public |
| `GET /api/v1/hospitals`, `GET /api/v1/specialties` | `LECTEUR` ou `REGULATEUR` |
| `POST /api/v1/hospitals/reservations` | `REGULATEUR` |
| `GET /actuator/metrics/**` | `REGULATEUR` |
| tout le reste | refusé |

## Alternatives écartées

- **Application monopage, client public, jetons dans le navigateur** : un script injecté
  pourrait lire les jetons. Écarté au profit des sessions côté serveur.
- **Jeton JWT stocké dans un cookie** : cookie volumineux et jeton impossible à révoquer
  avant son expiration.
- **Sessions en mémoire dans le processus SvelteKit** : perdues au redémarrage et
  incompatibles avec plusieurs instances du front.
- **Authentification maison dans l'API** (comptes, mots de passe, émission de jetons) :
  sensible à écrire et à maintenir, et non standard.
- **Fournisseur d'identité en SaaS** (Auth0, Okta…) : service tiers, dépendance réseau
  et données d'identité hors de l'infrastructure du consortium. Keycloak est open source
  et s'héberge sur place.

## Conséquences

- Les jetons OAuth ne transitent jamais par le navigateur, y compris à la déconnexion.
- L'API ne connaît que des jetons standard : changer de fournisseur d'identité OIDC ne
  demande qu'une configuration (émetteur, JWKS, audience, revendication des rôles).
- Le front et l'API sont sans état et peuvent être répliqués ; les sessions sont
  partagées par Redis.
- La connexion dépend de Redis : s'il est indisponible, le front répond `503`.
- Compromis de la PoC, à lever avant la production : Keycloak tourne en mode `start-dev`,
  les échanges entre le front, l'API et Keycloak passent en HTTP sur le réseau Docker, et
  Redis utilise un seul compte pour l'API et les sessions
  ([#3](https://github.com/TheFatAndFurious/MedHead/issues/3),
  [#16](https://github.com/TheFatAndFurious/MedHead/issues/16)). L'accès de
  `REGULATEUR` aux métriques est aussi à revoir
  ([#5](https://github.com/TheFatAndFurious/MedHead/issues/5)).
