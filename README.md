# Satchel

Satchel is a self-hosted, local-first clipping vault. It captures anything
worth keeping — bookmarks, articles, quotes, highlights, places, movies,
books, recipes — and makes it instant to search and retrieve. It is not a
note-taking app.

Each kind of content is handled by a dedicated module that knows how to
capture, enrich, and present it: an article gets a clean reading view and
highlights, a place gets a map, photos and a rating, a movie gets poster
art and a watchlist status.

Privacy comes first. Everything runs on your devices. Sync is optional
and always end-to-end encrypted: self-host it with a single Docker
container, or use the paid Satchel Cloud. Neither can read your vault.

Status: planning. No code yet.

## Planned components

| Component         | Tech                                  |
| ----------------- | ------------------------------------- |
| Core library      | Rust, SQLite, FTS5, nucleo, dom_smoothie |
| Desktop app       | Rust, GPUI + GPUI Kit (gpui-component) |
| Mobile app        | Expo (React Native) + Rust core via UniFFI |
| Sync server       | Rust (axum), one binary / one container; also runs Satchel Cloud |
| Browser clipper   | TypeScript (WXT), Defuddle            |
| AI access         | MCP server (local + remote), optional AI capture (own key or Cloud) |

## Docs

- [Product context](docs/product-context.md) — what Satchel is and isn't
- [Architecture](docs/architecture.md) — components, data, sync, search, MCP
- [Security and privacy](docs/security.md) — threat model, encryption,
  what leaves the device
- [Modules](docs/modules.md) — the content-type plugin system and v1 modules
- [Roadmap](docs/roadmap.md) — phases and open questions

## License

[AGPL-3.0](LICENSE). Self-hosting for yourself carries no obligations;
running a modified version as a service for others requires publishing
your changes.
