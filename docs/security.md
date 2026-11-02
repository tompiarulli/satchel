# Security and Privacy

Privacy is Satchel's first priority. When it conflicts with convenience,
privacy wins and the convenient option becomes an explicit opt-in.

## Threat model

Protect the vault from:

- **The sync server and its operator** — including Satchel Cloud and us.
  A full server breach must reveal nothing about vault contents.
- **Network attackers** between devices and the server.
- **Web pages** the user visits or clips: malicious HTML, tracking pixels,
  pages trying to reach the desktop app's localhost endpoint.
- **Other apps and extensions** on the same machine, as far as the OS
  allows.
- **Lost or stolen devices**, to the extent of OS storage encryption plus
  an app lock.

Out of scope, stated plainly to users:

- A compromised device with the vault unlocked.
- AI providers the user chooses to send data to. Once Claude or ChatGPT
  reads items through MCP, or a model fills in fields during capture,
  that provider has that data. Encryption protects against the server,
  not against the AI the user invites in.
- Websites seeing the user's IP when a device fetches a page they clipped
  (the same as visiting it).

## Core rules

1. **The server never reads vault contents.** Every vault is end-to-end
   encrypted, on Cloud and self-hosted alike. There is no unencrypted
   mode, so there is no server-side code path that could leak.
2. **All processing happens on devices.** Search, semantic search,
   extraction, enrichment and MCP run on devices against their local
   copy.
3. **Nothing leaves a device without the user turning it on.** No
   telemetry. Crash reports are opt-in and scrubbed. AI and enrichment
   providers are opt-in per provider.
4. **No custom cryptography.** Use well-reviewed primitives and
   libraries; get an external audit before Cloud launches.
5. **Verifiable.** Clients and server are open source so the privacy
   claims can be checked (license still to be decided; see roadmap).

## Encryption design

### Keys

```text
password ──Argon2id──▶ password key ─┐
recovery key (random, printed) ──────┼─▶ unwraps ──▶ vault key (random, 256-bit)
device key (X25519, per device) ─────┘                  │
                                                        ├─▶ record keys (derived)
                                                        ├─▶ blob keys (derived)
                                                        └─▶ blob-ID key (HMAC)
```

- The **vault key** is random and never leaves devices unwrapped. It is
  stored wrapped by the password key, by the recovery key, and by each
  enrolled device's public key.
- The **password** is never sent to the server. Account login uses
  OPAQUE (an asymmetric password-authenticated key exchange, e.g. the
  `opaque-ke` crate), so the server can't learn the password or brute-force
  it offline from what it stores.
- **Device enrollment:** a new device shows a QR code; an existing device
  scans it and wraps the vault key for the new device's public key. A
  password + recovery key also works when no other device is available.
- **Recovery kit:** a printable sheet with the recovery key, generated at
  setup, like 1Password's Emergency Kit. Losing the password and the
  recovery key means losing the data; Cloud can't reset it, and the app
  says so up front.
- **Device revocation:** removing a device rotates the vault key. New
  data uses the new key; old records are re-encrypted in the background.
- **Password change** re-wraps the vault key and touches no data.

### Primitives

- XChaCha20-Poly1305 for records and blobs (random 192-bit nonces, no
  nonce bookkeeping across devices).
- Argon2id for password stretching, with parameters tuned per platform.
- X25519 for device key wrapping, Ed25519 for device request signing.
- HKDF-SHA256 for deriving sub-keys; HMAC-SHA256 for blob IDs.
- Libraries: RustCrypto crates or libsodium bindings, pinned and audited
  by `cargo-deny` / `cargo-audit`.

### What the server can see

Sync records are opaque. The server stores, per vault:

| Server sees | Server does not see |
| ----------- | ------------------- |
| Random record IDs, sequence numbers, receive times | Item types, field names, values, titles, URLs, tags |
| Ciphertext sizes, padded into size buckets | Exact sizes |
| Blob IDs (HMAC of contents with a vault secret) | Blob contents, or whether two users stored the same file |
| Account email, billing details (Cloud only) | Password, vault key |
| Which device pushed which record | — |

