# PLAN 2 — عوداً عوداً app, accounts + sync

Connects the local app from Plan 1 to the Laravel backend. The app stays **offline-first**: Room is the
source of truth, the server is a sync peer. Nothing in Plan 1's rules changes; this plan adds a remote
module, auth, and a sync engine.

Prerequisites: Plan 1 done; backend phases 1–4 deployed (`PLAN-laravel.md`).
The contract is the backend's `openapi.yaml` (copy it to `contract/openapi.yaml` in this repo and keep it
in step). Work in vertical slices, Conventional Commits. **Ask instead of guessing.**

---

## 0. Decisions (fixed)

| Topic | Decision |
|---|---|
| HTTP | Ktor Client (KMP), kotlinx.serialization, Android engine OkHttp |
| DTOs | Hand-written in `:datasource:remote`, matching `openapi.yaml` field-for-field (`snake_case` via `@SerialName`) |
| Auth | Email + password and Google; JWT access (15 min) + refresh (60 days, rotated) |
| Google | Credential Manager (`GetGoogleIdOption`) → ID token → `POST /auth/google` |
| Verification | App works locally before verification; **sync runs only for verified accounts** |
| Before login | App fully usable; on first login everything local is uploaded |
| Account | Logout, delete (immediate), edit name |

### Decisions I made — owner should confirm

- **Email verification and password reset use 6-digit codes** (backend plan made the same call).
- **Logout** first tries a final push. If unsynced changes remain (offline), warn and let the user choose
  "log out anyway" (local data is wiped) or cancel. After logout the device returns to a clean,
  local-only heart.
- **Logging into an account that already has data** merges the local heart into it (same LWW rules), no
  prompt. Users who want a clean slate can use Clear (which backs up first).
- Tokens are stored in DataStore for now; moving them to encrypted storage comes with the planned
  encryption work.

---

## 1. New modules and pieces

```
:datasource:remote   Ktor client, DTOs, API service, auth token handling      (new)
:data                + SyncEngine, AuthRepository, mappers DTO ↔ domain       (extended)
:presentation        + auth screens, account screen, sync status              (extended)
:androidApp          + Credential Manager actual, WorkManager scheduler actual (extended)
```

New `expect/actual` (interfaces in `:domain` or `:data`):
- `GoogleIdTokenProvider` — Android: Credential Manager. iOS later: Google Sign-In SDK.
- `SyncScheduler` — Android: WorkManager. iOS later: `BGTaskScheduler`.

Local tables added: `sync_state(userId, lastPulledSeq)` and the signed-in `User` + tokens in DataStore.

---

## 2. Networking

