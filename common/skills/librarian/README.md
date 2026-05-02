# librarian — install & setup

Manual setup notes for the user. The skill itself (`SKILL.md`) is what Claude reads; this file is just for you. A proper setup script will replace these instructions once we have more skills with shared dependencies.

## Prerequisites

Librarian shells out to two converters. Install whichever you'll actually need — the skill degrades gracefully if one is missing (it just refuses to convert formats that need the missing tool).

### markitdown — primary converter

Handles PDF, PPTX, XLSX, HTML, and images. Python package:

```bash
pip install markitdown
# or, isolated:
pipx install markitdown
```

Verify:

```bash
markitdown --version
```

### pandoc — fallback for structured documents

Better than markitdown for `.docx`, `.odt`, `.epub`, `.tex`, `.rtf` (footnotes, complex lists, etc.).

```bash
# Debian / Ubuntu / WSL
sudo apt install pandoc

# macOS
brew install pandoc

# Windows (PowerShell, with winget)
winget install --id JohnMacFarlane.Pandoc
```

Verify:

```bash
pandoc --version
```

### Optional: PDF fallbacks

If markitdown struggles on a particular PDF (scanned pages, broken structure), these are useful:

- `pdftotext` — `sudo apt install poppler-utils` / `brew install poppler`
- `pymupdf4llm` — `pip install pymupdf4llm`

## Initialize a vault

Librarian needs at minimum a `raw/` directory somewhere. To opt into config and indexing:

```bash
cd /path/to/your/vault
mkdir -p raw

# optional: copy the config template
cp /path/to/pathfinder-packs/common/skills/librarian/default-vault.yaml .vault.yaml

# optional: copy the index template to opt into indexing
cp /path/to/pathfinder-packs/common/skills/librarian/index-template.md index.md
```

Edit `.vault.yaml` if the defaults don't fit (raw dir name, processed-raw location, image handling). Edit `index.md` to remove the example entry once you have real notes.

## Using the skill

Drop a file in `raw/`, then ask Claude:

- "ingest the file in raw/"
- "convert and file the PDF I just dropped in"
- "reindex the vault"
- "the index is out of date — find missing notes"

Claude will load `librarian` and walk you through conversion, placement, and indexing.

## Future

When pathfinder-packs has more skills with external dependencies, this README will be replaced by a single `setup.sh` (or equivalent) at the repo root that installs everything for the role you've selected. For now, manual.
