# PLAN 1 — عوداً عوداً app, local only (Compose Multiplatform)

The full app working offline with a local database only. No network, no accounts.
The data layer is **built sync-ready from day one** so Plan 2 only adds a remote module and a sync
engine, without rewriting anything here.

Read first: `heart-handoff/HANDOFF.md` (rendering, animation, assets) and `CLAUDE.md`.
The Android project already exists with `libs.versions.toml`; extend it, don't recreate it.
Work in vertical slices, Conventional Commits. **Ask instead of guessing** on anything not written here.

---

## 0. Decisions (fixed)

| Topic | Decision |
|---|---|
| UI | Compose Multiplatform, Material 3, Navigation Compose (JetBrains multiplatform artifact) |
| Targets | **Android only** for now. Modules are KMP with `androidTarget()` only; iOS targets get added later |
| DI | Koin |
| DB | Room KMP (bundled SQLite driver) |
| Prefs | DataStore (KMP) for device-level values (`device_id`) |
| JSON | kotlinx.serialization |
| Language | Arabic only, forced RTL regardless of device locale |
| Tests | Only what protects logic: undo/redo/jump/clear rules, LWW merge, export/import round-trip, UUIDv7 |

### Decisions I made — owner should confirm

- **No Android imports in `commonMain`.** With only an Android target the compiler won't always catch
  this, so keep platform code in `androidMain` or `androidApp` behind interfaces. This is what makes
  adding iOS later cheap.
- **No use-case class for trivial reads**; ViewModels read repositories directly for observation.

---

## 1. Modules

```
:domain              pure Kotlin (commonMain). Models, repository interfaces, use cases, UUIDv7, rules
:data                repositories, export/import, backups, mappers. Depends on :domain, :datasource:local
:datasource:local    Room KMP database + DAOs + DataStore
:presentation        Compose Multiplatform UI, ViewModels, heart renderer, resources
:androidApp          Activity, Koin start, Android actuals (sound, file picker, haptics)
```

Dependency direction: `presentation → domain ← data → datasource:local`; `androidApp` wires everything.
`:datasource:remote` does **not** exist in this plan.

`expect/actual` needed in this plan: sound player (Android: `SoundPool`), file save/open for
export/import (Android: Storage Access Framework), and anything else that turns out platform-specific.
Keep each behind a small interface in `:domain` or `:presentation`, implemented in `androidApp`.

---

## 2. Domain model

```kotlin
enum class Ink { WHITE, BLACK }

data class Shot(
    val id: Uuid,                 // UUIDv7, generated here
    val color: Ink,
    val x: Float, val y: Float,   // viewBox 400×460
    val r: Float,                 // 3..40
    val intensity: Int,           // 10..100
    val shotAt: Instant,
    val undoneAt: Instant?,
    val deletedAt: Instant?,
    val stateChangedAt: Instant,  // LWW clock for undoneAt + deletedAt
    val note: String?,            // ≤ 500
    val categoryId: Uuid?,        // kind must equal color
    val metaChangedAt: Instant,   // LWW clock for note + categoryId
    val deviceId: Uuid,
)

data class Category(
    val id: Uuid, val kind: Ink, val name: String,
    val color: String?,           // "#RRGGBB"
    val icon: String?,            // key from the app's fixed icon set
    val presetKey: String?,       // seeded defaults only
    val sort: Int,
    val changedAt: Instant, val deletedAt: Instant?,
)

data class Settings(val dotSize: Int, val intensity: Int, val soundEnabled: Boolean)

data class BackupMeta(val id: Uuid, val reason: BackupReason, val shotsCount: Int, val createdAt: Instant)
enum class BackupReason { CLEAR, IMPORT_REPLACE, RESTORE }
```

- **UUIDv7:** small util in `:domain` (48-bit Unix ms + random), with a test that ids are unique and
  sort by time.
- **Device id:** generated once (UUIDv7), stored in DataStore, stamped on every shot.
- **Time:** inject a `Clock` so rules are testable.

### Heart rules (one `HeartRules` / use cases, fully unit-tested)

- **Visible shots** = `deletedAt == null && undoneAt == null`, ordered by `shotAt`, then `id`.
- **Redo stack** = shots with `undoneAt != null && deletedAt == null`.
- **Throw:** every shot in the redo stack gets `deletedAt = now` (truncates redo, like the prototype),
  then insert the new shot. All state changes set `stateChangedAt = now`.
- **Undo:** the last visible shot gets `undoneAt = now`.
- **Redo:** the **earliest** shot in the redo stack gets `undoneAt = null`.
- **Jump to shot N:** every visible shot after N gets `undoneAt = now`.
- **Clear:** create a backup (`CLEAR`), then every non-deleted shot gets `deletedAt = now`. Not undoable;
  restorable from Backups.
- **Edit note/category:** updates `note` / `categoryId`, sets `metaChangedAt = now`. Shot geometry and
  color never change.
- **Category kind must match shot color**; assigning a deleted category is not allowed, but shots
  already pointing to a deleted category keep it and show its name.

### LWW merge (used by import now, by sync in Plan 2)

One pure function per entity, in `:domain`, mirrored exactly on the server:

- shot: insert if unknown; otherwise take incoming `(undoneAt, deletedAt)` if its `stateChangedAt` is
  newer, and incoming `(note, categoryId)` if its `metaChangedAt` is newer. Geometry is never overwritten.
- category / setting: take incoming if its `changedAt` is newer.
- Ties (equal clocks): keep the existing value. In Plan 2 the sync pull overrides this: on a tie the server's version wins.

---

## 3. Local database (Room KMP)

