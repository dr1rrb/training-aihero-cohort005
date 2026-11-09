# Contraintes techniques du Web Push pour Cadence

Recherche pour l'issue #3 (enfant de la map #1). Date : 2026-11-09.
Ce document rapporte des faits et des contraintes, avec leurs sources. Il ne tranche pas les décisions produit (UX de la demande de permission, envoi synchrone ou différé) : il en expose les compromis.

## Constat sur le dépôt

- **Version de React Router** : le ticket parle de React Router 7, mais `package.json` épingle `react-router`, `@react-router/dev`, `@react-router/node` et `@react-router/serve` en **8.3.0**. Le mode framework est SSR (`ssr: true` dans `react-router.config.ts`). Les routes sont déclarées dans `app/routes.ts`. Les routes `api/*` existantes (`api.switch-user.ts`, `api.video-tracking.ts`) sont déjà des routes de ressource.
- **Base** : un seul fichier SQLite (`data.db`) via `better-sqlite3` + Drizzle, en mode WAL (`app/db/index.ts`). Les tests créent une base `:memory:` migrée avec les migrations Drizzle (`app/test/setup.ts`).
- **Pas de file de jobs, pas d'envoi d'email réel** : `courses.$slug.purchase.tsx` contient seulement `// Would send welcome email here`. Aucun envoi sortant n'existe donc à imiter.
- **Pas de service worker ni de manifeste** : `public/` ne contient que `favicon.svg` et `images/`. `app/root.tsx` n'a ni `<link rel="manifest">` ni enregistrement de service worker.
- **`docs/adr/0006` n'existe pas** dans ce dépôt (aucun dossier `docs/adr/`).
- **Production** : `npm start` lance `react-router-serve`. Ce serveur Express sert `build/client/assets` avec `immutable, maxAge: "1y"`, puis la racine de `build/client` sans `maxAge` (`node_modules/@react-router/serve/dist/cli.js`, lignes 115-120).
- **Session** : cookie `sameSite: "lax"` (`app/lib/session.ts`).

## 1. VAPID, bibliothèque serveur, format d'abonnement

### VAPID (RFC 8292)

