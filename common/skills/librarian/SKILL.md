---
name: librarian
description: Ingest a document from a vault's raw/ folder, convert it to Markdown, place it in an existing location chosen with the user, and update the vault's index. Use whenever the user drops a file into raw/, says "ingest" / "file this" / "add this to the vault" / "convert this PDF/DOCX", asks to move or re-tag a vault note, or asks to "reindex" / "rebuild the index" / "find missing notes in the index". This skill ONLY indexes — it does not load notes into context, summarize for chat, extract requirements, or build graphs. Use it eagerly for anything that looks like vault housekeeping, even if the user does not say the word "librarian".
---

# Librarian

Librarian is the shared ingest layer for a Markdown vault. It takes a source document, converts it to MD, places it in the right existing folder, and maintains an index that other skills (BA, Architect, …) can read to find content.

## What this skill does

- Convert one file from `raw/` (or one specified by the user) to Markdown.
- Propose a destination from existing vault folders; let the user confirm.
- Write standard frontmatter on the new note (title, source, summary, relevance tier).
- Add or update one block in the vault's index file, including chapter/subchapter anchor links.
- Reindex on demand: refresh changed entries, find orphan notes, flag stale ones.

## What this skill does NOT do

- It does **not** load notes into the conversation context. A separate `loader` skill (future) is responsible for that. Stop at writing the index — never read it back to "brief Claude".
- It does **not** scaffold a vault, invent folder structures, or impose a schema. It discovers and mimics what already exists.
- It does **not** split large sources into multiple derived notes — one source, one note. Role-specific extractors (BA interview → requirements, Architect input → ADRs) do that downstream.
- It does **not** create an index file uninvited. If none exists, do conversion + placement and skip indexing entirely. The user opts in by copying `index-template.md` to their vault root.

## Vault model

Three pieces are configurable; everything else is discovered:

- `raw_dir` — where source files land (default `raw/`). The only required folder.
- `processed_raw` — where ingested originals are moved after conversion (default `raw/_done/`; `null` to leave them in place).
- `image_handling` — `describe` (write a sidecar MD with a vision-generated description) or `store-only` (just save the file).

Config lives at the vault root in `.vault.yaml`. If absent, use the defaults above. A starter is shipped as `default-vault.yaml` in this skill folder.

## Locating the vault

Walk up from the current working directory until you find one of:
1. `.vault.yaml`
2. A `raw/` directory
3. A file named `index.md`, `INDEX.md`, `MOC.md` containing links to other `.md` files

Use the first match. If none is found, ask the user where the vault is. Only initialize a new vault if they explicitly ask — and even then, the minimum is `mkdir raw` plus copying the templates from this skill folder. Do not invent additional structure.

## Discovering vault structure

Before proposing anything, scan what the user has. Read enough to mimic, not catalog:

- List top-level directories (excluding `raw/`, `processed_raw`, `.git`, hidden dirs).
- For each candidate destination dir, sample 2–3 files: capture frontmatter fields actually in use, filename style (kebab-case, dated prefix, etc.), and any obvious purpose hints (folder name, sibling H1s).
- Locate the index file if any: try `index.md`, `INDEX.md`, `README.md`, `MOC.md` at vault root. Treat as the index iff it contains at least one Markdown link to a `.md` inside the vault.

The point is to write notes that look like they belong, not to enforce a global standard.

## Conversion

Pick the converter from the file extension:

| Extension | Tool | Notes |
|---|---|---|
| `.pdf`, `.pptx`, `.xlsx`, `.html`, `.htm` | `markitdown <file> -o <out>.md` | Primary unified converter. |
| Image (`.png`, `.jpg`, `.jpeg`, `.webp`) | image pipeline (see below) | Not converted to MD directly. |
| `.docx`, `.odt`, `.epub`, `.tex`, `.rtf` | `pandoc -t gfm -o <out>.md <file>` | Better structure preservation than markitdown for these. |
| `.md`, `.markdown`, `.txt` | passthrough (copy + add frontmatter) | |
| Anything else | ask the user | Don't guess. |

