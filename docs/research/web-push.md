# Web Push from React Router (framework mode) + SQLite/Drizzle

Research for issue #5 (child of map issue #4). Planning input for a spec; nothing is implemented.

Repo facts checked: React Router 8.3.0 framework mode, Vite 7, better-sqlite3 + drizzle-orm, `public/` contains only `favicon.svg` and `images/`. There is no service worker or manifest today.

Each claim cites the owning source. Items marked **UNVERIFIED** came from model memory or were not stated in the fetched source, so check them before putting them in the spec.

## 1. Protocol shape

- Browser subscribes via `PushManager.subscribe({ userVisibleOnly: true, applicationServerKey })` and gets a `PushSubscription` (endpoint URL plus `p256dh` and `auth` keys). Needs a secure context and an active service worker. [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe), [W3C Push API](https://w3c.github.io/push-api/)
- `applicationServerKey` is an ECDSA P-256 public key. Once supplied, every push to that subscription must carry a VAPID JWT signed by the matching private key. It is not the encryption key. [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe), [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)
- The endpoint is a capability URL: anyone holding it can push to the app. Treat it as a secret and guard the subscribe endpoint against CSRF. [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- Call `subscribe()` and `Notification.requestPermission()` from a user gesture. Firefox has enforced this since v72. [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)

## 2. `web-push` library and VAPID keys

- Install with `npm install web-push`. API: `generateVAPIDKeys()` returns `{ publicKey, privateKey }` as URL-safe base64. `setVapidDetails(subject, publicKey, privateKey)`. `sendNotification(subscription, payload, options)`. [web-push README](https://github.com/web-push-libs/web-push)
- Generate the keys once, store them, and reuse them. A subscription is bound to the key it was created with, so changing keys breaks existing subscriptions. [web-push README](https://github.com/web-push-libs/web-push), [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)
- Plan: keep `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` and `VAPID_SUBJECT` in env vars. The private key stays server-only. The public key is exposed to the client (for example via a root loader).
- The subject must be a `mailto:` or `https:` URI. Use a real address, because Apple may reject localhost placeholders with 403 `BadJwtToken`. [web-push README](https://github.com/web-push-libs/web-push), [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)
- The payload is a string or a Buffer, so send JSON through `JSON.stringify`. The subscription object has the same shape as the browser's `subscription.toJSON()`: `{ endpoint, keys: { p256dh, auth } }`. [web-push README](https://github.com/web-push-libs/web-push)
- Options: `TTL` (seconds, default 4 weeks), `urgency` (`very-low`, `low`, `normal`, `high`), `topic` (up to 32 URL-safe base64 characters; replaces a pending message with the same topic), `timeout`, `contentEncoding`. [web-push README](https://github.com/web-push-libs/web-push), [RFC 8030 s5.2-5.4](https://www.rfc-editor.org/rfc/rfc8030)
- Under the hood: the JWT is ES256 over P-256 with `aud` set to the push service origin and `exp` at most 24h out. The header is `Authorization: vapid t=<jwt>, k=<key>`. Payloads use `aes128gcm` (RFC 8291). [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292), [RFC 8291](https://www.rfc-editor.org/rfc/rfc8291)
- Not documented in the README: supported Node versions. **UNVERIFIED**: check `engines` on npm. `web-push` is CommonJS, so confirm the import works under the repo's ESM and Vite SSR setup (spike).
- Keep the `web-push` call in server-only modules (loaders, actions, `*.server.ts`) so it never reaches the client bundle. **UNVERIFIED** against RR docs; confirm when speccing.

## 3. React Router and Vite requirements

- **Service worker.** It must be a real file at a stable, unhashed URL with scope covering the app (`/sw.js`). Vite's `public/` dir is copied as-is to the build root, unhashed and untransformed, and served at `/` in dev. A hand-written `public/sw.js` is the simplest fit. [Vite assets](https://vite.dev/guide/assets.html#the-public-directory) Whether `react-router-serve` serves `build/client/sw.js` at the root in production was not confirmed from docs (**UNVERIFIED**, spike).
- Alternative: a resource route (a module with a `loader` and no default export) returning the SW source with `Content-Type: text/javascript`. It is worse because there is no bundling or TypeScript. [RR resource routes](https://reactrouter.com/how-to/resource-routes)
- **Manifest.** Either `public/manifest.webmanifest` or a resource route (for example `app/routes/[manifest.webmanifest].ts` returning `Response.json(...)` with `Content-Type: application/manifest+json`). Resource routes handle GET via `loader` and POST/PUT/PATCH/DELETE via `action`. For external consumers return `Response`. [RR resource routes](https://reactrouter.com/how-to/resource-routes), [RR file route conventions](https://reactrouter.com/how-to/file-route-conventions), [MDN manifest](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest)
- Link it from the root layout: `<link rel="manifest" href="/manifest.webmanifest" />`. Add `crossorigin="use-credentials"` only if the manifest needs cookies. [MDN manifest](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest)
- Chromium installability needs: `name` or `short_name`, `icons` with 192px and 512px, `start_url`, `display` or `display_override`, and `prefer_related_applications` not `true`. It also needs HTTPS or localhost. A service worker is not required for installability. [MDN installable](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable)
- **Registration.** Register on the client only (a `useEffect` in `root.tsx` or `entry.client.tsx`), never during SSR. Wait on `navigator.serviceWorker.ready` before subscribing. [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)
- **Server endpoints.** Subscribe and unsubscribe are resource-route actions (POST/DELETE) that take the subscription JSON and tie it to the signed-in user (see the DevUI session note in CLAUDE.md). Sending is a server function called from existing actions.
- **Push handler in the SW.** On `push`, parse `event.data.json()` and call `event.waitUntil(self.registration.showNotification(...))`. On `notificationclick`, open or focus the target URL. On `pushsubscriptionchange`, resubscribe and POST the new subscription (see s5). [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- Dev: push requires HTTPS or localhost, so it works on `localhost` without a certificate. [MDN installable](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable)

## 4. iOS and browser support

- iOS/iPadOS 16.4 or later. The site must be added to the Home Screen with a manifest whose `display` is `standalone` or `fullscreen`. Otherwise it is only a bookmark. The permission request must come from a direct user gesture. [WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)
- The WebKit post describes Web Push on iOS only for Home Screen web apps, not plain Safari tabs. Permission is managed per web app. iOS uses APNs with no developer-program membership; allow `*.push.apple.com` if egress is filtered. [WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)
- Since iOS 16.4, Safari, Chrome, Edge, Firefox and Orion can add to the Home Screen. `beforeinstallprompt` is not supported on iOS, so the app needs its own "Share, then Add to Home Screen" instructions. [MDN installable](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable)
- macOS Safari 16 on Ventura has Web Push in the browser (same Web Push as iOS 16.4). [WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)
- The Push API is Baseline widely available since March 2023. [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) Firefox desktop cannot install PWAs, but push works in-browser. [MDN installable](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable)
- Chrome and Edge reject `subscribe()` unless `userVisibleOnly: true`. [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe) Use `true` everywhere.
- Feature-detect (`'serviceWorker' in navigator && 'PushManager' in window`) instead of sniffing the browser. [WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/) On iOS in a normal tab, `PushManager` being absent is the signal to show install instructions (design inference, **UNVERIFIED**).
- Silent push is not allowed: if the `push` handler does not show a notification, the subscription is revoked (original Web Push on Safari). [WebKit declarative push](https://webkit.org/blog/16535/meet-declarative-web-push/) Always call `showNotification`.
- Declarative Web Push (iOS/iPadOS 18.4, macOS 15.5 beta per the post) uses a JSON payload with `"web_push": 8030` and a `notification` object (`title` and `navigate` required). It needs no service worker and has no silent-push penalty. It is optional hardening, not a baseline. [WebKit declarative push](https://webkit.org/blog/16535/meet-declarative-web-push/)
- Apple's developer page ("Sending web push notifications in web apps and browsers") could not be fetched. Its specifics (status codes, Apple VAPID quirks) are **UNVERIFIED**; read it before speccing Apple-specific behavior.

## 5. Subscription expiry and 410 handling

- RFC 8030 uses **404** for an expired or unknown subscription, and **410 Gone** for delivery-receipt failure cases. Real push services are widely reported to answer 410 for dead endpoints, but that is not in the RFC text (**UNVERIFIED** per service). Treat both 404 and 410 as "subscription is gone". [RFC 8030 s6-7](https://www.rfc-editor.org/rfc/rfc8030)
- The `web-push` README does not document these codes. It only says a failed `sendNotification` rejects with an error carrying `statusCode`, `headers` and `body`. So the check is `err.statusCode === 404 || err.statusCode === 410`. [web-push README](https://github.com/web-push-libs/web-push)
- On 404 or 410, delete that row from the DB and do not retry. 429 and 5xx: retry later with backoff (**UNVERIFIED**, from memory). 400 and 403 point to a bug (bad VAPID subject, mismatched key, bad payload); log them. [RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)
- `PushSubscription.expirationTime` is often `null`. If set, the subscription will be deactivated and the browser should refresh it. [MDN expirationTime](https://developer.mozilla.org/en-US/docs/Web/API/PushSubscription/expirationTime), [W3C Push API](https://w3c.github.io/push-api/)
- `pushsubscriptionchange` fires in the service worker when a subscription is refreshed, expires or is revoked, with `oldSubscription` and `newSubscription` (null if none could be made). The spec suggests a more reliable channel than `fetch` for reporting it, such as Background Sync. [W3C Push API](https://w3c.github.io/push-api/) Also re-sync on app load: compare `getSubscription()` with the server record and re-POST when they differ (design recommendation, not from a spec).
- A subscription is unique to a service worker registration, and its options cannot change. To change `applicationServerKey`, unsubscribe and resubscribe. [W3C Push API](https://w3c.github.io/push-api/)

### Drizzle table sketch (for the spec to decide)

`push_subscriptions`: `id`, `userId` (FK), `endpoint` (unique), `p256dh`, `auth`, `expirationTime` (nullable), `createdAt`, optionally `userAgent` and `lastSuccessAt`. A user may have several devices. A schema change requires the `database-migrations` skill (generate migration, update seed, reseed).

## 6. Payload size limits

- A push service must accept at least 4096 bytes of body and may answer 413 above that. It must not return 413 at 4096 or less. [RFC 8030 s7.2](https://www.rfc-editor.org/rfc/rfc8030)
- With `aes128gcm` the overhead is an 86-byte header, 1 padding byte and a 16-byte tag, leaving at most **3993 bytes of plaintext**. [RFC 8291 s4](https://www.rfc-editor.org/rfc/rfc8291)
- Count bytes, not characters (`Buffer.byteLength`). Keep payloads small (title, short body, URL, ids). The web-push README states no limit of its own. [web-push README](https://github.com/web-push-libs/web-push)
- `TTL` is mandatory at the protocol level (missing gets a 400); the library defaults to 4 weeks. TTL 0 means deliver now or drop. [RFC 8030 s5.2](https://www.rfc-editor.org/rfc/rfc8030)

## 7. Open questions for the spec

1. Which events trigger a push, who receives it, and where does opt-in live (settings page vs. contextual prompt)?
2. DB schema and cleanup policy (delete on 404/410 only, or also prune by age).
3. Does `react-router-serve` serve `build/client/sw.js` at the root in production? (spike)
4. Does `web-push` import cleanly under the repo's ESM/Vite SSR setup? (spike)
5. iOS UX: detecting "not installed" and showing install instructions.
6. Testing: put `web-push` behind a thin module so tests can assert on payloads and 404/410 deletion without network.
7. Whether to adopt Declarative Web Push later.
