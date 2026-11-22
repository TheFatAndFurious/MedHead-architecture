# ADR 0002 : front SvelteKit, par dérogation à la liste imposée

## Statut

Acceptée, par **dérogation validée par le mentor** le
<!-- TODO Matt : date de validation -->.

## Contexte

La liste des technologies front imposée par le consortium comprend **Angular, React ou
VueJS**. Le front de la PoC doit permettre à un régulateur de se connecter, de choisir un
lieu et une spécialité, de réserver un lit et de voir l'hôpital retenu sur une carte.

La sécurité de l'authentification pèse sur ce choix. L'API est un serveur de ressources
OAuth 2.0 : elle attend un jeton d'accès JWT. Si le navigateur détient ce jeton, une faille
XSS suffit à le voler. La pratique recommandée pour une application web est donc qu'un
composant serveur, propre au front, garde les jetons et ne remette au navigateur qu'un
cookie de session (motif « Backend for Frontend »).

## Décision

Le front est écrit avec **SvelteKit** (Svelte 5, adaptateur Node), par dérogation à la liste
imposée, validée par le mentor.

**Critère de choix** : SvelteKit offre nativement des **routes et des actions exécutées
côté serveur** (`hooks.server.ts`, `+page.server.ts`, `+server.ts`). Le front mène donc
lui-même la connexion OpenID Connect en client confidentiel et appelle l'API depuis le
serveur. **Les jetons OAuth ne sont jamais exposés au navigateur**, sans ajouter de
service BFF séparé. S'y ajoutent :

- **la légèreté** : Svelte compile les composants en JavaScript sans moteur de DOM virtuel
  livré au navigateur ;
- **la simplicité** : un seul projet porte les pages, les formulaires et la partie
  serveur, avec des formulaires HTML qui fonctionnent sans JavaScript côté client.

## Alternatives écartées

- **Angular** : application monopage par défaut. Garder les jetons hors du navigateur
  demanderait un BFF séparé, donc un service de plus à écrire et à déployer.
- **React seul** : même limite qu'Angular. **Next.js** offrirait des routes serveur, mais
  avec un cadre plus lourd que nécessaire pour une PoC d'un seul écran.
- **VueJS** : c'est l'**alternative de repli**. Avec **Nuxt**, Vue offre aussi des routes
  serveur, ce qui permettrait de garder la même architecture (sessions côté serveur, jetons
  hors du navigateur) si la dérogation devait être retirée.

## Conséquences

- L'écart à la liste imposée est assumé et tracé par cette ADR et par la validation du
  mentor.
- Le contrat entre le front et l'API est une API REST versionnée (`/api/v1`),
  **indépendante du framework** : remplacer le front par une application Vue ou Angular ne
  demande aucune modification de l'API.
- Le front a une partie serveur (Node 24) à déployer et à superviser, et il dépend de
  Redis pour ses sessions (voir [ADR 0004](0004-authentification-keycloak-sessions-serveur.md)).
- L'écosystème de Svelte est plus petit que celui de React ou d'Angular ; les dépendances
  du front restent peu nombreuses (`openid-client`, `redis`, `maplibre-gl`).