- Base URL per build type (`debug` → owner's staging or local, `release` → production). Ask the owner for
  the URLs.
- Default headers on every call: `X-Device-Id` (the Plan 1 device id), `X-Platform: android`,
  `X-App-Version`.
- Ktor `Auth` plugin with bearer tokens: `loadTokens` from DataStore; `refreshTokens` calls
  `/auth/refresh` with a mutex so parallel 401s refresh once; on refresh failure → signed-out state
  (local data stays until the user logs in again or logs out).
- Error mapping: the contract's `{error:{code,message,fields}}` → a sealed `ApiError`
  (`Network`, `Unauthorized`, `EmailNotVerified`, `Validation(fields)`, `RateLimited`, `Server`).
- Timeouts: connect 10 s, request 30 s.

---

## 3. Sync engine (`:data`, commonMain)

One `SyncEngine.sync()` that is safe to call anytime; a mutex guarantees a single run at a time.

```
sync():
  if not signed in or email not verified → return
  PUSH  loop: take up to 500 dirty rows (shots, categories, settings, backups incl. payload)
        → POST /sync/push
        → for accepted ids: dirty = false (only if the row was not modified again meanwhile —
          compare the clock fields you sent with the current row)
        → for rejected ids: set dirty = false and log the reason (bad data must not retry forever)
  PULL  loop: GET /sync/pull?since=lastPulledSeq
        → apply each item with the Plan 1 LWW merge
        → set seq on applied rows; save next_since as lastPulledSeq
        until has_more = false
```

Rules:
- **Push before pull**, so the server already has our latest clocks when we merge its changes.
- **Pull merge:** if a local row is dirty and its clock is newer, keep local (it will be pushed next run).
  On equal clocks, **take the server's version** — the server is the tie authority.
- Applying pulled data must **not** mark rows dirty.
- Backups pull only metadata; fetch the payload with `GET /backups/{id}` when the user restores or
  exports one that is not local.
- The stain cache rebuilds when synced changes alter the visible shots (the renderer already rebuilds on
  data change).

### When it runs
- App start and every return to foreground.
- 2 s after the last local change (debounced; a burst of throws becomes one push).
- When connectivity returns.
- Every 6 h in the background.

Android `SyncScheduler`: WorkManager unique work (`KEEP` for periodic, `REPLACE` for debounced one-shot),
`NetworkType.CONNECTED` constraint, exponential backoff.

### Sync status in UI
A small, quiet indicator on the Heart screen: synced / syncing / offline with pending changes /
needs verification. No blocking dialogs.

---

## 4. Auth and account

### Screens
1. **Account entry** (from Settings): «تسجيل الدخول» / «إنشاء حساب» / «المتابعة باستخدام Google».
2. **Register:** name, email, password (min 8).
3. **Login:** email, password, «نسيت كلمة السر؟».
4. **Verify email:** 6-digit code input, resend with countdown. Reachable later from the sync status.
5. **Forgot / reset password:** email → code + new password.
6. **Account:** name (editable), email, verification state, last sync time, «مزامنة الآن», logout,
   delete account.

### Flows
- **Register / login / Google** → save tokens + user → if verified, trigger sync immediately (this is the
  "upload on first login": every Plan 1 row is still dirty).
- **Google:** `GetGoogleIdOption` with the **web** server client id from config (ask the owner);
  `filterByAuthorizedAccounts = false` on first sign-in. Send the ID token to `/auth/google`.
- **Delete account:** confirm with password (email accounts) or a fresh Google ID token → `DELETE /me` →
  wipe local DB, tokens, sync state → back to a clean local heart with seeded categories.
- **Logout:** see decisions above; revoke via `/auth/logout`, then wipe.

---

## 5. Contract fidelity

- DTOs mirror `openapi.yaml` exactly; mapping DTO ↔ domain lives in `:data`.
- Test: decode every file in the backend's `contract/fixtures/*.json` (copied into this repo) into the
  DTOs, and encode a `PushRequest` sample and compare its keys to the spec examples.
- When the backend changes the contract, update `contract/openapi.yaml` first, then DTOs, in one commit.

---

## 6. Tests (only what matters)

- Sync engine against a fake API: first-login upload of all dirty rows; push → pull convergence of two
  simulated devices (the "two offline phones" case: device A throws at 3:30 and 3:35, device B at 3:32 and
  3:36, B syncs first → both end with the same four shots ordered 3:30, 3:32, 3:35, 3:36); undo on one
  device vs redo on the other resolves by `stateChangedAt`; a row edited during a push stays dirty.
- Auth: refresh on 401 happens once for parallel calls; refresh failure → signed out without data loss.
- Fixture decoding (§5).

---

## 7. Phases

1. **Remote module:** Ktor client, headers, DTOs, error mapping, fixture tests.
2. **Auth:** repository, token storage, refresh, register/login/verify/forgot/reset screens.
3. **Google:** Credential Manager actual + `/auth/google`.
4. **Sync engine:** push/pull/merge, `sync_state`, tests including the two-device scenario.
5. **Scheduling + status:** WorkManager actual, triggers, sync indicator, «مزامنة الآن».
6. **Account:** account screen, logout rules, delete account, backups payload fetch.
7. **End-to-end check against the real backend:** two emulators, offline throws on both, reconnect,
   identical hearts.