- La clé de signature doit être une clé ECDSA P-256. Le JWT est signé en ES256. Il est envoyé dans `Authorization: vapid t=<JWT>, k=<clé publique base64url>`. ([RFC 8292](https://www.rfc-editor.org/rfc/rfc8292.html))
- Claims : `aud` (origine du service push) obligatoire. `exp` obligatoire, et « MUST NOT be more than 24 hours from the time of the request ». `sub` facultatif, mais s'il est présent il SHOULD être un URI `mailto:` ou `https:` de contact. ([RFC 8292](https://www.rfc-editor.org/rfc/rfc8292.html))
- L'abonnement est lié à la clé publique passée à `subscribe()` (`applicationServerKey`). Si un message est signé avec une autre clé, le service push le rejette (403 possible). ([RFC 8292](https://www.rfc-editor.org/rfc/rfc8292.html), [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe))
- **Conséquence** : la paire VAPID se génère une seule fois et se garde (variables d'environnement). En changer invalide tous les abonnements stockés. Le README de `web-push` le dit aussi : générer les clés « once, store them and use them for all future messages ». ([web-push README](https://github.com/web-push-libs/web-push))
- La clé publique doit aussi arriver au navigateur (loader ou constante injectée), car `subscribe()` en a besoin.

### Bibliothèque `web-push`

- **La v4.0.0 est sortie le 2026-11-06** (registre npm, commit « 4.0.0 » sur [web-push-libs/web-push](https://github.com/web-push-libs/web-push)). C'est un package **ESM** (`"type": "module"`), avec `engines.node >= 16`. Il exporte nommément `generateVAPIDKeys`, `setVapidDetails`, `sendNotification`, `WebPushError`, etc. (`package.json` et `src/index.js` du dépôt). Le projet est lui aussi `"type": "module"`, donc compatible.
- **Types** : le package n'embarque pas de types. `@types/web-push` est en 3.6.4 et décrit l'API v3. Il faut s'attendre à des écarts possibles avec la v4, ou bien épingler la v3.6.7.
- API ([README](https://github.com/web-push-libs/web-push)) : `sendNotification(subscription, payload, options)` renvoie une Promise. Options utiles :
  - `TTL` : par défaut **4 semaines** (`DEFAULT_TTL = 2419200` dans `src/web-push-lib.js`) ;
  - `urgency` : `very-low | low | normal | high` ;
  - `topic` : 32 caractères base64url au plus ;
  - `timeout` : en ms. **Aucun timeout par défaut** : il n'est appliqué que s'il est fourni (`src/web-push-lib.js`) ;
  - `contentEncoding` : `aes128gcm` par défaut ;
  - `vapidDetails`, `proxy`, `agent`, `headers`.
- Erreurs : toute réponse hors 2xx rejette avec `WebPushError { statusCode, headers, body, endpoint }` (`src/web-push-error.js`).
- Le chiffrement du payload (RFC 8291) est fait par la bibliothèque. Il exige `keys.p256dh` et `keys.auth` de l'abonnement.
- CLI : `web-push generate-vapid-keys [--json]` et `web-push send-notification --endpoint=... --key=... --auth=... --payload=... --vapid-subject=... --vapid-pubkey=... --vapid-pvtkey=...`.

### Format d'un abonnement à stocker

- `web-push` attend « the same output as JSON.stringify'ing a PushSubscription in the browser », soit `{ endpoint, expirationTime, keys: { p256dh, auth } }`. ([README](https://github.com/web-push-libs/web-push))
- `endpoint` est une **URL de capacité** : la connaître suffit pour envoyer des messages. Elle doit rester secrète. ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))
- `expirationTime` vaut un timestamp en ms, ou `null`. ([W3C Push API §3.4, §8](https://www.w3.org/TR/push-api/))
- « Each subscription is unique to a service worker. » ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)) Un même utilisateur a donc un abonnement par navigateur et par appareil.
- Une table SQLite minimale en découle : `endpoint` (unique), `p256dh`, `auth`, `expirationTime` nullable, `userId` (FK), dates de création et de dernier succès. Le choix du schéma relève du spec.
- MDN insiste : « it is vitally important that you protect against CSRF/XSRF issues » pour l'endpoint qui enregistre un abonnement. ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)) Le cookie `sameSite: "lax"` bloque déjà les POST cross-site portant le cookie.

## 2. Service worker dans React Router (8.3 ici)

- React Router n'a pas de support intégré des service workers. La voie la plus simple est un fichier statique `public/sw.js`. Vite sert `public/` à la racine `/` en dev et le copie « as-is » à la racine de la sortie de build, sans hachage. ([Vite, Static Asset Handling](https://vite.dev/guide/assets.html)) Ici, la sortie est `build/client/`, servie sans `maxAge` par `react-router-serve` (voir le constat).
- **Scope** : par défaut, le dossier du script. Il ne peut pas être plus large sans l'en-tête `Service-Worker-Allowed`. ([MDN register](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register)) Avec `/sw.js`, le scope `/` couvre toute l'app sans en-tête spécial.
- **Contexte sécurisé** : HTTPS, ou `localhost` / origine « potentially trustworthy ». Sinon `SecurityError`. ([MDN register](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register)) `npm run dev` sur `http://localhost` suffit donc en local.
- Options de `register()` : `type: "module"` (ESM dans le worker) ou `"classic"` (défaut), et `updateViaCache` (`all | imports | none`). ([MDN register](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register))
- L'enregistrement est forcément côté client (`navigator.serviceWorker` n'existe pas en SSR). Il se fait dans un `useEffect` ou dans un gestionnaire de clic.
- Côté serveur, l'endpoint d'abonnement se fait naturellement en **route de ressource** : un module qui exporte `action` sans composant par défaut. Un `fetch()` simple vers cette route ne déclenche pas la revalidation des loaders, alors que `useFetcher`/`<Form>` la déclenchent. ([React Router, Resource Routes](https://reactrouter.com/how-to/resource-routes)) C'est le même modèle que `api.video-tracking.ts`.
- Dans le worker :
  - l'événement `push` doit appeler `event.waitUntil(self.registration.showNotification(...))`. Sinon Chrome affiche « This site has been updated in the background ». ([web.dev, Handling messages](https://web.dev/articles/push-notifications-handling-messages)) ;
  - le lien profond se gère dans `notificationclick` (`clients.openWindow`/`focus`).

## 3. Support navigateur et permission

| Navigateur | Support Push API | Contraintes |
| --- | --- | --- |
| Chrome / Edge | depuis 50 ([caniuse](https://caniuse.com/push-api)) | `userVisibleOnly: true` obligatoire, sinon la Promise est rejetée ([MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)). Pas de quota de messages ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)). |
| Firefox | depuis 44 ([caniuse](https://caniuse.com/push-api)) | Depuis Firefox 72, `Notification.requestPermission()` et `PushManager.subscribe()` « can now only be called in response to a user gesture such as a click » ([MDN Firefox 72](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/72)). Quota de messages pour les push **sans** notification visible ; ceux qui affichent une notification en sont exemptés ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)). |
| Safari macOS | Safari 16 sur **macOS Ventura (13)** ou plus ([WebKit, Meet Web Push](https://webkit.org/blog/12945/meet-web-push/)). caniuse note un support partiel de 16.1 à 17.6 et complet à partir de 18.0. | Geste utilisateur explicite requis. `userVisibleOnly` obligatoire : chaque push doit afficher une notification, sinon l'abonnement est **révoqué**. Pas besoin de compte Apple Developer. Endpoints sur `*.push.apple.com`. |
| Safari iOS / iPadOS | **16.4+**, uniquement pour une **web app ajoutée à l'écran d'accueil** ([WebKit, Web Push for Web Apps on iOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)) | Il faut un manifeste avec `display: standalone` ou `fullscreen`, sinon l'ajout donne un simple signet. La permission doit suivre une interaction directe (tap sur un bouton). Dans un onglet Safari ordinaire, pas de push. Cadence n'a **pas de manifeste** aujourd'hui. |

- **Contexte sécurisé** partout : `subscribe()` et `requestPermission()` ne marchent qu'en HTTPS. ([MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe), [MDN requestPermission](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static))
- **Permission** : `subscribe()` demande la permission « push ». Si elle est `denied`, il rejette avec `NotAllowedError`. Une permission déjà accordée n'a pas à être reconfirmée. ([W3C Push API §7.1, subscribe steps](https://www.w3.org/TR/push-api/)) Les valeurs de `Notification.permission` sont `default | granted | denied`, et `default` se comporte comme `denied`. ([MDN requestPermission](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static))
- **Après un refus**, le site ne peut plus redemander : « if the user *blocks* the permission request, your web app can't ask for permission again ». ([web.dev, Permission UX](https://web.dev/articles/push-notifications-permissions-ux))
- **Geste utilisateur** : il est obligatoire sur Firefox et Safari, et recommandé partout par MDN (« should be done in response to a user gesture »). Le plus petit dénominateur commun est donc : demande déclenchée par un clic.
- **Options UX documentées** (la décision revient au spec) :
  - ne jamais demander au chargement de la page ;
  - demander « at a time when the benefit is obvious » ;
  - pattern « double permission » : une boîte de dialogue maison d'abord, puis l'invite du navigateur ;
  - case à cocher dans les réglages, qui déclenche l'invite au clic.
  - ([web.dev, Permission UX](https://web.dev/articles/push-notifications-permissions-ux)) La page `/settings` existe déjà et pourrait accueillir cette case.
- **Declarative Web Push** (Safari iOS/iPadOS 18.4, macOS 15.5) : un payload JSON avec `"web_push": 8030`, un objet `notification` (`title` obligatoire) et `navigate` s'affiche **sans service worker**. Le même payload reste compatible avec le push classique via service worker. ([WebKit, Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/)) C'est une option pour le format du payload, mais pas un remplacement du service worker sur les autres navigateurs.

## 4. Cycle de vie des abonnements

- « Subscriptions have a limited lifetime ». Le service push « MAY expire a subscription at any time ». ([RFC 8030 §2, §7.3](https://www.rfc-editor.org/rfc/rfc8030.html)) Aucune durée n'est normée. `expirationTime` est souvent `null` (exemple du README `web-push`).
- Le navigateur ou le service push peut **rafraîchir** un abonnement à tout moment. L'événement `pushsubscriptionchange` est alors envoyé au service worker, avec `oldSubscription` (qui « SHOULD NOT be used anymore ») et `newSubscription` (`null` si aucun nouvel abonnement n'a pu être établi). ([W3C Push API §3.4.2, §10.4](https://www.w3.org/TR/push-api/)) Le worker doit alors renvoyer le nouvel abonnement au serveur.
- Codes de réponse du service push ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html), [web.dev, Web Push Protocol](https://web.dev/articles/push-notifications-web-push-protocol)) :
  - **201** : accepté, mais « does not indicate that the message was delivered » ;
  - **404** : abonnement expiré (« MUST return a 404 », RFC 8030 §7.3). Il faut le supprimer ;
  - **410** : abonnement invalide (désabonné), « should be removed from application server » (web.dev). Dans la RFC, 410 sert aussi aux reçus de livraison. En pratique, on supprime sur 404 comme sur 410 ;
  - **400** : en-têtes invalides (par exemple TTL absent) ;
  - **403** : JWT VAPID invalide ou mauvaise clé (RFC 8292) ;
  - **413** : payload trop gros. Un service push ne peut pas refuser un corps de **4096 octets ou moins** (RFC 8030 §7.2) ;
  - **429** : limite de débit, avec `Retry-After` recommandé (RFC 8030 §8.4).
- **Plusieurs appareils** : un abonnement par service worker, donc par navigateur et par appareil. Le modèle est 1 utilisateur vers N abonnements, et un envoi « à une personne » est une boucle sur N endpoints.
- L'utilisateur peut aussi se désabonner côté client (`unsubscribe()`), ce qui donne ensuite des 410. Il faut aussi prévoir le changement d'utilisateur sur un même navigateur, par exemple via le panneau DevUI : le navigateur garde l'abonnement, mais la ligne en base est liée à l'ancien `userId`.

## 5. Envoi synchrone ou différé

Faits techniques :

- Chaque envoi est une requête HTTPS POST vers le service push d'un appareil (FCM, Mozilla autopush, `*.push.apple.com`). Elle inclut le chiffrement (ECDH + AES-GCM) et la signature JWT. Un événement qui touche N personnes × M appareils fait N×M requêtes.
- Un 201 signifie seulement « accepté » : le service push stocke le message et le livre selon `TTL` et `Urgency`. ([RFC 8030 §5](https://www.rfc-editor.org/rfc/rfc8030.html)) La livraison à l'appareil est donc déjà asynchrone, quel que soit le mode d'envoi côté Cadence.
- `TTL` est obligatoire dans le protocole. `TTL: 0` veut dire livraison immédiate ou abandon. `web-push` met 4 semaines par défaut. `Topic` remplace un message en attente qui a le même topic. `Urgency` vaut `normal` par défaut. ([RFC 8030 §5.2-5.4](https://www.rfc-editor.org/rfc/rfc8030.html))
- `web-push` n'a **pas de timeout par défaut** : sans l'option `timeout`, un service push lent bloque la Promise.
- Les limites de débit (429) ne sont pas chiffrées dans les specs. Firefox applique un quota aux push sans notification visible seulement. Chrome n'a pas de limite selon MDN.

Compromis pour ce stack (pas de file de jobs) :

| Option | Pour | Contre |
| --- | --- | --- |
| **Synchrone dans l'action** (`await Promise.allSettled(...)` avant de répondre) | Le plus simple. Les erreurs et le nettoyage 404/410 se font dans le même flux. Facile à tester. | La latence de l'action dépend du service push le plus lent : il faut un `timeout`. La latence grandit avec N×M (par exemple une annonce à tous les étudiants d'un cours). Aucune relance après un échec transitoire (429/5xx). |
| **Fire-and-forget en mémoire** (lancer la Promise sans l'attendre) | Réponse HTTP immédiate, aucune infra en plus. | Envois perdus si le process redémarre. Erreurs hors du cycle de la requête : il faut les attraper, sinon `unhandledRejection`. Plus dur à tester de façon déterministe. |
| **Outbox SQLite + boucle dans le process** (table de messages en attente, `setInterval` ou drainage après chaque commit) | Durable : survit à un redémarrage. Permet relances et `Retry-After`. Réutilise le fichier SQLite existant ; en WAL, un écrivain à la fois suffit à ce volume. | Nouvelle table et boucle de fond à écrire. Avec plusieurs instances du serveur, il faut verrouiller ou réclamer les lignes. Le premier job asynchrone de l'app. |

Le choix revient au spec (map #1, « Mode d'envoi »).

## 6. Tester

- **Unitaire et intégration (Vitest, CI)** : isoler l'envoi derrière une petite interface (par exemple un module `push.server.ts`) et la simuler. Ce qui est testable sans navigateur :
  - quels abonnements sont ciblés pour un événement ;
  - le contenu du payload, dont la limite de 4096 octets ;
  - la suppression des abonnements sur un `WebPushError` 404/410 ;
  - l'enregistrement et la mise à jour d'un abonnement via l'action.
  Le projet a déjà `createTestDb()` (SQLite `:memory:` migrée), qui suffit pour ces tests.
- **Bout en bout réel** : le dépôt `web-push` lui-même teste contre de vrais services push avec Playwright. Il lance Chromium en profil persistant `headless: false`, et Firefox avec les préférences `dom.push.serverURL`, `dom.push.testing.ignorePermission`, `notification.prompt.testing*`. Il accorde la permission via `context.grantPermissions(['notifications'])`. ([test/testBrowsers.js](https://github.com/web-push-libs/web-push/blob/master/test/testBrowsers.js)) Cela demande un accès réseau sortant aux services push et un navigateur non headless (xvfb en CI). C'est faisable, mais lourd et dépendant de services tiers.
- **Manuel en local** :
  - `npm run dev` sur `http://localhost` est un contexte sécurisé, donc Chrome et Firefox desktop marchent sans HTTPS ;
  - la DevTools Chrome (Application > Service workers) a un bouton **Push** qui « emulates a push notification without a payload » ([Chrome DevTools](https://developer.chrome.com/docs/devtools/progressive-web-apps)). Pratique pour tester le worker sans serveur ;
  - la CLI `web-push send-notification` envoie un vrai push vers un abonnement copié depuis la console.
- **iOS** : il faut HTTPS (pas `localhost` sur l'appareil), un manifeste `standalone` et l'ajout à l'écran d'accueil. Un tunnel HTTPS vers le serveur de dev est nécessaire.
- **DevUI** : le panneau (`app/components/dev-ui.tsx`) permet de changer d'utilisateur. On pourrait y ajouter un déclencheur d'envoi de test ; c'est une décision du spec.

## Sources

- RFC 8030, Generic Event Delivery Using HTTP Push : https://www.rfc-editor.org/rfc/rfc8030.html
- RFC 8292, VAPID : https://www.rfc-editor.org/rfc/rfc8292.html
- RFC 8291, Message Encryption for Web Push (appliqué par `web-push`) : https://www.rfc-editor.org/rfc/rfc8291.html
- W3C Push API : https://www.w3.org/TR/push-api/
- MDN :
  - Push API : https://developer.mozilla.org/en-US/docs/Web/API/Push_API
  - PushManager.subscribe : https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe
  - Notification.requestPermission : https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static
  - ServiceWorkerContainer.register : https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register
  - Firefox 72 : https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/72
- WebKit :
  - Meet Web Push : https://webkit.org/blog/12945/meet-web-push/
  - Web Push for Web Apps on iOS and iPadOS : https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
  - Meet Declarative Web Push : https://webkit.org/blog/16535/meet-declarative-web-push/
- web.dev :
  - Web Push Protocol : https://web.dev/articles/push-notifications-web-push-protocol
  - Permission UX : https://web.dev/articles/push-notifications-permissions-ux
  - Handling messages : https://web.dev/articles/push-notifications-handling-messages
- web-push (README et sources v4.0.0) : https://github.com/web-push-libs/web-push
- React Router, Resource Routes : https://reactrouter.com/how-to/resource-routes
- Vite, Static Asset Handling : https://vite.dev/guide/assets.html
- Chrome DevTools, PWA : https://developer.chrome.com/docs/devtools/progressive-web-apps
- caniuse, Push API : https://caniuse.com/push-api
