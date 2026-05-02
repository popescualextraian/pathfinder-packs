# Vault Index

This index is maintained by the `librarian` skill. It is the entry point Claude (and downstream skills) use to find content in this vault. Edit it freely — librarian mimics whatever style it finds here on the next ingest or reindex.

**Per-document block format:**

- `` - `[tier]` [Title](path/to/note.md) — 1–3 sentence description ``
- nested H1/H2 headings as anchor links into the note

**Tier tag** (from the note's `relevance:` frontmatter) lets downstream skills filter what to load:

- `core` — foundational, broadly relevant (workflows, glossary, system maps).
- `reference` — pulled when topically relevant (specs, ADRs, process docs). Default.
- `archive` — rarely loaded unless explicitly queried (most meeting notes, transcripts).

Section headings (`##`) below typically match top-level folders. Add new sections as needed; librarian will append to a matching one or ask before creating a new heading.

## sources

<!-- Example entry — delete once real entries exist. -->
- `[reference]` [Example Document](sources/example.md) — one-to-three sentence description of what this document covers and why it's in the vault.
  - [Introduction](sources/example.md#introduction)
  - [Methodology](sources/example.md#methodology)
  - [Results](sources/example.md#results)

## notes

<!-- Add `## section` headings matching your top-level folders. -->