If a binary is missing, report it once with the install command (`pip install markitdown`, `apt install pandoc`, etc.) and skip that file rather than aborting the batch. The user may have other files to process.

If markitdown fails on a `.pdf` (scanned document, broken PDF), suggest pandoc isn't an option and offer `pdftotext` or `pymupdf4llm` as fallbacks.

## Image handling

Standalone images dropped in `raw/`:
- Move the file into the vault's existing image location if you can identify one (look for `attachments/`, `images/`, `_assets/`, or wherever sibling notes link images from). Otherwise place it next to the note in the chosen destination folder.
- If `image_handling: describe`, use the Read tool on the image to generate a 1–3 sentence description. Write a sidecar `.md` with frontmatter (`type: image`, `source:`, `summary:`) and the image embedded via `![](relative/path.png)`. The sidecar is what gets indexed.
- If `image_handling: store-only`, skip the description; the image is moved but not indexed.

Images embedded inside a converted document are extracted by markitdown/pandoc into a sibling folder; leave them in place and just verify the relative links in the converted MD still resolve.

## Destination proposal

After conversion, decide where the note goes:

1. Form a one-line "what this is" from the filename + first paragraph + any obvious headings.
2. Match against the discovered top-level folders. Score by folder-name similarity, frontmatter `type:` matches in siblings, and content cues.
3. Present the top 1–2 candidates with one-line reasoning each, plus an explicit "or somewhere else — type a path" option. Always confirm before writing — don't auto-place even when the match looks obvious. Cheap to confirm; expensive to misfile.
4. If the user names a path that doesn't exist, confirm folder creation explicitly. Never create folders silently.

## Frontmatter on the new note

Copy the schema observed in the chosen folder's siblings. On top of whatever they use, ensure these fields are present (add them if missing in siblings — they're the contract librarian and downstream skills rely on):

```yaml
---
title: ...                  # title — frontmatter title from siblings if used; otherwise from filename
source: raw/foo.pdf         # original path inside raw/
source_kind: pdf            # pdf | docx | pptx | xlsx | html | image | md | txt | ...
ingested_at: 2026-05-02     # ISO date, today
summary: |
  1–3 sentence description of what the document is and why it might matter.
  This is what feeds the index entry. Edit freely after ingest.
relevance: reference        # core | reference | archive — see "Relevance tiers"
---
```

Leave domain-specific fields (e.g. `decision:`, `stakeholder:`, `process:`) blank for downstream skills or the user to fill in later. Don't fabricate values.

## Relevance tiers

Each note gets a `relevance:` tier so downstream loader skills can filter what to pull into context.

| Tier | Meaning | Typical examples |
|---|---|---|
| `core` | Foundational, broadly relevant | Domain glossary, business workflows, system maps, governing rules |
| `reference` | Pulled when topically relevant (default) | Specs, ADRs, process docs, technical write-ups |
| `archive` | Rarely loaded unless explicitly queried | Most meeting/standup notes, raw transcripts, historical artifacts |

Propose a tier from cues — filename like `meeting-*` / `standup-*` / `notes-YYYY-MM-DD-*` → `archive`; words like `glossary`, `workflow`, `domain`, `policy` in filename or H1 → `core`; everything else → `reference`. Always ask the user to confirm or override; tier choice is opinionated and a wrong default propagates everywhere downstream.

The vocabulary is extendable per-vault — if you find existing notes using `relevance: critical` or similar, mimic the local convention rather than forcing the three-value default.

## Index handling

The index is **for Claude and downstream skills** to find content. Format favors machine readability over visual prettiness; users can still browse and edit, and librarian mimics whatever style they've established.

### Detection

At the vault root, look for `index.md`, `INDEX.md`, `README.md`, `MOC.md` in that order. Treat as the index iff it contains at least one link to a `.md` inside the vault. If nothing qualifies, **skip indexing entirely** for this run — don't create one. Suggest copying `index-template.md` from this skill folder if the user wants to opt in.

