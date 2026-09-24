# Flashcard App — Architecture Brief

Decisions from the design phase, written as the starting brief for implementation. Companion files, which should sit next to this one in `docs/`:
- `schema.sql`: the deck file schema (v1)
- `events.md`: the event catalog and apply rules

## Goals

1. **Offline-first multi-client sync.** Every client works fully offline and syncs when it reconnects.
2. **Longevity.** The app should be usable in 50 years. The **data format** is the durable artifact; code will be rewritten.
3. **User sovereignty.** A user's data is a plain SQLite file they can back up and read with any SQLite tool, without this app.

The flashcard domain is somewhat incidental. The sync architecture is the core.

## System shape

- **One SQLite database per deck, everywhere.** Clients and server use the same schema and the same core code.
- **Per-deck server process.** The server runs one lightweight, single-writer process per active deck, each owning one SQLite file. There is no shared multi-tenant cards table.
- **The server is the sequencer, not just a peer.** For each incoming event it:
  1. validates it
  2. assigns a monotonic `seq`
  3. persists it
  4. fans it out to connected clients
- **Clients only push their own local events.** Everything else arrives by pull.
- **Control plane: a Rails app.** It handles:
  - registration, accounts, and auth
  - deck creation and ACLs
  - routing: it tells a client which deck process to connect to

  After authentication, Rails mints a **short-lived signed token** scoped to `{user, deck, permissions}` plus the deck's endpoint. The deck process verifies the token statelessly and never calls Rails on the hot path. Deck creation is idempotent: the SQLite file is created on first open.
- **Cross-deck queries** such as "due today across all decks" fan out on the **client**, across local files, which is fast. Server-side aggregation is only needed later for push notifications or email; a small per-deck summary can serve that.

## Sync protocol

- **Client-generated IDs** (UUIDv7), so offline creates never need the server.
- **Optimistic local apply.** Every local mutation is applied to local SQLite immediately and appended to `_events` with `seq = NULL`. Those rows are the outbox.
- **Push.** Outbox events go to the server in order. The server dedupes on the event `id`, so retries are safe.
- **Pull.** The client fetches events with `seq > cursor`, applies them, and advances the cursor. The live WebSocket/SSE push is only an optimization; pull-by-cursor is the protocol, because offline clients miss broadcasts.
- **No rollback needed.** LWW plus delete-wins make property edits commutative (see Conflicts), so pulled events are applied on top of pending local ones.
- **Bootstrap.** A new device takes a snapshot plus the current `seq`, not the full history.
- **Tombstones.** Deletes are tombstones. The log has a prune horizon; a client whose cursor is older than the horizon does a full resync.
- **Endpoints.** Conceptually `POST /push` and `GET /pull?cursor=`, plus a live channel.

## Conflicts and clocks

- **Hybrid logical clocks (HLC)** on every client and on the server.
  - Local edit: `wall = max(wall, now)`; bump the counter if the wall didn't advance.
  - Receive: jump strictly past the larger of the local and remote stamps.
  - Encoded as sortable text: `<wall_ms:13>-<counter:5>-<client_id>`.
  - The server rejects HLCs more than ~5 min in the future.
- **Field-level LWW.** Each mutable property has its HLC in `_clocks`, and a write applies only if its HLC is greater.
- **Delete wins.** Any event targeting a tombstoned entity is a no-op.
- **Reviews are append-only** and never conflict. Two devices reviewing the same card offline both count.
- **Scheduling state is derived** by deterministically replaying a card's reviews in `(reviewed_at, id)` order. `card_state` is a rebuildable cache.
- **Apply must be deterministic:** no clock reads, randomness, or ID generation inside apply.

## Data model

Full details are in `schema.sql` and `events.md`. Summary:

- **Deck:** the file itself, with a single `deck` row.
- **FactType:** a user-defined kind of fact ("Country", "Vocab"). It is **per deck**; there are no cross-deck shared types, since sharing adds no capability and would break deck self-containment. A starter library that is copied into decks is a possible future UI feature.
- **Field:** a user-defined column on a FactType. The data types are text, rich_text, number, image, and audio.
- **Real tables per FactType.** Chosen for sovereignty: a Country type is a real table, `fact_country`, with real columns.
  - Schema changes (DDL) are events that every replica applies in `seq` order.
  - Events reference `field_id`, never column names. The `fields` table maps `field_id` to the current `column_name`, so renames are safe.
  - User tables hold live data only; tombstones live in the `facts` registry. Sync bookkeeping lives in `_`-prefixed tables.