Entities mirror the domain plus sync bookkeeping that stays unused until Plan 2:

| table | columns |
|---|---|
| `shots` | all `Shot` fields (ids/instants as String/Long) + `seq: Long?` + `dirty: Boolean` |
| `categories` | all `Category` fields + `seq: Long?` + `dirty: Boolean` |
| `settings` | `key` PK, `value` (JSON string), `changedAt`, `seq: Long?`, `dirty: Boolean` |
| `backups` | `id`, `reason`, `shotsCount`, `payload` (JSON `ExportFile`), `createdAt`, `deletedAt`, `seq: Long?`, `dirty: Boolean` |

- Every local write sets `dirty = true`. In this plan nothing reads `dirty` or `seq`; in Plan 2 the sync
  engine pushes dirty rows, so a user who later logs in uploads their whole heart with no extra code.
- Indices: `shots(deletedAt, undoneAt, shotAt)`, `categories(kind, deletedAt, sort)`.
- Room schema export on; write a real migration from the first released version onward.
- Expose `Flow`s from DAOs; repositories map to domain models.

### Seeding (first launch only, inside one transaction)

Default categories with `presetKey`, Arabic names, a color and an icon each:

| white | black |
|---|---|
| ذكر `dhikr` | غيبة `gheeba` |
| صلاة `salah` | نظر محرّم `forbidden_gaze` |
| صدقة `sadaqah` | كذب `lying` |
| بر الوالدين `birr` | غضب `anger` |
| غض البصر `lowering_gaze` | تضييع صلاة `missed_salah` |
| كظم الغيظ `restraining_anger` | لغو `idle_talk` |

Settings defaults: `dot_size = 14`, `intensity = 80`, `sound_enabled = true`.
**No sample shots** — the heart starts clean.

---

## 4. Export, import, backups

**Export file** (`ExportFile`, identical to the backend contract):

```json
{ "format": "oodan-export", "version": 1, "exported_at": "…Z",
  "shots": [ …all non-deleted shots, snake_case fields… ],
  "categories": [ …all categories incl. deleted ones referenced by shots… ],
  "settings": { "dot_size": 14, "intensity": 80, "sound_enabled": true } }
```

- **Export:** write the file through the platform file-save `expect/actual`.
- **Import → Merge (default):** run the LWW merge for every item; unknown ids are inserted, known ids
  resolved by the rules. Nothing is lost.
- **Import → Replace (explicit, confirmed):** create a backup (`IMPORT_REPLACE`) of the current heart,
  mark every current shot deleted, then merge the file.
- **Restore a backup:** create a backup (`RESTORE`) of the current heart, then Replace with the backup's
  payload (revived shots get `deletedAt = null`, `stateChangedAt = now`).
- **Backups screen:** list (date, reason, shots count), restore, export as file, delete. Kept until the
  user deletes them.
- Validate on import: `format`, `version`, ranges, note length, category kind. Show a clear error for a
  bad file; never half-import (one transaction).

---

## 5. UI

Arabic, RTL forced (`CompositionLocalProvider(LocalLayoutDirection provides Rtl)` at the root).
Fonts Amiri (hadith/title) and IBM Plex Sans Arabic (UI) as Compose resources. Palette and heart
rendering exactly per `HANDOFF.md`.

### Destinations (Navigation Compose)

1. **Heart** (start): hadith, stage, counters, ratio bar, white/black buttons, size + intensity sliders
   with preview, undo/redo buttons, sound toggle, entry points to History and Settings.
2. **History:** newest first, each row: number, color chip, category (icon/color/name), note preview,
   `r`, intensity, time. Tap → jump to that point. Row action → edit note/category. Undone rows dimmed
   and struck through.
3. **Shot details sheet** (bottom sheet): note field (500 max, counter), category picker filtered by the
   shot's color, "new category" inline.
4. **Categories:** two tabs (بيضاء / سوداء), create, rename, set color/icon, reorder, delete (soft).
5. **Backups:** as in §4.
6. **Settings:** export, import (merge / replace), backups, clear heart (two-step confirm, mentions the
   automatic backup), about.

### Throw feedback

- After impact: **Snackbar** «نكتة بيضاء/سوداء» with actions **«أضف ملاحظة»** (opens the details sheet
  for that shot) and **«تراجع»**.
- After undo: **Snackbar** with **«إعادة»**.
- A new snackbar replaces the previous one. Snackbar actions call the same use cases as the buttons.

### Rendering and animation

Exactly per `HANDOFF.md`: baked layers, blend modes in an offscreen `graphicsLayer`, stain bitmap
cache, arrow timings, heartbeat, reduced-motion handling, keyboard shortcuts (`Ctrl+Z`, `Ctrl+Shift+Z`,
`Ctrl+Y`, `W`, `B`). Move the baked layers and sounds into Compose resources so iOS can use them later.

---

## 6. Phases

1. **Module setup:** create the KMP modules and Gradle wiring, Koin, Room KMP, DataStore, Compose
   Multiplatform resources; app launches an empty RTL screen on the emulator.
2. **Domain:** models, UUIDv7, `HeartRules`, LWW merge — with their unit tests.
3. **Local data:** entities, DAOs, repositories, seeding, `dirty` on every write.
4. **Heart screen:** renderer and animations per `HANDOFF.md`, wired to the repository; throw/undo/redo
   with snackbars; sliders persisted as settings. Screenshot vs `reference-composite.png`.
5. **History + shot details + categories.**
6. **Export / import / backups / clear.** Round-trip test: export → import (merge) → identical heart.
7. **Polish:** empty-heart hint for first launch, sound/haptics, keyboard shortcuts, reduced motion.