### Per-document block

Each document gets one block:

```
- `[reference]` [Document Title](relative/path.md) — 1–3 sentence description from frontmatter summary
  - [Chapter heading](relative/path.md#chapter-heading)
  - [Another chapter](relative/path.md#another-chapter)
    - [Subchapter](relative/path.md#subchapter)
```

- **Top-level entry**: `` - `[tier]` [Title](path) — description ``. The tier tag is wrapped in inline-code backticks so skills can grep `` `[core]` `` reliably.
- **Sub-entries**: H1 and H2 headings from the converted note, as nested bullets with anchor links. H3+ stay inside the note, not in the index — the index is a navigation aid, not a full TOC dump.
- **Anchor slugs**: GitHub-style — lowercase, spaces → hyphens, strip punctuation. Match what most renderers produce so links actually work.

### Sources for each piece

- **Title**: frontmatter `title:` → first H1 in the note → humanized filename. First match wins.
- **Path**: relative from the index file's directory, forward slashes.
- **Description**: frontmatter `summary:`, copied verbatim (collapsed to a single line if multi-line). On reindex, if a note has no `summary:`, leave the description blank — don't fabricate. The user or a downstream skill can fill it in later.
- **Headings**: H1/H2 lines from the converted MD, in document order.

### Style inference (mimic when possible)

If the existing index uses a different bullet character (`*`, numbered), separator (` - `, ` : `), or path style (root-absolute vs. relative), copy whatever's dominant. The block structure (top-level entry + nested H1/H2 anchors) is the contract — keep that even if the surrounding style varies.

### Where the new block goes

- **Sectioned index** (entries grouped under `##` headings, typically one per top-level folder) with a heading whose name matches the destination folder (case- and kebab-insensitive) → append the block to that section.
- **Sectioned index** with no matching heading → ask the user: append under an existing section (list them) or create a new `##` heading named after the folder. Don't create headings silently.
- **Flat index** → append at the end unless an obvious sort order is in use (alpha by title, by date prefix, etc.).

### Update vs. append on re-ingest

Match by the link target path of the top-level entry. If a block with that path exists, replace it in place — top line plus all nested sub-entries up to the next top-level entry or `##` heading. Never duplicate.

## Reindex

Reindex is a **diff-and-reconcile** operation triggered by phrases like "reindex", "rebuild the index", "find missing notes in the index", or "the index is out of date". It catches three states:

1. **Existing entries** whose underlying note has changed — refresh.
2. **Orphan notes** — `.md` files in the vault that aren't in the index (user created notes manually, or another skill produced them). Add them.
3. **Stale entries** — index points to a file that no longer exists. Flag them.

### Strategy: stateless scan-and-compare

No hash files, no mtime tricks, no hidden metadata in index lines. Each reindex run:

1. **Walk the vault**: list all `.md` files, excluding `raw/`, the configured `processed_raw` dir, attachment folders, the index file itself, and anything in `.vaultignore` if present.
2. **Parse the index**: extract the link target path from every top-level entry.
3. **Compute three sets**:
   - `in_both` — regenerate the block from current frontmatter + headings, but **only rewrite if the regenerated block actually differs**. This avoids spurious diffs and preserves manual edits where the content is unchanged.
   - `only_in_vault` (orphans) — propose to add.
   - `only_in_index` (stale) — propose to handle.
4. **Place orphans**:
   - If the file's parent folder name matches an existing `##` section heading → auto-place there.
   - Otherwise — batch the unplaceable orphans and ask the user once: pick an existing section per file, or create a new heading. Don't pepper them with one question per file.
   - Files without a `summary:` get an empty description in the index entry. Leave it for the user or a downstream skill to fill in.
5. **Handle stale**:
   - Default: comment the entry out in place with a marker — `<!-- stale: <path> not found -->` above the original block. The user can investigate (renamed? deleted?) without losing the entry.
   - On explicit "prune" / "reindex --prune": delete the stale block instead of commenting.
