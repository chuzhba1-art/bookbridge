# Settings redesign

Settings grew one card at a time for a year. Users said three things confused them: some changes
restart the container and some don't, on/off controls come in several styles, and it is unclear
which settings belong to the server and which to each reader. This plan regroups the page around
what each setting is *for*, and makes every save say up front what it will do.

## Phase 1 — on `feature/settings-redesign`

**Saving no longer always restarts.** Every save used to restart the app. Most settings are read per
call, so they now apply as soon as you save. `RESTART_REQUIRED_SETTINGS`
(`src/utils/config_loader.py`) lists the ones captured at startup:

- the scheduler, the socket listener, logging and Telegram;
- folders;
- values singletons read in `__init__`;
- server connections held by the global clients.

The page marks those fields **restart**, and the Save bar switches to "⟳ Saving restarts
BookBridge" when one of them is unsaved. `tests/test_settings_live_save.py` derives the startup
reads from the source with an AST scan, so a new startup read that is missing from the list fails
the suite.

**Tabs follow intent, not implementation.**

| Tab | Holds |
| --- | --- |
| Services | one card per service (unchanged); the ABS ebook-sync options moved into the Audiobookshelf card |
| Reading & Sync | When BookBridge syncs · How often each app is checked · Finished books · Protecting your place · Reading sessions · Fine tuning (collapsed) |
| Library & Matching | Suggestions · Matching books you read (the KoSync auto-map, text-order and hash-relink options that sat under Suggestions) · Shared library · Series display · Shelfmark |
| Alignment | Forced alignment · Transcription · Safety checks · Experimental · Alignment Health (out of System → Advanced and Sync) |
| AI | unchanged |
| Server | General · Telegram · Reverse proxy · Paths · Background jobs · Diagnostics |
| Maintenance | the one-off clean-up and backfill buttons, which are actions and not settings; no Save bar |

Old `#features` and `#tools` links redirect to `#library` and `#maintenance`.

**One control language.** Every on/off is the same switch. The old checkbox, `toggle-label` and
switch markups render alike through CSS only; field names and posted values are unchanged.

**Per-client polling: 24 fields → 8 rows.** Each app gets one row: *With each sync / On its own
timer*, the seconds, and *wait until you pause*. Rows for apps that are switched off are hidden,
not omitted. This also fixes a bug: their *wait until you pause* box was missing from the form, so
every save reset it to off.

**Find things.** The search box now searches every tab, not only service cards. **Only what I've
changed** shows every card holding a value that differs from its default, and each such field has
a *reset* link.

## Phase 2 — per-user reading preferences (your call)

These describe how *you* read, not how the server runs. Each consumer already runs in a
user-scoped path (KoSync request, per-user sync cycle or client bundle):

| Setting | Read at | Notes |
| --- | --- | --- |
| `KOREADER_ANNOTATION_SYNC`, `KOREADER_STATUS_SYNC_ENABLED`, `KOREADER_SYNC_READ_HISTORY`, `KOREADER_COMBINE_DEVICE_STATS` | `kosync_server.py`, `sync_manager.py` | "your devices" |
| `KOSYNC_FURTHEST_WINS`, `KOSYNC_ACTIVE_DEVICE_WINS` | `kosync_server.py:2097`, `kosync_device_arbiter.py:47` | how your own readers disagree |
| `SYNC_COMPLETION_PROPAGATION`, `SYNC_COMPLETION_THRESHOLD` | `sync_manager.py:392`, `readest_upload_service.py:528` | when *you* count a book finished |
| `ABS_READING_CHECKPOINTS`, `STORYTELLER_LISTENING_SESSIONS`, `GRIMMORY_READING_SESSIONS`, `BOOKORBIT_READING_SESSIONS` | sync clients, `sync_manager.py` | what lands in *your* stats |
| `SERIES_SHOW_CURRENT_BOOK_DETAIL` | template only | display preference |

**Decision needed:** is the admin's value a *default* each reader can override (recommended), or
a value that is *enforced* for everyone?

Mechanism for "default": a new `PER_USER_PREFERENCE_KEYS` set, stored in the user's credential
dict but **not** in `PER_USER_CREDENTIAL_KEYS`, so `resolve_setting` falls back to the global value
for everyone, regular users included. Consumers switch from `env_truthy(K)` to
`user_setting(K)`. On My Integrations the controls must be three-way, *Server default (On) / On /
Off*. A plain checkbox always saves true or false, so a reader could never go back to following
the server default.

Larger, so later: shelf watch per user (it runs as the primary admin today), Telegram per user
(one global log handler), and `ABS_PROGRESS_OFFSET_SECONDS` (captured in `ABSSyncClient.__init__`).

## Phase 3 — combine further

- **Reading history:** one card with a row per app (ABS checkpoints, Storyteller listening, Grimmory
  and BookOrbit sessions) instead of a switch inside each service card.
- **"Up Next" shelf watch:** Grimmory, BookOrbit and Kavita each have the same four fields. Use one
  card with a row per app, like the polling table.
- **Highlights & notes:** the `*_ANNOTATION_SYNC` switches and intervals for KOReader, Grimmory,
  BookFusion, Readest, Hardcover and BookOrbit in one place.
- **Trackers:** the three `*_UPDATE_COOLDOWN_MINS` become one field, unless someone needs them
  to differ.
- **`OLLAMA_*` vs `LLM_*`:** retire the legacy duplicates with a one-time migration.
- **Services cards:** show the admin's own account status inline ("Your account: connected")
  instead of the "set this under Users → (you) → Integrations" boxes.

## Shrinking the restart list

Each of these is a self-contained follow-up:

- `SYNC_PERIOD_MINS`: re-register the `schedule` jobs.
- `LOG_LEVEL`: `_reconfigure_logging()` exists but also resets the Telegram handler's level, so it
  needs a Telegram-safe variant first.
- Sync deltas and the ABS offset: read them per call in the sync clients.
- `EbookParser` and `AlignmentService` options: read per call, or rebuild those singletons.
