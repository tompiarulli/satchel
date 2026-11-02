# Modules

A module defines one kind of clip. It is the reason Satchel exists: since
the app only does clipping, each content type can get a capture and
viewing experience built for it.

## What a module provides

Modules are Rust code in `satchel-modules`, registered with the core at
startup. Each one declares:

| Part | Purpose |
| ---- | ------- |
| `id`, `version`, name, icon | e.g. `place`, v1 |
| Field schema | Typed fields stored in `items.fields`, with flags for filterable / sortable / searchable. Versioned, with migrations between versions |
| Matchers | Rules that propose this module for a capture: URL patterns (`imdb.com/title/*`, `maps.google.com/*`), JSON-LD `@type`, OpenGraph type, MIME type |
| Extractor | Turns a captured page (HTML, JSON-LD, Defuddle output) or shared payload into field values |
| Enrichers | Jobs that fill in more data from outside sources, run on the capturing device |
| AI prompts | Optional prompt templates for AI-assisted capture: how to fill this module's fields from free text |
| Search projection | Which text goes into the full-text index and which fields become filter keys in the query language |
| View hints | A declarative description of list rows, cards and the detail view (field order, layout blocks such as `map`, `gallery`, `poster`, `reader`, `rating`) |
| MCP description | How the type and its fields are explained to AI clients |

### Rendering on three platforms

The desktop app is GPUI, the mobile app is React Native, and both need to
render every module. To avoid writing every module twice, views are mostly
declarative: the module lists layout blocks and fields, and each client
has one renderer per block type (`map`, `gallery`, `reader`, `rating`,
`status`, `facts`, …). A module that needs something special adds a new
block type, implemented once per client.

The extractor in the browser extension is TypeScript, since it runs in the
page. It produces a generic capture payload (URL, Defuddle result,
JSON-LD, selection, screenshots), and the module's Rust extractor in the
core maps that payload to fields on the receiving device. That keeps the
per-module logic in Rust.

### Third-party modules (later)

v1 modules are compiled in. Later options: modules described as data
(schema + matchers + views) with enrichers as WASM plugins (e.g. Extism),
so users can add types without rebuilding the app.

## Shared item features

Every module gets these for free: title, URL, tags, collections, rating,
favorite, status, comment, highlights, attachments, capture source and
date, full-text search, export.

## v1 modules

### Bookmark

The fallback for any URL. Title, description, favicon, preview image,
domain. Optional snapshot of the page.

### Article

- Clean reading view from Defuddle (extension, Safari share) or the Rust
  extractor (URL-only captures), stored as a block document (see
  [architecture](architecture.md#article-bodies)).
- Highlights with notes, kept in sync with position anchors in the text.
- Fields: author, site, published date, word count, reading time, reading
  progress, read status (unread / reading / read).
- Reader settings: font, width, theme.

### Quote

A standalone piece of text: the quote, who said it, source (book, URL,
talk), optional link to another item. Captured from a text selection or
typed in.

### Place

- Fields: name, coordinates, address, category, website, phone, opening
  hours, price level, own rating, visit status (want to go / been),
  visit dates, photos.
- Map block (MapLibre, OSM tiles), "near me" and `near:` filters,
  map view of all places in a collection or search.
- Capture from Google Maps / Apple Maps share links, OSM, or a typed
  address. Enrichment via Nominatim and Overpass.

### Movie / TV

- Fields: title, year, kind (movie / series), poster, backdrop, runtime,
  genres, director, cast, TMDB/IMDb ids, watch status (want / watching /
  watched), watched dates, own rating, where to stream (optional).
- Watchlist view sorted and filtered by status, genre, year.
- Capture from IMDb, TMDB, Letterboxd or a title search. Enrichment via
  TMDB.

### Book

- Fields: title, author, cover, ISBN, publisher, year, pages, read status,
  own rating.
- Capture from Goodreads, Amazon, Open Library, ISBN scan on mobile.
  Enrichment via Open Library.

### Image

Screenshots and saved images, with OCR text in the full-text index
(platform OCR on device: Apple Vision on macOS/iOS, ML Kit on Android).

### Later candidates

Recipe (schema.org Recipe: ingredients, steps, servings), product (price,
store, wishlist), video (YouTube with transcript), podcast episode,
music album, GitHub repository, code snippet, PDF/document.
