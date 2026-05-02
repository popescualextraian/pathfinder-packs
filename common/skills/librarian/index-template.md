# Vault Index

This index is maintained by the `librarian` skill. It is the entry point Claude (and downstream skills) use to find content in this vault. Edit it freely — librarian mimics whatever style it finds here on the next ingest or reindex.

**Per-document line format** (one line per doc, no nested anchors):

```
- `[tier]` [Title](path/to/note.md) — 1–3 sentence description
```

The tier tag (from the note's `relevance:` frontmatter) lets downstream skills filter what to load:

- `core` — foundational, broadly relevant (workflows, glossary, system maps).
- `reference` — pulled when topically relevant (specs, ADRs, process docs). Default.
- `archive` — rarely loaded unless explicitly queried (most meeting notes, transcripts).

Section headings (`##`) typically match top-level folders. Add new sections as needed; librarian appends to a matching one or asks before creating a new heading. The one auto-created heading is `## attachments` (paired with the `attachments/` folder librarian creates for standalone images).

If a section grows past ~30 entries, librarian flags it and you can run "split <section> section" to move it into its own `<section>/index.md`. The master then carries a single pointer line for that section.

## sources

<!-- Example entry — delete once real entries exist. -->
- `[reference]` [Example Document](sources/example.md) — one-to-three sentence description of what this document covers and why it's in the vault.

## notes

<!-- Add `## section` headings matching your top-level folders. -->
