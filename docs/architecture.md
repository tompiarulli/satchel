# Architecture

## Overview

```text
 ┌─────────────────────┐   ┌─────────────────────┐   ┌──────────────────┐
 │ desktop (GPUI)      │   │ mobile (Expo)       │   │ browser extension│
 │ satchel-core        │   │ satchel-core (FFI)  │   │ (WXT, Defuddle)  │
 │ local SQLite vault  │   │ local SQLite vault  │   │ sends captures to│
 │ local stdio MCP     │   │ share extension     │   │ desktop or sync  │
 └──────────▲──────────┘   └──────────▲──────────┘   └────────┬─────────┘
            │        optional sync    │                       │
            └───────────┬─────────────┘◀──────────────────────┘
          ┌─────────────┴──────────────────────────────────────┐
          │ satchel-server — one binary / one container         │
          │ stores + relays end-to-end encrypted records/blobs  │
          │ MCP relay · SQLite + files, no other services       │
          │  ├─ self-hosted: homelab, single user/household     │
          │  └─ Satchel Cloud: same server, multi-tenant, paid  │
          └─────────────────────────────────────────────────────┘
```

Everything that makes Satchel useful — capture, extraction, enrichment,
search — runs on the device, like the Obsidian Web Clipper. The apps work
with no server at all. The server only stores and relays end-to-end
encrypted sync data; it can never read a vault. See
[security](security.md) for the full privacy design.

That gives two ways to sync, with identical client behaviour:

- **Self-hosted:** one Docker container (or a single binary) with a data
  directory. No database server, no crawler, no queue service.
- **Satchel Cloud:** the same server run as a paid hosted service, for
  people who don't run a homelab.

One Rust core library holds the data model, storage, search, extraction,
enrichment, module definitions and the sync client. The desktop app, the
mobile app and the server all link it, so the rules for what an item is
and how it is searched exist in exactly one place.

## Repository layout (planned)

```text
crates/
  satchel-core/      data model, SQLite storage, search, module registry
  satchel-extract/   HTML → article/metadata extraction (dom_smoothie,
                     JSON-LD, OpenGraph), capture payload types
  satchel-modules/   first-party modules (article, place, movie, …)
  satchel-crypto/    key hierarchy, record/blob encryption, enrollment
  satchel-sync/      sync protocol types + client; HLC clock
  satchel-server/    axum server: encrypted sync, blobs, accounts, MCP relay
  satchel-desktop/   GPUI app
  satchel-ffi/       UniFFI bindings for the mobile app
  satchel-mcp/       MCP tool definitions shared by server and local stdio
apps/
  mobile/            Expo app (iOS first, Android later)
  extension/         WXT browser extension
packages/
  types/             TypeScript types generated from Rust (ts-rs or specta)
docker/              Dockerfile + compose example for self-hosting
docs/
```

Cargo workspace at the root, pnpm workspace for `apps/*` and `packages/*`,
matching the tooling of the other personal projects (pnpm, oxlint/oxfmt
for TS; `cargo fmt`, `clippy -D warnings` for Rust).

## Storage format: a database, not files

Satchel is a vault, not a notebook, so it stores records in SQLite rather
than a folder of Markdown files. Obsidian's file-per-note model has three
costs Satchel avoids:

| Markdown files (Obsidian) | SQLite vault (Satchel) |
| ------------------------- | ---------------------- |
| The app reads and indexes files on startup, and re-checks them when it opens | Indexes are persistent; opening the app is a database open plus one query |
| Sync moves whole files and has to work out which files changed, so a phone shows a progress bar when it wakes up | Sync moves small field-level changes since a cursor, typically a few KB, in the background |
| Structure is squeezed into front matter and text conventions: no typed fields, no reliable anchors for highlights, no relations | Typed fields with indexes, relations between items, stable IDs for highlights |
| Readable without the app | Readable through the Markdown export, which is a way out, not the storage |

