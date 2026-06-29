# Frontistirio — Announcements Brain

> **Full-stack reference** for the announcements feature: a store-user composes an **announcement**, picks an **audience** (teachers / students / parents, with **grade + class filters** exactly like the email feature), and sends it **now** or **schedules** it for later. The audience is **snapshotted to user accounts at send time**, and the announcement is designed to be delivered as a **native push notification** to the mobile apps. Spans **Frontistirio-API** (Node/Express/MongoDB) and **Frontistirio** (Angular/Ionic/NgRx). Closely mirrors the [email-services](https://github.com/KiziridisK/Frontistirio-email-services-brain-repo) feature.

---

## Scope & product decisions

Agreed up front with the user (these shape everything below):

| Decision | Choice | Consequence |
|---|---|---|
| **Delivery channel** | **Native push** (Capacitor + FCM/APNs) | No push infra exists yet (no Capacitor in the frontend, no `firebase-admin` in the backend) — it is added from scratch, in later phases. |
| **Recipient inbox / read-tracking** | **No** | Send + notify only. No recipient-facing "my announcements" page, no read/unread status. The per-recipient record only tracks *delivery* status. |
| **Audience storage** | **Snapshot of resolved `user_id`s at send time** | Same as email: the audience is "frozen". Later enrollment changes do not retroactively change who an already-sent announcement targeted. |

### Why a phased build

The feature is large and the push half needs **external setup the user must do** (a Firebase project, an Apple Developer account + APNs key, a service-account key). So it was split:

- **Phase 1 — DONE (2026-06-30):** full CRUD + audience targeting + live list. **No push yet.**
- **Phase 2 — pending (code):** device-token registry, `/devices` routes, `push.service.ts`, `pushDispatcher`, Capacitor scaffolding, `firebase-admin`.
- **Phase 3 — pending (external + wiring):** create the Firebase/APNs assets, drop them in, test end-to-end.

---

## Change log

### 2026-06-30 — Phase 1: CRUD + audience targeting (no push)

Built the whole core feature end-to-end by mirroring the email subsystem across both repos.

- New **Announcement** model (recipients snapshot, rollup status/counts, soft-delete, scheduling).
- **Audience resolution is server-side**: the compose screen sends selected entity ids + roles; the backend resolves them to `user_id`s and **re-validates** that every account belongs to the store (no cross-tenant targeting).
- Send **now** or **schedule** for later; **cancel** a still-scheduled announcement.
- Live `/announcements` list via the existing Socket.IO + NgRx pipeline (`announcementAdded/Updated/Deleted`).
- Reused the existing **`SelectRecipientsComponent`** by adding a 3rd `mode: 'announcement'` that selects by **account presence** instead of email, and carries entity/user ids.
- Home card under **Communication**, route `/announcements` (+ `/add`, `/:id`), i18n `announcements.*` in el + en.

**Verification:** backend `node --check` on every new/edited file = clean; frontend `ng build` (dev) succeeds (only pre-existing warnings, none related to announcements); i18n JSON validated.

> ⚠️ **Phase-1 honesty note:** with no push channel yet, "sending" only persists the announcement and broadcasts it to store-users over the socket. `services/announcementDispatcher.js` is a **stub** that marks every pending recipient `sent`. Phase 2 swaps the stub body for a real FCM push that sets `sent`/`failed` from the actual result — **the controller and scheduler do not change.**

---

## Architecture overview

```
COMPOSE (frontend) ──POST /announcements/send (JSON)──► Express
   │                                                     │
   │                          resolveRecipients(): entity ids → user_ids,
   │                          re-validated against the store (no cross-tenant)
   │                                                     │
   │                          create Announcement doc (status=sending|scheduled)
   │                                                     │
   │            immediate ─────► dispatchAnnouncement() ◄───── scheduled
   │                                    │                 (jobs/announcementScheduler.js, 60s tick)
   │                          PHASE 1: mark pending recipients → "sent" (STUB)
   │                          PHASE 2: push via FCM, set "sent"/"failed" per device result
   │                                    │
   │                          doc.save() → MongoDB change stream
   │                                    ▼
   └──────── socket "announcementAdded"/"announcementUpdated" ◄──┘ (watchAnnouncements)


PHASE 2 device registration (every role, on native app login):
  push.service.ts ──register FCM token──► POST /devices/register ──► DeviceToken collection
```

**Key design decision — resolve the audience on the server.** The frontend may already hold a `user_id` for students/parents, but **not for teachers** (the frontend `Teacher` model has no `user_id`). Rather than depend on what the bootstrap payload happens to include, the compose screen sends whatever identifies each pick (`userId` *or* an entity id), and the backend resolves + **re-validates** every account against the store. This is robust *and* closes the cross-tenant gap.

---

## Status lifecycle

**Per-recipient** (`announcement.recipients[].status`) → drives the colored pills on the detail page:

| status | colour | meaning |
|---|---|---|
| `pending` | 🟠 orange | created, not pushed yet (e.g. still scheduled, or Phase 1 before the stub runs) |
| `sent` | 🟢 green | push handed off to FCM successfully *(Phase 1 stub sets this directly)* |
| `failed` | 🔴 red | push send failed *(Phase 2)* |
| `canceled` | ⚪ grey | scheduled send canceled before it left |

> Note vs. email: there is **no `delivered`/`bounced`** — FCM only tells us send success/failure, not end-device delivery. `sent` is therefore the terminal "success" state (green).

**Message rollup** (`announcement.status`), recomputed from recipients by `recomputeRollup()`:
`scheduled` → `sending` (any still `pending`) → `sent` (all handed off) / `partially_failed` (some failed) / `failed` (all failed); or `canceled`.

---

## Targeting / recipient resolution (the distinctive part)

### What the frontend sends
`POST /announcements/send` body (`recipients[]`), one entry per pick — see `AnnouncementRecipientInput`:
```ts
{ role: 'student'|'teacher'|'parent',
  name?: string,
  userId?: string,      // students & parents have it client-side
  studentId?: string,   // student _id (also used to resolve a student's parent)
  teacherId?: string,   // teacher _id (teacher has NO user_id client-side → resolved server-side)
  parentKey?: 'father'|'mother' }
```

### How the backend resolves it — `resolveRecipients(rawRecipients, storeId)`
1. Collect the entity ids that need a lookup (only where no `userId` was supplied).
2. `Teacher.find({_id:{$in}, store_id, isDeleted:false}).select('user_id')` → `teacher _id → user_id`.
3. `Student.find({_id:{$in}, store_id, isDeleted:false}).select('user_id parents').populate('parents.father','user_id').populate('parents.mother','user_id')` → `student _id → { self, father, mother }`.
4. For each raw recipient, pick the `user_id` (direct, or via the maps), **de-dupe** by resolved id.
5. **Final guard:** `User.find({_id:{$in}, store: storeId, isDeleted:false})` — keep only accounts that exist **and** belong to this store. This is the cross-tenant safety net.

All three roles link to `User`: `Student.user_id`, `Teacher.user_id`, `Parent.user_id`. **The `Parent` model lives in `models/parent.js`** (its schema variable is confusingly named `gradesSchema`, but it `module.exports = mongoose.model("Parent", …)`).

---

## Backend (`Frontistirio-API`)

### Model — `models/announcement.js` (`Announcement`)
```javascript
{
  title: String (required), body: String,

  recipients: [{                       // snapshot — one sub-doc per targeted account
    user → User (required),
    name, role: 'student'|'teacher'|'parent',
    status: 'pending'|'sent'|'failed'|'canceled',
    error, lastEventAt
  }],

  store → Store, createdBy → User,

  mode: 'now'|'scheduled', sendAt: Date, sentAt: Date,
  status: 'scheduled'|'sending'|'sent'|'partially_failed'|'failed'|'canceled',
  counts: { total, pending, sent, failed },   // denormalized for list badges

  isDeleted, deletedAt, updatedBy, timestamps: true
}
// method: recomputeRollup() → refresh counts + rollup status from recipients
```

### Routes — `routes/announcements.js`
Plain `application/json` (no attachments, unlike emails), so the default `express.json()` parser handles them.

| Method | Path | Middleware | Handler |
|---|---|---|---|
| POST | `/announcements/send` | auth, isStoreUser | `sendAnnouncement` |
| GET | `/announcements/store-announcements` | auth, isStoreUser | `getStoreAnnouncements` |
| GET | `/announcements/get-announcement/:announcementId` | auth, isStoreUser | `getAnnouncement` |
| POST | `/announcements/cancel` | auth, isStoreUser | `cancelScheduled` |

> **Phase 2 will add** `routes/devices.js`: `POST /devices/register`, `POST /devices/unregister` — auth, **any role** (students/parents/teachers register their device tokens).

### Controller — `controllers/announcements.js`
- `sendAnnouncement` — validate `title` + non-empty `recipients`; for scheduled, validate `sendAt` is a future date; `resolveRecipients()` (throws `no_valid_recipients` if nothing resolves); create the doc; if immediate → `dispatchAnnouncement`, else leave `scheduled`; re-read and return fresh.
- `getStoreAnnouncements` / `getAnnouncement` — filter `isDeleted:false`.
- `cancelScheduled` — 409 if it already started sending.

### Handler (DB + change stream) — `controllers/handlers/announcements.js`
- `resolveRecipients(raw, storeId)` — see [Targeting](#targeting--recipient-resolution-the-distinctive-part).
- `createAnnouncement`, `getStoreAnnouncements`, `getAnnouncement`.
- `claimDueScheduledAnnouncement()` — atomic `findOneAndUpdate` `scheduled→sending` (prevents double-send).
- `cancelScheduledAnnouncement(id, userId)` — guarded `status:"scheduled"`; marks all recipients `canceled`, zeroes `counts.pending`.
- `watchAnnouncements(io, userSockets)` — MongoDB change stream → socket `announcementAdded` / `announcementUpdated` (full doc as `changes`) / `announcementDeleted`. Uses `fullDocumentBeforeChange:"whenAvailable"` and an `on("error")` handler that logs + re-establishes after 5s.
  - 🟢 **Emits via the AWAITED `emitToStoreUsers` helper** (same shape as `emails.js`) — i.e. it `await Promise.all(...)` the per-user `isStoreUser` checks before filtering. It deliberately does **NOT** use the buggy `Object.keys(userSockets).filter(async …)` pattern that leaks updates cross-tenant in the older watch handlers (see the [bugs brain repo](https://github.com/KiziridisK/Frontistirio-bugs-brain-repo)).

### Service — `services/announcementDispatcher.js`
`dispatchAnnouncement(doc)` — shared by immediate send + scheduler (mirrors `emailDispatcher`).
- **Phase 1 (current):** for every `pending` recipient set `status='sent'`; stamp `sentAt`; `recomputeRollup`; `save()` (→ change stream → socket).
- **Phase 2:** replace the per-recipient loop body with a call into `services/pushDispatcher.js` (look up the recipient's device tokens, `admin.messaging().sendEachForMulticast(...)`, set `sent`/`failed`, prune invalid tokens).

### Background worker — `jobs/announcementScheduler.js`
`startAnnouncementScheduler()` = dependency-free `setInterval` (60s). Each tick drains all due announcements via `claimDueScheduledAnnouncement()` (atomic) → `dispatchAnnouncement`. In-process only — move to BullMQ for multi-instance.

### `index.js` wiring
`require` + `app.use("/announcements", AnnouncementRoutes)`; `watchAnnouncements(io, userSockets)` (after `watchEmails`); `startAnnouncementScheduler()`.

### Env vars
**None for Phase 1.** Phase 3 adds the Firebase service-account (e.g. `GOOGLE_APPLICATION_CREDENTIALS` / `FIREBASE_*`) for `firebase-admin`.

---

## Frontend (`Frontistirio`)

- **Model** `models/announcement.model.ts` — `Announcement`, `AnnouncementRecipient`, `AnnouncementCounts`, status unions, `AnnouncementRecipientInput`, `SendAnnouncementPayload`.
- **Service** `services/announcements.service.ts` — `sendAnnouncement(payload)` (plain JSON), `getStoreAnnouncements`, `getAnnouncement`, `cancelScheduled`.
- **NgRx** `state/announcements/announcements.{actions,reducer}.ts` — `setAnnouncements`/`addAnnouncement`(upsert)/`updateAnnouncementFields`(merge full doc)/`deleteAnnouncement`; registered in `app.reducer.ts`; selectors `selectAllAnnouncements`, `selectAnnouncementById(id)`.
- **Socket** `services/web-socket.service.ts` — `announcementAdded`/`announcementUpdated`/`announcementDeleted` → dispatch NgRx (keeps the list live).
- **Status styling** `announcements/announcement-status.util.ts` — `recipientStatus*` + `announcementStatus*` → Ionic colour/icon/i18n-key.
- **Components** (under `app/announcements/`) — **reuse the email SCSS** via relative `styleUrls` since the markup uses the same class names:
  - `announcements.component.*` — list bound to store + status filter + search (by title) + cancel; pulls latest via service on init.
  - `announcement-item/*` — card: title, rollup badge, sent/failed/pending counts, scheduled-cancel button.
  - `add-announcement/*` — compose: title/body, recipients (via `select-recipients` modal in `'announcement'` mode), schedule now/later, send.
  - `announcement-details/*` — route `announcements/:id`; per-recipient pills (name + role + status); live via `selectAnnouncementById`.
- **Reused + extended** `select-recipients/select-recipients.component.*`:
  - new `@Input() mode: 'email' | 'sms' | 'announcement'`.
  - `searchRecipients()` now attaches ids to each row: students `userId`+`studentId`; teachers `teacherId`+`userId`(cast, may be undefined); parents carry `userId`+`studentId` (the "parents-only" branch deliberately omits the **student row's** ids so the student is a non-selectable grouping header).
  - selection now gates on **`canSelect(r)`** — announcement → `hasAccount` (userId/studentId/teacherId), else → has email. `selectedCount` / `setAllSelected` / the checkbox `[disabled]` / parent `*ngIf` all use it. Empty hint shows `recipients.no_account` in announcement mode, `recipients.no_email` otherwise.
- **Routing** `app-routing.module.ts` — `announcements`, `announcements/add`, `announcements/:id` (`:id` AFTER `add`), `RoleGuard` + `roles: ['store-user']`.
- **Home** `home/home.page.html` — card in the **Communication** section (`megaphone-outline`).
- **i18n** `assets/i18n/{el,en}.json` — `announcements.*` (status, recipient_status, compose, cancel_*, …), `menu.communication.announcements`, `communication.announcements`, `general.add.announcement`, `general.search_by.title`, `recipients.no_account`, `no.announcements`.

---

## Files touched (Phase 1)

**Frontistirio-API (new):** `models/announcement.js`, `controllers/announcements.js`, `controllers/handlers/announcements.js`, `services/announcementDispatcher.js`, `jobs/announcementScheduler.js`, `routes/announcements.js`.
**Frontistirio-API (edited):** `index.js`.

**Frontistirio (new):** `models/announcement.model.ts`, `services/announcements.service.ts`, `state/announcements/announcements.actions.ts`, `state/announcements/announcements.reducer.ts`, `announcements/announcement-status.util.ts`, `announcements/announcements.component.{ts,html}`, `announcements/announcement-item/announcement-item.component.{ts,html}`, `announcements/add-announcement/add-announcement.component.{ts,html}`, `announcements/announcement-details/announcement-details.component.{ts,html}`.
**Frontistirio (edited):** `state/app.reducer.ts`, `state/app.selectors.ts`, `services/web-socket.service.ts`, `select-recipients/select-recipients.component.{ts,html}`, `app-routing.module.ts`, `home/home.page.html`, `assets/i18n/el.json`, `assets/i18n/en.json`.

---

## Next steps / Εκκρεμότητες

### 🟠 Phase 2 — push plumbing (code, no external accounts yet)
- [ ] **`models/deviceToken.js`** — registry: `{ user → User, token (unique), platform: 'android'|'ios'|'web', store, lastSeenAt }`. Separate collection (not an array on `User`) → multi-device + easy cleanup of stale tokens.
- [ ] **`routes/devices.js` + controller** — `POST /devices/register` (upsert by token, attach `user`+`store`), `POST /devices/unregister` (on logout). Auth, **any role**.
- [ ] **`services/firebaseAdmin.js`** — lazy singleton init of `firebase-admin` from env (service-account JSON / `FIREBASE_*`). Add `firebase-admin` to `package.json`.
- [ ] **`services/pushDispatcher.js`** — `sendToUsers(userIds, {title, body, data})`: load device tokens → `admin.messaging().sendEachForMulticast(...)` → return per-token results; delete tokens that come back `messaging/registration-token-not-registered`.
- [ ] **Wire it into `dispatchAnnouncement`** — replace the Phase-1 stub loop with a `pushDispatcher` call, setting each recipient `sent`/`failed` from the result.
- [ ] **Frontend Capacitor** — add `@capacitor/{core,cli,android,ios,push-notifications}`, `capacitor.config.ts`, `npx cap add android` / `ios`.
- [ ] **Frontend `services/push.service.ts`** — native-only (no-op on web): after login, request permission → register → on `registration` send the token to `POST /devices/register`; handle `pushNotificationReceived` (foreground toast) + token refresh; unregister on logout. Call it from the same place that does `WebSocketService.connect()`.

### 🔴 Phase 3 — Firebase / APNs setup (user-provided, then wire)
- [ ] **Firebase project** → `google-services.json` (android), `GoogleService-Info.plist` (ios), and a **service-account key** for the backend.
- [ ] **iOS:** Apple Developer account, APNs auth key uploaded to Firebase, enable Push Notifications capability.
- [ ] **Backend env:** point `firebase-admin` at the service-account; deploy.
- [ ] **End-to-end test:** send an announcement → device receives the push (foreground + background); scheduled send fires once at `sendAt`.

### 🟢 Enhancements (later)
- [ ] **Recipient inbox** (if the "no inbox" decision is revisited): a `/announcements` view per recipient + read/unread tracking (would re-add a `read` field per recipient, like email's per-recipient status).
- [ ] **Edit / resend** an announcement; **delete** from the list (soft-delete endpoint).
- [ ] **Move scheduling/sending to a job queue (BullMQ)** for retries + multi-instance safety (shared with email).
- [ ] **Audience preview / count** in compose before sending (how many accounts will actually be reached).
- [ ] **Dynamic audience rule** option (store the rule, resolve at send time) as an alternative to the current snapshot.
```
