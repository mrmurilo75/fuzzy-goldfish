# Fuzzy Goldfish — Product Requirements Document (MVP)

Status: Draft v2 — 2026-07-05 (architecture decisions resolved)

## 1. Overview

Fuzzy Goldfish is a self-hosted, Django-based note-taking application. Its
core differentiator is **fast capture** through a custom Markdown-like
language — a deliberately **simplified subset of Markdown** — parsed **live**
in a side-by-side editor. Notes are strictly private per user.

## 2. Target users

- **Students / researchers** — capturing study notes, references, and
  long-form knowledge.
- **Developers / technical users** — Markdown-fluent, keyboard-driven, need
  code snippets in notes, and value self-hosting their data.

## 3. Problem statement

1. **Fast capture** — existing editors add friction (heavy toolbars, full
   Markdown complexity). A minimal, live-parsed syntax gets thoughts down
   faster.
2. **Organization & retrieval** — notes pile up and become unfindable.
   (Addressed minimally in MVP via a filterable list; tags and full-text
   search are post-MVP.)
3. **Ownership / self-hosting** — users want their notes on their own Django
   server, not a proprietary cloud.

## 4. MVP scope summary

In scope: note CRUD, the custom language ("FGM" — Fuzzy Goldfish Markup)
with live side-by-side preview, autosave + manual save, a note list with
client-side title filter, trash/soft delete, private per-user access.

Out of scope for MVP: tags/folders, full-text search, sharing or
collaboration, pinning/favorites, attachments/images, mobile apps, offline
mode, note version history.

---

## 5. Features

### 5.1 Note CRUD

- **Name** — Note CRUD
- **Problem/context** — Users need the basic lifecycle of a note — create,
  open, edit, delete — with zero ceremony, or nothing else matters.
- **Description** — Authenticated users can create a new note, open an
  existing one in the editor, edit its title and body, and move it to trash.
  A new note opens directly into the editor, ready to type.
- **User story** — When a thought strikes, I want to create and start typing
  a note in one action, so I can capture it before I lose it.
- **Acceptance criteria**
  - Given I am logged in, when I click "New note", then a note is created
    and the editor opens with focus in the body within one interaction.
  - Given I own a note, when I open it from the note list, then its current
    title and raw FGM body load in the editor.
  - Given I edit a note's title or body, when the note is saved, then
    reloading the page shows the updated content and an updated
    last-edited timestamp.
  - Given a note belongs to another user, when I request its URL, then I
    receive a 404 (not a 403, to avoid leaking existence).
- **Out of scope** — Sharing, duplication, import/export, note templates.

### 5.2 FGM language (simplified Markdown subset)

- **Name** — FGM language
- **Problem/context** — Full Markdown/CommonMark is larger than fast capture
  needs; a small, predictable subset is faster to type and to parse live.
- **Description** — A custom markup language that is a simplified subset of
  Markdown, limited to: headings, bold/italic emphasis, bullet/numbered
  lists, `- [ ]` task checkboxes, fenced code blocks, inline code, and
  links. One grammar spec with shared test fixtures governs two parser
  implementations: JavaScript (live preview) and Python (canonical
  server-side rendering).
- **User story** — When I write notes, I want a minimal Markdown-like syntax
  that renders identically in the live preview and the saved view, so I can
  trust what I typed is what I keep.
- **Acceptance criteria**
  - Given the shared fixture suite (FGM input → expected HTML), when it runs
    against both the JS and Python parsers, then every fixture passes in
    both.
  - Given a note containing each supported element (heading, bold, italic,
    bullet list, numbered list, checkbox, fenced code block, inline code,
    link), when rendered, then each produces the correct HTML element and
    links render clickable.
  - Given input containing raw HTML or script content, when rendered by
    either parser, then it is escaped/sanitized and never executed.
  - Given Markdown syntax outside the subset (e.g. tables, footnotes), when
    rendered, then it appears as plain text rather than erroring.
- **Out of scope** — Full CommonMark compatibility, tables, footnotes,
  images/embeds, wiki-links/backlinks, syntax highlighting inside code
  blocks.

### 5.3 Live side-by-side editor

- **Name** — Live editor
- **Problem/context** — Waiting for a save/preview round-trip breaks writing
  flow; users want to see structure appear as they type.
- **Description** — The editor shows raw FGM text in a left pane and the
  rendered result in a right pane, re-parsed client-side as the user types.
  The preview updates without any server round-trip.
- **User story** — When I'm typing a note, I want the rendered preview to
  update as I type, so I can stay in flow and catch syntax mistakes
  immediately.
- **Acceptance criteria**
  - Given the editor is open, when I type FGM syntax in the left pane, then
    the right pane updates to the rendered result without a page reload or
    server request.
  - Given a note with existing content, when the editor opens, then the
    preview pane shows the fully rendered note immediately.
  - Given a syntactically incomplete construct (e.g. an unclosed code
    fence), when I type, then the preview degrades gracefully (renders as
    plain text) and never blanks or throws.
  - Given the preview and the saved read view of the same note, then both
    render identical structure (per the shared parser fixtures).
- **Out of scope** — In-place/WYSIWYG rendering, toggle edit/preview mode,
  synchronized scrolling, mobile-optimized editing.

### 5.4 Autosave + manual save

- **Name** — Autosave + manual save
- **Problem/context** — Losing a captured thought to a missed save defeats
  the purpose of fast capture, but users also want an explicit "it's saved"
  confirmation.
- **Description** — The editor autosaves changes in the background
  (debounced) while the user types, and also offers an explicit save action
  (button and Ctrl/Cmd+S). A visible indicator shows saved / saving /
  unsaved state.
- **User story** — When I'm writing quickly, I want my note saved
  automatically and on demand, so I never lose work and can confirm it's
  safe before leaving.