What stays: the vault is still one file per device, so backup is simple
(see [Storage, backup and export](#storage-backup-and-export)), and the
Markdown export keeps the data portable.

### Article bodies

Article text is stored as a **block document**: a versioned list of typed
blocks (heading, paragraph, list, quote, code, image, figure with
caption, table, embed placeholder) with inline marks (bold, italic, link,
code). Every block has a stable ID.

- Both native UIs render blocks directly. GPUI has no HTML engine, and a
  block list maps cleanly to GPUI elements and React Native views alike.
- Highlights anchor to `(block ID, start offset, end offset)` plus the
  quoted text, so they survive re-rendering, and can be re-anchored by
  fuzzy text match if an article is re-extracted.
- Nothing to sanitize at render time: there is no HTML to execute.
- Defuddle's HTML and dom_smoothie's output are converted to blocks in
  `satchel-extract`. Markdown is generated from blocks for export and for
  AI tools.
- Stored as compact binary (e.g. CBOR or postcard) in `item_text`, with
  a plain-text projection for FTS.

## Data model

All records are rows in SQLite. Core tables:

| Table        | Purpose |
| ------------ | ------- |
| `items`      | One row per clip: `id` (UUIDv7), `type` (module id), `title`, `url`, `canonical_url`, `source`, `created_at`, `updated_at`, `captured_at`, `status` (e.g. inbox / archived), `rating`, `favorite`, `comment`, `fields` (JSON, module-specific), `deleted_at` |
| `item_text`  | Large bodies kept out of `items` so list queries stay small: article block document, transcript, OCR text, plus a plain-text projection for FTS |
| `highlights` | Quotes/highlights belonging to an item: quoted text, block anchor, note, color |
| `tags`, `item_tags` | Flat tags with optional nesting by path (`travel/japan`) |
| `collections`, `collection_items` | Manual collections plus saved searches (a stored query) |
| `attachments`| Content-addressed blobs: `hash` (BLAKE3), mime, size, role (cover, photo, snapshot, favicon) |
| `jobs`       | Pending local work: extraction or enrichment that waits for network |
| `changes`    | Local outbound change log for sync |

Module fields live in the `fields` JSON column, validated by the module's
schema. Fields the module marks as filterable or sortable get generated
columns plus indexes (e.g. `place.lat`, `movie.release_year`,
`movie.watch_status`), so filters are index lookups rather than JSON scans.

A standalone quote (not from a saved page) is an item of type `quote`.
A highlight made while reading an article is a `highlights` row on that
article. Search returns both.

## Search and filtering

Speed is mostly a data-layer property, so every query stays local and
in-process.

- **Instant fuzzy match** (titles, URLs, domains, tags): an in-memory
  index of lightweight item headers matched with `nucleo` (the fuzzy
  matcher from Helix). Loaded from a compact header table after the first
  screen is drawn, updated incrementally. Until it is ready (well under a
  second for 100k items), search falls back to FTS, so nothing waits.
- **Full text** (article bodies, highlights, comments, transcripts):
  SQLite FTS5 with the `unicode61` tokenizer for word search and a
  trigram table for substring search, ranked with bm25.
- **Structured filters**: a small query language shared by every client
  and the MCP server, e.g.
  `type:place tag:japan rating:>=4 status:want near:"Kyoto"` or
  `type:movie watch:want year:1990..1999`. Parsed in the core, compiled to
  SQL against indexed columns.
- **Semantic search** (optional): embeddings computed on the device with
  a small in-process ONNX model (e.g. via `fastembed`) and stored locally
  with `sqlite-vec`. They sync inside encrypted records, so each device
  computes an item once and the others reuse it. A provider key or
  Satchel Cloud can compute them instead, as an opt-in. Used by the MCP
  `search` tool and a "related items" panel.

Encryption applies to sync and the server, not to local queries, so it
costs search nothing.

Budget: fuzzy + filter results for 100k items within one frame on
desktop; FTS within ~30 ms. A benchmark suite with a generated 100k-item
vault guards this from the start.

## Capture and extraction (on device)

Extraction happens wherever the capture happens. The best results come
from a live, rendered DOM, so use one whenever it exists.

| Capture path | Extraction |
| ------------ | ---------- |
| Browser extension | Defuddle runs in the page on the rendered DOM, plus JSON-LD / OpenGraph. Same approach as the Obsidian Web Clipper |
| iOS share from Safari | The share extension's JavaScript preprocessing step runs Defuddle on the live page before handing it to the app |
| Share from other apps, paste, hotkey, MCP `clip_url` (URL only) | The core fetches the HTML and runs `satchel-extract`: `dom_smoothie` (a Rust port of Readability.js) for the article, plus JSON-LD / OpenGraph for metadata |
| Offline capture | Saved immediately as a bookmark; extraction is queued in `jobs` and runs when the device is online |

Both paths produce the same capture payload (URL, title, Markdown body,
metadata, JSON-LD, selection, images), and the module's Rust extractor
maps it to fields. Defuddle-based extraction and the Rust fallback should
be compared on a fixture set of real pages; if the gap is large, port
Defuddle's site-specific rules into `satchel-extract`.

The extension sends captures to the desktop app over localhost when it is
running (works fully offline), otherwise to the sync server, otherwise
holds them in extension storage until one is reachable.

## Enrichment (on device)

Module enrichers (TMDB for movies/TV, Nominatim for places, Open Library
for books, favicons and cover images) live in the core and run on the
device that made the capture, right after saving or later from the `jobs`
queue. Results are ordinary edits, so they sync like any other change.

To avoid two devices enriching the same item, an enrichment job is tied to
the device that captured the item; others only pick it up if it is still
pending after a timeout.

Some providers need API keys or limit request rates:

- Self-hosters can enter their own keys (e.g. TMDB) in settings; the
  apps work without them, just with less enrichment.
- Satchel Cloud proxies these providers for subscribers, so keys never
  ship inside the apps (the same pattern as AI-assisted capture below).
  The proxy sees only the lookup term, stores nothing, and hides the
  user's IP from the provider.
- Every provider is opt-in and can be turned off.
- Providers with usage policies (Nominatim: about 1 request/s, identifying
  user agent) are rate-limited in the core. Platform services such as
  Apple MapKit on iOS can be used where they are free.

## AI-assisted capture (optional)

Like the Obsidian Web Clipper's Interpreter, capture can call a language
model to fill in what rules-based extraction can't. Uses:

- Pick the module for an ambiguous page (is this a place, a recipe, an
  article?).
- Fill module fields from free text, e.g. a restaurant name, address and
  price level from a blog post, or a movie title from a tweet.
- Suggest tags and collections, write a short summary.
- Split one page into several items (a "10 best cafés in Lisbon" article
  becomes 10 places linked to the article).
- Per-module prompt templates, editable by the user.

Off by default. The model runs from the device through one provider
interface in the core:

- **Local model** (Ollama, LM Studio): recommended; nothing leaves the
  device.
- **Bring your own key** for technical users: Anthropic, OpenAI, or any
  OpenAI-compatible endpoint. Requests go straight from the device to the
  provider.
- **Satchel Cloud** for everyone else: subscribers get AI capture through
  a pass-through proxy that stores and logs nothing, using providers under
  zero-data-retention terms. Clearly labelled as leaving the device.

The same choice applies to embeddings for semantic search: a local
in-process model, a provider key, or Cloud.

AI is never on the capture path. The item is saved first; AI results
arrive as edits a moment later, and the user can turn suggestions into
automatic edits or keep them as proposals to accept.

## UI

### Desktop — GPUI + GPUI Kit

- `gpui-component` for virtualized lists, tables, inputs, docks, theming.
- Layout: sidebar (types, collections, tags, saved searches) → virtualized
  result list/grid → detail pane rendered by the item's module.
- Keyboard-first: command palette, everything reachable without a mouse.
- Global hotkey quick-capture panel (paste URL/text, pick type, save) and
  a Raycast-style quick-search panel.
- A localhost endpoint for the browser extension.
- Database work runs on a background executor; the render thread only
  reads in-memory state.
- Targets macOS first; Linux and Windows come along as GPUI supports them.

### Mobile — Expo + Rust core

- Expo dev builds (no Expo Go) with the Rust core exposed through UniFFI,
  generated with `uniffi-bindgen-react-native`.
- iOS share extension for capture from any app (candidate: an Expo share
  extension config plugin), with JavaScript preprocessing for Safari.
  Captures are written straight to the local vault, then synced.
- Lists use a virtualized list (FlashList or Legend List); search runs in
  Rust, the JS side only renders results.
- Attachments download lazily with a configurable on-device cache size;
  metadata and text are always fully synced.
- Maps for the place module via MapLibre with OSM-based tiles.

### Browser extension — WXT

- Captures the current page, a selection (as a quote/highlight), or a link.
- Picks a module from URL matchers and page metadata; the user can
  override before saving.

## Sync

Optional. Clients hold full replicas; the server holds the reference copy
that all devices converge on. The data is mostly append-heavy with rare
concurrent edits to the same field, so a simple scheme is enough; no CRDT
library.

- Every write produces a change: `(entity, id, field, value, hlc)`.
  Hybrid logical clocks order changes across devices.
- Changes are encrypted on the device as whole tuples. The server sees an
  opaque record with a random ID and assigns it a sequence number; it
  never needs to read contents, because merging happens on devices.
- Conflict rule: last writer wins per field, so editing a movie's rating
  on the phone and its tags on the desktop both survive. Highlights and
  tags are separate rows, so they merge naturally.
- Deletes are tombstones (`deleted_at`), purged after a retention window.
- Protocol: `POST /sync/push` with pending local changes, `GET
  /sync/pull?since=<cursor>` for server changes, cursor = server sequence
  number. Same protocol over a WebSocket later for live updates.
- Blobs are encrypted and uploaded/downloaded separately, addressed by an
  HMAC of their contents with a vault secret, so the same image is stored
  once per vault and the server can't match files across users.
- The protocol is the same for self-hosted and Cloud; switching between
  them is a matter of pointing the app at another URL and pushing.

### The app never waits for sync

Opening the app must look the same online, offline, or after a month
away: the local vault is shown immediately, and sync catches up behind it.

- **Open is local.** The first screen comes from the local database. Sync
  starts after first paint and never blocks input, search or capture.
- **Small deltas.** A normal catch-up pulls only changes since the
  device's cursor: field edits, not whole items or files.
- **Fresh before you open it.** The server sends a content-free "vault
  changed" push (APNs/FCM) so iOS/Android can sync in the background, and
  the mobile app also uses background refresh. Opening the app usually
  finds nothing to pull. The push is opt-in, since it reveals timing to
  Apple/Google.
- **Priority order.** Pull item headers first, then fields and highlights,
  then text bodies, then blobs. New items show up in lists while the rest
  streams in.
- **Lazy blobs.** Images and snapshots download when viewed or when the
  device is idle on Wi-Fi, within a cache limit.
- **Quiet status.** A small sync indicator, never a progress bar or a
  modal. Errors appear only when they need the user's action.

### Snapshots and log compaction

The server can't read or compact encrypted changes, so devices do it:

- A device periodically uploads an **encrypted vault snapshot** (a
  compressed export of the current state, no history) tagged with the
  cursor it covers. The server can then drop changes older than the
  snapshot once every device has pulled them.
- A **new device** downloads the latest snapshot in one stream and then
  pulls only the changes after it, instead of replaying years of history.
  Headers come first, so the vault is usable within seconds even for a
  large vault.
- A device that has been offline longer than the retention window
  re-bootstraps from the snapshot and pushes its own pending changes on
  top; those merge like any other changes.

## Server

One Rust binary, also shipped as one Docker image.

- **Storage:** one SQLite file per vault (WAL mode) holding opaque
  encrypted records, plus an encrypted blob directory. Per-vault files
  keep tenants isolated in Cloud and make backup and deletion of one
  account trivial. Cloud can store blobs in S3-compatible storage behind
  the same interface.
- **Accounts:** OPAQUE login, so the server never learns passwords; device
  keys sign requests. Self-hosted defaults to one admin account; Cloud
  adds sign-up, billing and quotas. Same code, a config switch decides
  which features are on.
- **No background services and no content processing.** The server
  stores, orders and relays. It runs no extraction, enrichment or
  embeddings because it can't read anything.
- **MCP relay:** forwards remote MCP connections to the user's desktop
  app without decrypting them (see below).

## AI access — MCP

Shared tool definitions in `satchel-mcp`, served two ways:

- **Local** (desktop): a stdio MCP server reading the local vault, for
  Claude Desktop, Claude Code and other local clients. Works offline and
  with no server. Read-only by default.
- **Remote** (desktop, via relay): MCP over Streamable HTTP with OAuth
  2.1, which Claude and ChatGPT custom connectors require. The desktop app
  serves it; the sync server only relays encrypted bytes to it, routing
  by TLS SNI so TLS ends on the desktop. Works while the desktop app is
  running. Self-hosters can use their own tunnel instead (Tailscale
  Funnel, Cloudflare Tunnel).

Using either sends whatever the AI reads to that AI provider; the app
says so when a client is connected.

Initial tools:

| Tool | Description |
| ---- | ----------- |
| `search` | Query language + free text + optional semantic mode; returns compact item summaries |
| `get_item` | Full item: fields, highlights, tags, collections |
| `get_item_text` | Extracted body text, paginated |
| `list_types` | Available modules and their filterable fields |
| `list_tags`, `list_collections` | Vault structure |
| `clip_url` | Save a URL; extracted and enriched by the core |
| `create_item` | Save a typed item from structured data (e.g. a place the AI found) |
| `update_item` | Change status, rating, tags, collections, comment |
| `add_highlight` | Attach a quote to an item |

Tool results return small summaries by default and full content only on
request, to keep model context cheap. Write tools can be disabled per
connection.

## Storage, backup and export

- Devices: one SQLite vault in the app-data directory, plus a blob cache.
- Server: `vaults/<id>.sqlite3` + `blobs/`, all ciphertext. Backups are a
  nightly `VACUUM INTO` snapshot plus the blobs directory; Litestream is
  an option for continuous replication. Never copy the live database and
  WAL files separately. Because everything is encrypted, backups can go
  anywhere.
- **Automatic backups on the desktop, on by default once a folder is
  picked** (local disk, NAS share, iCloud Drive, Dropbox — any folder):
  - A consistent `VACUUM INTO` copy of the vault, encrypted, written as one
    file per backup with a retention schedule (e.g. 7 daily, 4 weekly,
    12 monthly).
  - Blobs are copied once into a shared encrypted `blobs/` folder next to
    the backups. They are addressed by hash, so each backup only adds new
    files and a large photo library isn't copied again every night.
  - Each backup is verified after writing (opened, integrity-checked,
    item count compared) and the result shown in settings.
  - Restore is a button in the app: pick a backup, preview its date and
    item count, restore into a new vault or replace the current one.
- The sync server is a second copy, not the only backup: with E2EE, a
  forgotten password makes server data unrecoverable, so local backups
  still matter.
- Export: a full vault as Markdown files with YAML front matter (one file
  per item, grouped by type), highlights inline, attachments in an
  `assets/` folder. Readable in any editor and importable into Obsidian.
  Runs on the desktop app only (the server can't read the vault), as an
  explicit choice with a warning that the output is unencrypted.
- Import: Anybox, Obsidian Web Clipper output, browser bookmarks HTML,
  Pocket/Omnivore/Readwise exports, Letterboxd CSV.

## Security

See [security](security.md): threat model, key hierarchy, what the server
can see, device storage, extension and localhost hardening, AI and MCP
privacy.
