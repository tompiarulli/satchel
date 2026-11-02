# Roadmap

Each phase ends with something usable day to day.

## Phase 0 — Core and speed baseline

- Cargo + pnpm workspace, tooling, CI.
- `satchel-core`: schema, migrations, item CRUD, tags, collections.
- Search: nucleo header index, FTS5, query language parser.
- Benchmark with a generated 100k-item vault; set the budgets from
  [architecture](architecture.md#search-and-filtering) as CI checks.
- `satchel-extract` with dom_smoothie + JSON-LD/OpenGraph, and a fixture
  set of real pages.
- Bookmark and article modules.

## Phase 1 — Desktop app, single device

- GPUI app: sidebar, virtualized list, detail pane, command palette,
  quick search and quick capture hotkeys.
- Reading view and highlights for articles.
- Import from Anybox and browser bookmarks, to get a real dataset in.
- Local stdio MCP server against the desktop vault.
- WXT extension with Defuddle, talking to the desktop app over localhost.
- Automatic encrypted backups with verify and restore; Markdown export.
- Block document format for articles, with highlight anchoring.

## Phase 2 — Typed modules and AI capture

- Place (map, near filters), movie/TV (TMDB, watchlist), book, quote,
  image with OCR.
- On-device enrichment jobs, each provider opt-in.
- AI-assisted capture with a local model or bring-your-own key.
- On-device semantic search and related items.

## Phase 3 — Encrypted self-hosted sync

- `satchel-crypto`: key hierarchy, record and blob encryption, recovery
  kit, device enrollment by QR code. Built before any sync code ships,
  so no unencrypted format ever exists.
- `satchel-server`: one binary and one Docker image; opaque record sync,
  encrypted blob store, OPAQUE login, device-signed requests, backups.
- Sync client in the core; desktop and extension sync with the server.
- Remote MCP served by the desktop app, documented setup behind the
  user's own tunnel.

## Phase 4 — Mobile

- Expo app with the Rust core via UniFFI; iOS first.
- Share extension capture (Defuddle preprocessing in Safari), offline
  queue, lazy attachments.
- Map and watchlist views on mobile.

## Phase 5 — Satchel Cloud

- External security and cryptography audit (launch blocker).
- Multi-tenant mode: sign-up, billing, quotas, per-vault SQLite files,
  S3-compatible blob storage.
- Pass-through proxies for AI capture, embeddings and enrichment
  providers, with zero-data-retention providers.
- MCP relay to desktop apps (SNI routing, TLS ends on the desktop).

## Open questions

- **Extraction quality.** How close does the Rust fallback (dom_smoothie)
  get to Defuddle on the fixture set? If the gap is large, port Defuddle's
  site-specific rules into `satchel-extract`, or run Defuddle on mobile in
  a hidden WebView.
- **Licensing.** Privacy claims need auditable code, so clients and
  server should be open source. Which license: AGPL for everything, or
  AGPL server with a permissive core? It affects whether others can
  resell the server as a competing cloud.
- **Remote MCP when the desktop is off.** The relay design needs the
  desktop app running. Is that acceptable, or should a phone be able to
  answer too (harder: iOS limits background work)?
- **Local database encryption.** Rely on OS storage encryption, or add
  optional SQLCipher? Benchmark the search cost first.
- **Crypto library choice.** RustCrypto crates or libsodium bindings;
  decide alongside the audit firm's preferences.
- **Enrichment provider terms.** Check TMDB and other providers' terms
  for app distribution and for proxying through a paid service.
- **Rust ↔ React Native bridge.** Confirm `uniffi-bindgen-react-native`
  works with current Expo and the new architecture before Phase 4; spike
  it early since it is the riskiest dependency.
- **GPUI stability.** GPUI and GPUI Kit move fast and GPUI isn't stable
  on crates.io; pin git revisions and budget for upgrades.
- **Snapshots.** Store a full-page snapshot for every bookmark, or only
  on request? Affects disk use and Cloud storage costs.
- **Android.** Ship alongside iOS from the Expo app, or later?