6. **Report**: terse summary, e.g. `4 refreshed, 2 added, 1 stale (commented)`.

### Dry run

If the user asks for a "dry run" or "show what reindex would change", print the three sets and the proposed edits without writing. Useful before committing to a big reconcile.

### Why stateless

Vaults are small enough that re-parsing on every reindex is cheap. The diff is transparent — you can see what changed in `git diff`, no hidden state. There's no risk of stale hash metadata leaking into commits. If perf becomes an issue, an opt-in hash comment can be bolted on later without changing the user-visible format.

## Log

If `log.md` already exists at the vault root, append one line per operation:

```
2026-05-02T14:23 ingest raw/foo.pdf -> notes/papers/foo.md  [reference]
2026-05-02T14:24 reindex 4 refreshed, 2 added, 1 stale
```

If `log.md` does not exist, don't create one. The log is opt-in like the index.

## Move the original

After successful conversion + placement, move the original from `raw/` to `processed_raw` (default `raw/_done/`). If `processed_raw: null` in config, leave it in place. Never delete the original — it's the source of truth.

## Reporting

Per ingested file, give the user a 3-line summary and stop:

```
foo.pdf -> notes/papers/foo-bar-baz.md  [reference]
index: appended to ## sources (4 sub-entries)
original: raw/foo.pdf -> raw/_done/foo.pdf
```

For reindex, one summary line is enough. For a batch ingest, list each file's three lines, then a one-line tally.

## Examples

### Example 1: ingest a PDF

User: "ingest the architecture review I just dropped in raw/"

1. Find the PDF in `raw/`. Confirm with the user if there are multiple.
2. Run `markitdown raw/architecture-review.pdf -o /tmp/arch.md`.
3. Read the converted output; extract title, first paragraph, headings.
4. Scan the vault: top-level folders are `glossary/`, `processes/`, `decisions/`, `meetings/`. Sibling notes in `decisions/` use `type: adr` and dated kebab filenames.
5. Propose: "This looks like an architecture review covering the new payments service. Best fit: `decisions/` (siblings are ADRs, similar tone) or `processes/`. Or another path?"
6. User picks `decisions/`. Write `decisions/2026-05-02-payments-architecture-review.md` with frontmatter copied from siblings + standard fields. Tier: propose `reference` (it's a review, not a foundational policy); confirm.
7. Update `index.md` under `## decisions`. Move original to `raw/_done/`.

### Example 2: reindex finds orphans

User: "reindex"

1. Scan `index.md`: 12 top-level entries.
2. Walk the vault: 15 `.md` files outside `raw/` and `_done/`.
3. Diff: 12 in_both, 3 orphans (`processes/onboarding.md`, `glossary/payments.md`, `meetings/2026-04-15-retro.md`), 0 stale.
4. Auto-place: `processes/onboarding.md` under `## processes`, `glossary/payments.md` under `## glossary`, `meetings/2026-04-15-retro.md` under `## meetings` — all matched.
5. For `in_both`, regenerate blocks; 2 had new H2 headings → rewrite those 2; 10 unchanged → leave alone.
6. Report: `2 refreshed, 3 added, 0 stale`.

### Example 3: no index file present

User: "ingest this docx"

1. Convert with pandoc, propose destination, write the note with frontmatter.
2. Look for an index file at vault root: none of `index.md`, `INDEX.md`, `README.md`, `MOC.md` qualifies.
3. Skip indexing. Report: "Note created at `notes/specs/foo.md`. No index file detected at the vault root, so nothing was indexed. Copy `<this-skill>/index-template.md` to your vault root if you'd like an index started."

## Files in this skill

- `SKILL.md` — this file.
- `default-vault.yaml` — starter config the user copies to their vault root as `.vault.yaml`.
- `index-template.md` — starter index the user copies to their vault root as `index.md` (or similar) to opt into indexing.