- **Acceptance criteria**
  - Given I stop typing, when the debounce interval elapses, then the note
    persists to the server and the indicator shows "saved".
  - Given I press Ctrl/Cmd+S or click save, then the note persists
    immediately and the browser's native save dialog is suppressed.
  - Given an autosave request fails (e.g. network drop), then the indicator
    shows an unsaved/error state and the editor retries; my text is never
    cleared.
  - Given the note was modified elsewhere since my editor loaded it (e.g. a
    second tab), when a save is attempted with a stale version, then the
    server rejects it and the editor shows a conflict warning instead of
    silently overwriting.
  - Given unsaved changes, when I navigate away or close the tab, then I am
    warned before leaving.
- **Out of scope** — Offline persistence, automatic conflict merging,
  version history / undo across sessions.

### 5.5 Note list with title filter

- **Name** — Note list
- **Problem/context** — Without tags or search in MVP, users still need a
  fast way to find a note among dozens.
- **Description** — The home screen lists the user's notes ordered by last
  edited (most recent first), showing title and last-edited time, with a
  client-side filter box that narrows the list by title as you type.
- **User story** — When I return to my notes, I want the most recently
  edited ones on top and a quick title filter, so I can reopen the note I
  need in seconds.
- **Acceptance criteria**
  - Given I have notes, when I open the note list, then only my own notes
    appear, ordered by last-edited descending, each showing title and
    last-edited time.
  - Given the filter box, when I type a substring, then the list narrows
    client-side (no page reload) to notes whose titles match,
    case-insensitively.
  - Given I clear the filter, then the full list is restored.
  - Given a note is in the trash, then it does not appear in the main list.
- **Out of scope** — Full-text (body) search, tags/folders, pinning,
  sorting options, pagination beyond a sensible default.

### 5.6 Trash (soft delete)

- **Name** — Trash
- **Problem/context** — Accidental deletion of a note is unrecoverable pain;
  a safety net is expected in any note app.
- **Description** — Deleting a note moves it to a per-user trash instead of
  destroying it. Trashed notes can be restored or permanently deleted from a
  trash view.
- **User story** — When I delete a note by mistake, I want to restore it
  from the trash, so I can't permanently lose my work with one click.
- **Acceptance criteria**
  - Given I delete a note, then it disappears from the note list and appears
    in my trash view, and its content is unchanged.
  - Given a trashed note, when I click restore, then it returns to the note
    list with content and timestamps intact.
  - Given a trashed note, when I permanently delete it (with confirmation),
    then it is removed irrecoverably.
  - Given a trashed note's direct URL, when opened, then editing is not
    possible (restore is offered instead).
- **Out of scope** — Automatic time-based purge, bulk operations, admin
  recovery of purged notes.

### 5.7 Private per-user access

- **Name** — Private notes
- **Problem/context** — On a shared self-hosted instance, users must trust
  that their notes are theirs alone.
- **Description** — Every note belongs to exactly one user account. All note
  views, endpoints, and lists are scoped to the authenticated owner; there
  is no sharing surface at all in MVP.
- **User story** — When I use a Fuzzy Goldfish instance shared with others,
  I want my notes visible only to me, so I can write freely.
- **Acceptance criteria**
  - Given an unauthenticated visitor, when any note URL or endpoint is
    requested, then they are redirected to login (pages) or receive 401/403
    (JSON endpoints).
  - Given user A and a note owned by user B, when A requests it by URL or
    id (view, edit, autosave, trash, restore), then the response is 404.
  - Given any note list or trash view, then it only ever contains the
    requesting user's notes.
- **Out of scope** — Share links, multi-user collaboration, teams/roles,
  end-to-end encryption.

---

## 6. Architecture decisions

All decided:

1. **Parser strategy** — dual implementation (JS for live preview, Python
   for canonical rendering) governed by a single grammar spec and a shared
   fixture suite both must pass.
2. **Preview model** — client-side parsing; no server round-trip while
   typing.
3. **API style** — plain Django views for navigation plus lightweight JSON
   endpoints (fetch API) for editor operations. No DRF, no HTMX.
4. **Storage model** — store raw FGM text only; render on demand with the
   Python parser. No cached HTML column.
5. **Autosave concurrency** — optimistic concurrency: each save carries the
   version/timestamp the editor loaded; on mismatch the server rejects the
   save and the editor surfaces a conflict warning. No automatic merging.
6. **Database & deployment** — PostgreSQL in production; Docker Compose is
   the canonical self-host path (cookiecutter defaults).
7. **App structure** — a `notes` Django app for models/views/templates,
   plus a **standalone, framework-free `fgm/` parser package** with its own
   tests, imported by the app. The JS parser lives alongside it as the
   second implementation of the same spec.
8. **Grammar spec** — three artifacts, kept consistent: a formal grammar
   file (EBNF/PEG) as the source of truth, a human-readable spec doc
   (`docs/fgm-spec.md`), and a shared JSON fixture suite (FGM → expected
   HTML) that both parsers must pass in CI.
9. **Editor pane** — a plain `<textarea>` with a small vanilla-JS module
   (debounce, Ctrl/Cmd+S, preview wiring). No editor framework in MVP.
10. **Testing** — pytest for unit/view tests; the shared fixture suite runs
    against the Python parser via pytest and the JS parser via a Node test
    runner, both in CI; the PRD's Given/When/Then acceptance criteria are
    encoded as executable pytest-bdd feature files.

## 7. Success measures (MVP)

- A new user can sign up, create a note, and see live-rendered FGM in under
  a minute.
- Both parsers pass 100% of the shared fixture suite in CI.
- No cross-user note access is possible (verified by tests for every note
  endpoint).
