# Frontistirio — Announcements Brain

> **Κατάσταση τεκμηρίωσης** (σφραγίδα 2026-10-24) · τελευταία γραφή **2026-07-02**.
> **Δεν φέρει επαλήθευση έναντι κώδικα** — κανόνας #4 του ευρετηρίου. Από εκείνη την ημερομηνία ο κώδικας
> έχει προχωρήσει **134 commits** στο frontend και **49** στο API (όλο το repo, όχι μόνο αυτό το feature).
> Σημερινά HEAD: frontend `0f466e0` / API `cabaa2c`.
> Τι είναι **γνωστό** ότι λείπει: §5 και §5.1 του ευρετηρίου (`Frontistirio-brain-repos-index`).

> **Full-stack reference** for the announcements feature: a store-user composes an **announcement**, picks an **audience** (teachers / students / parents, with **grade + class filters** exactly like the email feature), and sends it **now** or **schedules** it for later. The audience is **snapshotted to user accounts at send time** and delivered as a **native push notification** (Capacitor + Firebase/FCM) to the mobile app. Spans **Frontistirio-API** (Node/Express/MongoDB) and **Frontistirio** (Angular/Ionic/NgRx/Capacitor). Closely mirrors the [email-services](https://github.com/KiziridisK/Frontistirio-email-services-brain-repo) feature.

> For the **build/emulator/Firebase setup procedure** (how the APK is built, how to run the Android emulator, how Firebase push was wired + every gotcha we hit), see the companion runbook: [Frontistirio-android-build-firebase-brain-repo](https://github.com/KiziridisK/Frontistirio-android-build-firebase-brain-repo).

---

## Status (2026-07-02)

| Piece | State |
|---|---|
| Phase 1 — CRUD + audience targeting + live list | ✅ done (2026-06-30) |
| Phase 2 — native push (device registry, FCM dispatch, Capacitor) | ✅ done + **tested E2E on Android emulator** (2026-07-02) |
| Student recipient inbox (home preview + full page) | ✅ done (2026-07-02) — *revised the original "no inbox" decision, for the student role* |
| iOS push | ⏳ pending (needs a Mac + Apple Developer + APNs key) |
| Production hardening (androidScheme back to https, etc.) | ⏳ see [Next steps](#next-steps--εκκρεμότητες) |

---

## Scope & product decisions

| Decision | Choice | Consequence |
|---|---|---|
| **Delivery channel** | **Native push** (Capacitor + FCM/APNs) | Added from scratch — no Capacitor/firebase-admin existed before. Android works end-to-end; iOS pending. |
| **Recipient inbox / read-tracking** | Originally **no**; **revised 2026-07-02** to add a **student** inbox (view only). | Students see their announcements (home preview + `/my-announcements`). **Still no read/unread tracking.** Endpoint is any-role, so parent/teacher inboxes are a trivial extension. |
| **Audience storage** | **Snapshot of resolved `user_id`s at send time** | Same as email: the audience is "frozen". Later enrollment changes don't retroactively change who an already-sent announcement targeted. |

---

## Change log

### 2026-07-02 — Phase 2: native push (Android E2E ✅) + student inbox

**Push plumbing (backend):**
- `models/deviceToken.js` — FCM token registry (one doc per device).
- `routes/devices.js` + `controllers/devices.js` + `controllers/handlers/devices.js` — `POST /devices/register` & `/unregister` (auth, **any role**).
- `services/firebaseAdmin.js` — lazy, **defensive** `firebase-admin` wrapper (server boots even without the package/creds → push just disabled).
- `services/pushDispatcher.js` — `sendToUsers(userIds, payload)` → `sendEachForMulticast`, aggregates per-user, prunes dead tokens.
- `services/announcementDispatcher.js` — **the Phase-1 stub is now real**: calls `pushDispatcher`; if push isn't configured it falls back to marking `sent` (dev-safe).
- `scripts/sendTestPush.js` — send a test push to a raw token (isolates "are the creds working?").
- `index.js` — mounts `/devices`. `package.json` — added `firebase-admin ^13`.

**Push plumbing (frontend):**
- Installed `@capacitor/{core,cli,android,ios,push-notifications}@6` (Capacitor 6 pairs with Ionic 8). `capacitor.config.ts`, `npx cap add android`.
- `services/devices.service.ts` (POST /devices/register,/unregister), `services/push.service.ts` (native-only: permission → register → send token; foreground toast; tap → `/home`; unregister on logout).
- Wired in `app.component.ts`: `pushService.init()` after `webSocketService.connect()`; `pushService.unregister()` in `logout()`. JWT auto-attached by the existing `auth.interceptor.ts`.

**Bug fixed — `CastError: … "[object Object]" … path "_id" for model "User"`:** the bootstrap **populates** `student.user_id`, so the compose screen was sending the whole populated `User` object (incl. `passwordHash`!) as `userId`. `String(obj)` → `"[object Object]"` → `$in` cast blew up. Fixed **both sides**: backend `resolveRecipients` added an `idOf(v)` normalizer (string | `{_id}` object → string id) used for `userId`/`studentId`/`teacherId`; frontend `select-recipients` now sends a clean string id via the existing `extractId()`. *(Separate finding: the bootstrap leaking `passwordHash` to the client is a real security bug — flagged for its own fix.)*

**Student inbox:**
- Backend `GET /announcements/my-announcements` (auth, any role) → `getUserAnnouncements(userId)`: `{ "recipients.user": userId, isDeleted:false, status ∉ [scheduled,canceled] }` with a **projection that omits the recipients array** (a recipient never sees who else got it).
- Frontend: `AnnouncementsService.getMyAnnouncements()`, reuses the `announcements` NgRx slice. New standalone `MyAnnouncementItemComponent` (read-only card, `compact` input) + `MyAnnouncementsComponent` page at `/my-announcements` (`RoleGuard ['student']`). Home preview: `home.page.ts` loads for students → top-3 rows in the student branch of `home.page.html` (`.announcements-preview-card`, styled in `home.page.scss`). i18n `announcements.{my_announcements,latest,view_all}`.
- Students get their data via **HTTP fetch** (not socket) — live in-app delivery to recipients would need a *sanitized* socket event (recipients array must stay hidden); real-time to students = the FCM push.

**Config / infra learnings (see the runbook repo for the full procedure):**
- `angular.json` — bumped `anyComponentStyle` budget `8kb → 16kb` (home.page.scss was already ~10.6 kB compiled; prod build enforces budgets, dev builds don't).
- `capacitor.config.ts` — set `server.androidScheme: 'http'` so the WebView (served from `http://localhost`) can call the local **http** backend without **mixed-content** blocking. *(Revert to `https` for production.)*
- `index.js` CORS — added `http://localhost`, `https://localhost`, `capacitor://localhost` (the native WebView origin) to both the express `allowedOrigins` and the socket.io `cors.origin`.
- Android `usesCleartextTraffic="true"` (main + debug manifest) so plain-http dev calls aren't blocked.
- `environment.ts` — for emulator builds use `http://10.0.2.2:3000` (emulator alias for the host); revert to `http://localhost:3000` for browser dev.

**Verification:** backend `node --check` clean; `ng build` (dev + prod) passes; `firebaseAdmin.isConfigured()` → true with the service account; **full E2E confirmed**: store-user sends → student's emulator receives the push (foreground toast + background tray).

### 2026-06-30 — Phase 1: CRUD + audience targeting (no push)
Built the core feature by mirroring the email subsystem: Announcement model (recipients snapshot, rollup status/counts, soft-delete, scheduling), server-side audience resolution + cross-tenant re-validation, send-now/schedule/cancel, live `/announcements` list (Socket.IO + NgRx), reused `SelectRecipientsComponent` (new `mode:'announcement'`), home card, routes, i18n. The dispatcher was a **stub** (marked recipients `sent` without a real push) — replaced in Phase 2.

---

## Architecture overview

```
COMPOSE (store-user) ──POST /announcements/send (JSON)──► Express
   │                          resolveRecipients(): entity ids → user_ids,
   │                          re-validated against the store (no cross-tenant)
   │                          create Announcement doc (status=sending|scheduled)
   │            immediate ─────► dispatchAnnouncement() ◄───── scheduled (announcementScheduler, 60s)
   │                                    │
   │                          pushDispatcher.sendToUsers(userIds, {title, body, data})
   │                                    │   look up DeviceToken(s) per user
   │                                    ▼
   │                          firebase-admin → FCM → 📱 Android (direct) / iOS (via APNs)
   │                                    │   per-user result → recipient.status = sent|failed
   │                          doc.save() → change stream → socket "announcementUpdated"
   │
DEVICE REGISTRATION (any role, on native app login):
   push.service.ts ──FCM token──► POST /devices/register ──► DeviceToken collection

STUDENT INBOX (HTTP): home preview + /my-announcements ──GET /announcements/my-announcements──► (sanitized, no recipients[])
```

**Two key design decisions:**
1. **Resolve the audience on the server.** The frontend may hold a `user_id` for students/parents but **not for teachers**. The compose screen sends whatever identifies each pick (`userId` *or* an entity id) and the backend resolves + **re-validates** every account against the store.
2. **`sendEachForMulticast` per blast, aggregated per user.** One FCM call per ≤500 tokens; the result is folded back per `user_id` so each recipient sub-doc gets `sent`/`failed`; tokens that come back `registration-token-not-registered` are deleted.

---

## Status lifecycle

**Per-recipient** (`announcement.recipients[].status`):

| status | colour | meaning |
|---|---|---|
| `pending` | 🟠 | created, not pushed yet (e.g. still scheduled) |
| `sent` | 🟢 | push handed off to FCM OK (or push not configured → dev fallback) |
| `failed` | 🔴 | every device failed **or** the user has no registered device (`no_device`) |
| `canceled` | ⚪ | scheduled send canceled before it left |

> No `delivered`/`bounced` — FCM reports send success/failure, not end-device delivery. `sent` is the terminal success state.

**Message rollup** (`announcement.status`), via `recomputeRollup()`: `scheduled` → `sending` (any `pending`) → `sent` / `partially_failed` / `failed`; or `canceled`.

---

## Targeting / recipient resolution

**Frontend sends** (`recipients[]`, one per pick — `AnnouncementRecipientInput`):
```ts
{ role:'student'|'teacher'|'parent', name?, userId?, studentId?, teacherId?, parentKey?:'father'|'mother' }
```
**Backend `resolveRecipients(raw, storeId)`:**
1. `idOf(v)` normalizes every id (string **or** populated `{_id}` object → string) — guards against the bootstrap's populated `user_id`.
2. Look up teacher/student entity ids that lack a `userId`: `Teacher.find(...).select('user_id')`; `Student.find(...).select('user_id parents').populate('parents.father'|'mother','user_id')`.
3. Resolve each pick to a `user_id`; de-dupe.
4. **Final guard:** `User.find({_id:{$in}, store: storeId, isDeleted:false})` — keep only accounts that exist **and** belong to this store (cross-tenant safety net).

All three roles link to `User`: `Student.user_id`, `Teacher.user_id`, `Parent.user_id`. **The `Parent` model lives in `models/parent.js`** (schema var confusingly named `gradesSchema`, but exports model `"Parent"`).

---

## Backend (`Frontistirio-API`)

### Models
- `models/announcement.js` (`Announcement`) — `title`, `body`, `recipients[{ user→User, name, role, status, error, lastEventAt }]`, `store`, `createdBy`, `mode`, `sendAt`, `sentAt`, `status`, `counts{total,pending,sent,failed}`, soft-delete, `recomputeRollup()`.
- `models/deviceToken.js` (`DeviceToken`) — `{ user→User (indexed), token (unique), platform:'android'|'ios'|'web', store→Store, lastSeenAt }`, timestamps. Own collection (not an array on User) → multi-device + cheap stale-token cleanup.

### Routes
`routes/announcements.js` (JSON; `express.json()`):

| Method | Path | Middleware | Handler |
|---|---|---|---|
| POST | `/announcements/send` | auth, isStoreUser | `sendAnnouncement` |
| GET | `/announcements/store-announcements` | auth, isStoreUser | `getStoreAnnouncements` |
| GET | `/announcements/my-announcements` | **auth only (any role)** | `getMyAnnouncements` |
| GET | `/announcements/get-announcement/:announcementId` | auth, isStoreUser | `getAnnouncement` |
| POST | `/announcements/cancel` | auth, isStoreUser | `cancelScheduled` |

`routes/devices.js` (auth only, any role): `POST /devices/register`, `POST /devices/unregister`.

### Controllers / handlers
- `controllers/announcements.js` — `sendAnnouncement` (validate → `resolveRecipients` → create → immediate `dispatchAnnouncement` else `scheduled`), `getStoreAnnouncements`, `getMyAnnouncements`, `getAnnouncement`, `cancelScheduled` (409 if already sending).
- `controllers/handlers/announcements.js` — `resolveRecipients` (+ `idOf`), `createAnnouncement`, `getStoreAnnouncements`, `getAnnouncement`, `getUserAnnouncements` (recipient inbox, projection omits `recipients`), `claimDueScheduledAnnouncement` (atomic scheduled→sending), `cancelScheduledAnnouncement`, `watchAnnouncements` (change-stream → socket; uses the **awaited** `emitToStoreUsers` — NOT the buggy non-awaited `filter`; see the [bugs brain repo](https://github.com/KiziridisK/Frontistirio-bugs-brain-repo)).
- `controllers/devices.js` + `controllers/handlers/devices.js` — `registerDeviceToken` (upsert by token), `removeDeviceToken` (scoped by user), `getTokensForUsers`, `removeTokens`.

### Services / jobs
- `services/firebaseAdmin.js` — lazy singleton. Guarded `require('firebase-admin')` + credential loader supporting `FIREBASE_SERVICE_ACCOUNT` (JSON string), `FIREBASE_SERVICE_ACCOUNT_PATH` (abs or **relative to project root**), individual `FIREBASE_PROJECT_ID/CLIENT_EMAIL/PRIVATE_KEY`, or `GOOGLE_APPLICATION_CREDENTIALS`. `getMessaging()`/`isConfigured()`. Never crashes the server if unconfigured.
- `services/pushDispatcher.js` — `sendToUsers(userIds, {title,body,data})` → `null` if unconfigured (caller falls back), else a `Map<userId,{successCount,failureCount,error}>`. Batches 500 tokens/call; prunes dead tokens.
- `services/announcementDispatcher.js` — `dispatchAnnouncement(doc)`: `pushDispatcher.sendToUsers(...)` for pending recipients → set `sent`/`failed` (or dev-fallback `sent` when `null`); `sentAt`; `recomputeRollup`; `save`.
- `jobs/announcementScheduler.js` — `setInterval` 60s; drains due via `claimDueScheduledAnnouncement` → `dispatchAnnouncement`. In-process only.
- `scripts/sendTestPush.js` — `node scripts/sendTestPush.js <fcmToken> ["title"] ["body"]`.

### `index.js` wiring
Mounts `/announcements` + `/devices`; `watchAnnouncements(io, userSockets)`; `startAnnouncementScheduler()`. CORS `allowedOrigins` + socket.io `cors.origin` include the native Capacitor origins.

### Env vars (`.env`)
```
FIREBASE_SERVICE_ACCOUNT_PATH=serviceAccountKey.json   # simplest; file is .gitignored (secret!)
# OR FIREBASE_SERVICE_ACCOUNT={…JSON one line…}  OR  FIREBASE_PROJECT_ID/CLIENT_EMAIL/PRIVATE_KEY
```
On boot with valid creds: `🔔 Firebase Admin initialized — push notifications enabled`. **nodemon does NOT watch `.env`** → restart manually after changing it.

---

## Frontend (`Frontistirio`)

### Announcements feature
- **Model** `models/announcement.model.ts`; **Service** `services/announcements.service.ts` (`sendAnnouncement`, `getStoreAnnouncements`, `getMyAnnouncements`, `getAnnouncement`, `cancelScheduled`).
- **NgRx** `state/announcements/*` (registered in `app.reducer`; selectors `selectAllAnnouncements`, `selectAnnouncementById`). **Socket** `announcementAdded/Updated/Deleted` in `web-socket.service.ts`. **Status util** `announcements/announcement-status.util.ts`.
- **Store-user components** (`app/announcements/`, reuse email SCSS via relative styleUrls): `announcements.component` (list), `announcement-item`, `add-announcement` (compose; recipients via `select-recipients` modal in `'announcement'` mode), `announcement-details`.
- **Student inbox components**: `my-announcement-item` (standalone read-only card, `compact` input — own scss), `my-announcements` (page at `/my-announcements`). Home preview lives inline in `home.page.html` (`.announcements-preview-card`, styled in `home.page.scss`, data from `home.page.ts` → `getMyAnnouncements` → NgRx).
- **Extended** `select-recipients.component.*`: `mode:'announcement'`; `searchRecipients()` attaches `studentId`/`teacherId`/`userId` (cleaned via `extractId`); selection gates on `canSelect(r)` (has-account vs has-email).
- **Routing** `app-routing.module.ts`: `announcements`, `announcements/add`, `announcements/:id` (store-user); `my-announcements` (student). **i18n** `assets/i18n/{el,en}.json`.

### Native push (Capacitor)
- `capacitor.config.ts` — `appId com.frontistirio.app`, `webDir www`, `server.androidScheme:'http'` (dev), `PushNotifications.presentationOptions`.
- `services/devices.service.ts` — thin HTTP to `/devices`.
- `services/push.service.ts` — **no-op on web**; on native after login: `checkPermissions`/`requestPermissions` → `register()` → `registration` listener posts the FCM token to `/devices/register`; `pushNotificationReceived` → toast (foreground); `pushNotificationActionPerformed` → navigate `/home`; `removeAllListeners` + unregister on logout.
- `app.component.ts` — `pushService.init()` after `webSocketService.connect()`; `pushService.unregister()` in `logout()`.
- `android/` — committed native project; `google-services.json` in `android/app/` (package `com.frontistirio.app`); Google Services Gradle plugin **auto-applied** by Capacitor 6's conditional in `android/app/build.gradle`.

---

## Files touched

**Frontistirio-API — new:** `models/announcement.js`, `models/deviceToken.js`, `controllers/announcements.js`, `controllers/devices.js`, `controllers/handlers/announcements.js`, `controllers/handlers/devices.js`, `services/announcementDispatcher.js`, `services/firebaseAdmin.js`, `services/pushDispatcher.js`, `jobs/announcementScheduler.js`, `routes/announcements.js`, `routes/devices.js`, `scripts/sendTestPush.js`.
**Frontistirio-API — edited:** `index.js` (routes + watchers + CORS), `package.json` (firebase-admin), `.gitignore` (service-account key).

**Frontistirio — new:** announcement model/service/state/status-util; components `announcements/{announcements,announcement-item,add-announcement,announcement-details,my-announcement-item,my-announcements}`; `services/{devices,push}.service.ts`; `capacitor.config.ts`; the whole `android/` project.
**Frontistirio — edited:** `state/app.reducer.ts`, `state/app.selectors.ts`, `services/web-socket.service.ts`, `select-recipients/select-recipients.component.{ts,html}`, `app-routing.module.ts`, `home/home.page.{ts,html,scss}`, `home/home.module.ts`, `app.component.ts`, `assets/i18n/{el,en}.json`, `angular.json` (style budget), `environment.ts`, `android/app/src/main/AndroidManifest.xml` (+ debug manifest).

---

## Next steps / Εκκρεμότητες

### 🔴 Before production
- [ ] **`androidScheme` back to `https`** in `capacitor.config.ts` + point the native build at the **https** prod API — the `http` scheme was a local-dev workaround for mixed content.
- [ ] **Fix the `passwordHash` leak** — the bootstrap populates `user_id` (and parents) without excluding secrets, shipping bcrypt hashes to the client. Exclude `passwordHash` from all user populates.
- [ ] **iOS push** — Mac + Apple Developer, `npx cap add ios`, `GoogleService-Info.plist`, APNs auth key uploaded to Firebase, Push + Background-Modes capabilities, test on a physical device.

### 🟠 Robustness / scale
- [ ] **Queue (BullMQ)** for scheduling/sending (retries + multi-instance) — shared with email.
- [ ] **Live in-app delivery to recipients** — a *sanitized* socket event (no `recipients[]`) so the student home/list update without a refresh.
- [ ] **Stale-token hygiene** beyond the `registration-token-not-registered` prune (e.g. `lastSeenAt` TTL).

### 🟢 Enhancements
- [ ] **Parent/teacher inbox** — the `/my-announcements` endpoint is already any-role; needs home/menu entry points for those roles (they currently have almost no UI).
- [ ] **Read/unread tracking** per recipient (deliberately omitted so far).
- [ ] **Edit / resend / delete** an announcement.
- [ ] **Audience preview / count** in compose (how many accounts will actually be reached).
- [ ] **Deep-link** a tapped push to a recipient announcement view (needs the inbox to expose a detail route).
```