- **Template:** belongs to a FactType, with mustache-style `front` and `back`. Its `generation` is either `one` (one card per fact) or `per_cloze` (one card per `{{cN::…}}` in the fact).
- **Fact:** one instance of a FactType. The name "Fact" was chosen deliberately instead of Anki's "Note".
- **Card:** **derived, never created by events.** Its identity is `(fact_id, template_id, ordinal)`, and its ID is a UUIDv5 of that triple, so every replica generates identical cards. A cloze that is removed and later re-added regains its review history.
- **Review:** an append-only log entry, with kind `review` or `reset`. Undo works by voiding a review.
- **Media:** content-addressed (sha256) blobs stored **inside** the deck file. The event carries only the hash; bytes sync out of band.
- **Readability conventions:**
  - UUIDs, ISO 8601 timestamps, and JSON event payloads
  - plain values (text or Markdown, not proprietary rich-text formats)
  - an `_about` table that self-describes the format
  - comments in the DDL, which SQLite preserves in `sqlite_schema`

## Technology

- **Core in Rust.**
  - A synchronous library of functions over SQLite. **No async in the core**, since async is the main source of ecosystem churn.
  - Minimal dependencies: `rusqlite` (bundled SQLite), `serde`/`serde_json`, `uuid`.
  - Events are a `serde` enum. Malformed input becomes an `Err`, never a panic.
- **Bindings in separate thin crates,** so their churn never touches core logic:
  - UniFFI for Swift and Kotlin
  - wasm-bindgen for the browser
- **Native UIs:** Swift (iOS), Kotlin (Android), TypeScript (web).
- **Browser:** the core and SQLite run as WASM in a Web Worker, using the official sqlite-wasm build with OPFS persistence. **This is the riskiest target; prototype it early.**
- **Server:** runs the same core natively.
- **Wire format:** JSON via serde for now. Protobuf is an option later if its formal schema-evolution rules become worth it. Events are stored as JSON in SQLite regardless, for readability.
- **Longevity hygiene:**
  - commit `Cargo.lock`
  - pin the toolchain in `rust-toolchain.toml`
  - `cargo vendor` dependencies
  - audit transitive dependencies with `cargo tree`

## Undecided

- **Hosting and placement for deck processes.** The requirements:
  - exactly one live process per deck cluster-wide (a directory plus leases)
  - spin-up on demand and hibernation when idle
  - durability across node loss (Litestream, LiteFS, or object storage)

  This is the hardest infrastructure piece. Cloudflare Durable Objects fit the model, but they expose SQLite through an API rather than a file, which conflicts with the file-based core and with sovereignty. The current lean is real files on Fly Machines or VMs.
- **Identifier collisions.** Two offline clients may both create a "Country" type, both claiming `fact_country`. Proposal: the server rejects the later event, and the client resubmits with a suffix. The same applies to column names.
- **Log compaction:** snapshot and prune policy.
- **Field type conversion rules.**
- **Scheduler.** FSRS is likely; evaluate existing Rust implementations. Its version is pinned in `card_state.scheduler`.
- **Shared or collaborative decks.** `user_id` is already on `reviews` and `card_state` to allow for them.
- **Media byte transport.**

## Suggested first steps

1. **Cargo workspace:** `core` (pure, synchronous), with `core-ffi`, `core-wasm`, and `server` stubbed for later.
2. **HLC module,** with unit tests for skew, the receive rule, and string ordering.
3. **Schema bootstrap** from `schema.sql`, with migrations keyed on `PRAGMA user_version`.
4. **Event enum and a deterministic `apply`** for every event in `events.md`, including the DDL events.
5. **Card generation,** including cloze parsing and deterministic UUIDv5 IDs.
6. **Review replay** into `card_state`, using a simple placeholder scheduler until FSRS is chosen.
7. **Convergence test harness.** Simulate a server and N clients in memory with random offline periods and event interleavings. Assert that every replica ends byte-identical in its user tables. Property-based testing (e.g. `proptest`) is a good fit. **This test is the core's main correctness guarantee.**
8. Then the server's push/pull endpoints, the Rails control plane, and the first UI.