Remaining leaks are timing and volume: the server can tell when and how
much a user clips. Batching and padding reduce this but can't remove it.

Change records encrypt the whole `(entity, field, value, hlc)` tuple, not
just the value, so field names don't leak which kinds of edits happen.
Merging happens on devices, so the server only orders opaque records.

## On devices

- **Storage:** the local SQLite vault is unencrypted inside the app's
  sandbox, protected by OS storage encryption (FileVault, iOS Data
  Protection class "complete until first unlock" or stricter, Android
  file-based encryption). This keeps search fast. Keys live in the OS
  keychain/keystore.
- **App lock:** optional biometric or passcode lock on desktop and mobile.
  Optional SQLCipher for the local database is a candidate for users who
  want encryption independent of the OS; benchmark its cost on search
  first.
- **Reader view loads nothing remote.** Images in clipped pages are
  downloaded at capture time and stored as blobs, so reading an article
  never contacts tracking servers. Scripts and iframes are never kept.
  Content is stored as a typed block document, never as HTML, so there
  is nothing to execute at render time.
- **URL cleanup:** tracking parameters (`utm_*`, `fbclid`, `gclid`, …) are
  stripped from clipped URLs.
- **Backups and exports** written by the desktop app are encrypted by
  default (with the vault key or a separate export passphrase); a plain
  Markdown export is an explicit choice with a warning.

## Browser extension

- Minimal permissions: `activeTab` plus a user gesture to clip, instead of
  access to all sites. Optional per-site permissions only for features
  that need them.
- Captures go to the desktop app over localhost, or are encrypted in the
  extension before going to the sync server. The extension is enrolled
  like a device.
- **Desktop localhost endpoint:** binds to `127.0.0.1` only, requires a
  pairing token exchanged when the extension is set up, checks the
  extension's origin, and rejects DNS-rebinding attempts (Host header
  check). Web pages can't talk to it.

## AI and outside services

All opt-in, off by default, and listed in one privacy settings screen
that shows what goes where.

- **Local models first.** Ollama / LM Studio and in-process embedding
  models are first-class and recommended in onboarding.
- **Bring your own key:** requests go straight from the device to the
  provider, never through Satchel servers.
- **Cloud AI proxy:** for non-technical users. Requests pass through
  without being stored or logged, using providers under zero-data-
  retention terms. The app labels this clearly as "leaves your device".
- **Enrichment lookups** (TMDB, Nominatim, Open Library) send only the
  search term. Through Cloud's proxy they also hide the user's IP from the
  provider. Each provider can be turned off.
- **Semantic search** embeddings are computed on the device, stored inside
  the encrypted sync records, and searched locally. Embeddings can be
  partially inverted back to text, so they are never stored unencrypted
  on the server.

## MCP

- **Local MCP** (desktop, stdio): read-only by default; write tools need
  an explicit toggle. Clients are listed in settings and can be revoked.
  Tool calls are logged locally so the user can see what an AI read.
- **Remote MCP** (Claude/ChatGPT connectors): the server can't answer
  these, so it relays them to the user's desktop app, which answers from
  its local vault.
  - TLS terminates on the desktop, not the relay: the relay routes by SNI
    and only forwards encrypted bytes, like Tailscale Funnel. The desktop
    holds the certificate key for its own relay hostname.
  - OAuth for the connector is handled by the desktop app too.
  - Works only while the desktop app is running; the app says so.
  - Self-hosters can skip the relay and expose the desktop app's MCP
    endpoint through their own tunnel.

## Server hardening

- Device-signed requests on top of TLS; short-lived session tokens.
- Rate limiting and lockout on login.
- Account deletion removes the vault file and blobs immediately; Cloud
  backups age out within a stated window (e.g. 30 days).
- Server backups are ciphertext by construction, so they can live
  anywhere.
- Minimal logs: no request bodies, IPs kept only briefly for abuse
  prevention.

## Supply chain and process

- `cargo-deny` and `cargo-audit` in CI; `pnpm audit` for the extension
  and mobile app; dependencies pinned with lockfiles.
- Signed releases; reproducible builds as a later goal.
- A security policy with a disclosure address before the first public
  release.
- External cryptography and security audit before Cloud launches.
