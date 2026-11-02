# Product Context

## The problem

Years of using Obsidian and Notion as a "second brain" showed that note
taking and clipping are different problems. Notes are written, linked and
revised. Clips are captured in a second and retrieved later, usually by
searching. Note apps treat a clipped article, a restaurant and a movie the
same way: a page of text with some properties. Capture is slow, and the
clipped thing loses its shape.

Anybox got the clipping side right: one quick way to save any kind of
information, and fast search to get it back. Satchel takes that idea,
combines it with the capture quality of the Obsidian Web Clipper, makes it
self-hostable, and drops note taking entirely.

## What Satchel is

- **A vault of typed clips.** Every item has a type (article, place,
  movie…) provided by a module. Modules give each type its own fields,
  capture logic, enrichment and views.
- **Fast to capture.** One action from the browser, the phone share sheet,
  a desktop hotkey, or the clipboard. Capture never blocks on the network.
- **Fast to retrieve.** Search and filtering respond on every keystroke,
  against the local copy of the vault. Speed is a core feature, not a
  polish item.
- **Local-first.** Every app keeps the full vault on the device and does
  its own capture, extraction and search, like the Obsidian Web Clipper.
  No server is needed to use it on one device.
- **Private by design.** Sync is end-to-end encrypted, always. Neither a
  self-hosted server nor Satchel Cloud can read a vault, and nothing
  leaves a device unless the user turns it on.
- **Sync your way.** Homelabbers self-host sync with one Docker container
  and no other services. Everyone else subscribes to Satchel Cloud.
- **Optional AI capture.** A language model can classify clips, fill in
  fields and suggest tags. Technical users bring their own API key or a
  local model; Cloud subscribers get it built in.
- **A real database, not a folder of files.** Items are typed records in
  SQLite, so the app opens instantly, search stays fast at any size, and
  sync moves small changes in the background instead of re-checking
  files behind a progress bar.
- **Easy to back up and leave.** Automatic, verified, encrypted backups
  to any folder. The vault can always be exported as plain Markdown + YAML
  front matter + attachments, readable without Satchel.
- **Open to AI.** An MCP server lets Claude, ChatGPT or any MCP client
  search and read the vault, and add clips.

## What Satchel is not

- Not a note editor. No backlinks, no daily notes, no block editor. An
  item can carry a short comment, but long-form writing belongs elsewhere.
- Not a read-it-later service run by someone else. No cloud dependency.
- Not a general database builder like Notion. Types come from modules, not
  from user-defined schemas (custom fields on top are fine).

## Principles

0. **Privacy first.** When privacy and convenience conflict, privacy wins
   and the convenient option becomes an explicit opt-in. Details in
   [security](security.md).
1. **Speed.** Keystroke-to-results under one frame (~16 ms) for a
   100k-item vault on desktop. Any feature that threatens this needs a
   reason.
2. **Capture is sacred.** Saving must succeed offline and instantly;
   extraction and enrichment happen afterwards.
3. **The content keeps its shape.** Each module presents its type the way
   that type wants to be seen.
4. **Your data outlives the app.** Open formats, full exports, boring
   backups.
5. **No infrastructure tax.** Self-hosting is one container with a data
   folder. Anything that would need a second service runs on the device
   instead.
6. **Same app, two audiences.** Self-hosted and Cloud users run the same
   apps and the same sync protocol. Cloud sells convenience (no server,
   no keys, remote MCP that just works), not features held back from
   self-hosters.

## Business model

- Apps and self-hosted server: free and open source (AGPL-3.0).
- Satchel Cloud subscription: end-to-end encrypted sync and storage, an
  MCP relay so Claude/ChatGPT connectors reach the desktop app without a
  tunnel, AI-assisted capture without API keys, and proxied enrichment
  providers (TMDB etc.). Cloud never has access to vault contents, so it
  can't sell features that would need it.

## Inspirations

- Anybox — capture anything, fast retrieval
- Obsidian Web Clipper — capture quality, templates, Defuddle extraction
- Raycast / Zed — keyboard-driven speed
- Letterboxd, Google Maps saved places, Goodreads — what a dedicated
  movie / place / book experience feels like
