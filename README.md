# bmctl

**Firefox bookmark toolkit** - audit duplicates, compare exports, merge collections, and generate an interactive self-hosted dashboard from the command line.

---

## Overview

bmctl is a single-file Python CLI for managing Firefox bookmark exports (`.json`). It parses the full folder hierarchy, preserves the original Firefox tree order, and provides five commands covering the full lifecycle of a bookmark collection.

```
audit      Audit a single export - duplicates, stats, folder tree
compare    Diff two exports - what was added, what was removed
merge      Merge two exports into a clean Netscape HTML file
export     Export to CSV, Excel or Markdown
dashboard  Generate a self-hosted interactive HTML dashboard
```

---

## Requirements

```bash
pip install pandas openpyxl
```

Python 3.8+. No other dependencies for core functionality (`pandas`/`openpyxl` only required for `export --format xlsx`).

---

## Export your Firefox bookmarks

`Bookmarks > Show All Bookmarks > Import and Backup > Backup...`

Save as `.json`. This file is the input for all commands.

---

## Commands

### `audit` - Inspect a single export

```bash
python bmctl.py audit -i bookmarks.json
```

Output:
```
======================================================================
               RAPPORT D'AUDIT GLOBAL
======================================================================
 Total des favoris trouves    : 3110
 Dossiers parcourus           : 244
 Liens uniques                : 3059
 Doublons detectes            : 51 (1.6%)
======================================================================

[!] Top 10 des liens les plus dupliques :
  - GitHub
    URL : https://github.com
    Present 3 fois :
      * Dossier : Dev
      * Dossier : CTI, OSINT & SocMint > Code Search Engines
```

**Options:**

| Flag | Description |
|---|---|
| `-i / --input` | Firefox JSON export (required) |
| `--top N` | Show top N most duplicated URLs (default: 10) |
| `--show-short` | Stats only, skip duplicate list |
| `--show-tree` | Print full folder hierarchy with bookmark counts |

**`--show-tree`** is useful to verify that your folder structure was parsed correctly before generating a dashboard:

```bash
python bmctl.py audit -i bookmarks.json --show-tree
```

```
  |   [   0]  Blogs & Press
    +-- [  69]  Cybersecurity & CTI
      +-- [   6]  Onion
    +-- [   4]  Business
    +-- [   4]  Crypto
    +-- [   2]  Sport
```

---

### `compare` - Diff two exports

```bash
python bmctl.py compare -o bookmarks-old.json -n bookmarks-new.json
```

Output:
```
======================================================================
                 RAPPORT DE COMPARAISON
======================================================================
 Favoris uniques V1 (Ancien)  : 2980
 Favoris uniques V2 (Nouveau) : 3059
 Delta net                    : +79
======================================================================
 [+] Nouveaux favoris ajoutes   : 102
 [-] Anciens favoris supprimes  : 23
======================================================================

[+] APERCU DES NOUVELLES ENTREES (Max 15) :
  + HackTricks                     (Dossier: Cybersecurity > Documentation & Articles)
    https://book.hacktricks.xyz
```

**Options:**

| Flag | Description |
|---|---|
| `-o / --old` | Old JSON export (required) |
| `-n / --new` | New JSON export (required) |
| `--show-full` | Show complete added/removed lists (no limit) |
| `--show-short` | Stats only, skip item lists |

---

### `merge` - Merge two exports

Merges two bookmark collections into a single Netscape HTML file (importable by any browser). Detects URL conflicts (same URL in different folders) and resolves them.

```bash
python bmctl.py merge -b bookmarks-base.json -n bookmarks-new.json -o merged.html
```

With automatic conflict resolution (keeps most recent):
```bash
python bmctl.py merge -b base.json -n new.json -o merged.html --no-confirm
```

Without `--no-confirm`, conflicts trigger an interactive prompt:
```
[?] CONFLIT DE DOSSIER DETECTE POUR :
    - URL : https://example.com
    - Titre : Example Site
    Dans quels dossiers souhaitez-vous le conserver ?
      1) [Garder] -> Dev > Tools
      2) [Garder] -> Misc.
      3) Ignorer / Garder la version avec la date la plus recente
    Votre choix (1, 2...) :
```

Tags from all instances are merged onto the surviving node.

**Options:**

| Flag | Description |
|---|---|
| `-b / --base` | Base JSON export (required) |
| `-n / --new` | JSON to merge in (required) |
| `-o / --output` | Output HTML file (required) |
| `--no-confirm` | Auto-resolve conflicts silently (keeps most recent) |

---

### `export` - Export to flat formats

```bash
# CSV
python bmctl.py export -i bookmarks.json --format csv -o bookmarks.csv

# Excel
python bmctl.py export -i bookmarks.json --format xlsx -o bookmarks.xlsx

# Markdown (organized by folder)
python bmctl.py export -i bookmarks.json --format md -o bookmarks.md
```

All formats include: Title, URL, Folder path, Tags, Date added.

**Options:**

| Flag | Description |
|---|---|
| `-i / --input` | Firefox JSON export (required) |
| `--format` | `csv`, `xlsx`, or `md` (required) |
| `-o / --output` | Output file path (required) |

---

### `dashboard` - Interactive HTML dashboard

Generates a fully self-contained single-file HTML dashboard. No server required - open directly in a browser.

```bash
python bmctl.py dashboard -i bookmarks.json -o dashboard.html
```

**Features:**
- Sidebar with full collapsible folder tree (Firefox order preserved)
- Three views: Dashboard (widget grid), Cards, Table
- Global search across title, URL and tags
- "Recent additions" quick view (last 50)
- Folder-aware widget titles (relative path in folder view, full path in global view)
- Pure black enterprise theme

**Options:**

| Flag | Description |
|---|---|
| `-i / --input` | Firefox JSON export (required) |
| `-o / --output` | Output HTML file (default: `dashboard.html`) |

> **WSL users:** use Linux paths to avoid backslash stripping.
> ```bash
> # Correct
> python bmctl.py dashboard -i /mnt/c/Users/you/Desktop/bookmarks.json \
>                           -o /mnt/c/Users/you/Desktop/dashboard.html
> # Wrong - bash strips backslashes without quotes
> python bmctl.py dashboard -i ... -o C:\Users\you\Desktop\dashboard.html
> ```

---

## URL deduplication logic

bmctl normalizes URLs before comparing them to find true duplicates:

- `http://` and `https://` are treated as the same scheme
- `www.` prefix is stripped
- Trailing slashes are removed
- `utm_*` tracking parameters are removed
- Query parameters are preserved (different queries = different pages)

So `http://www.github.com/` and `https://github.com` are considered the same URL.

---

## Firefox JSON compatibility

bmctl handles both Firefox export formats:

- Legacy format: `typeCode: 1` (bookmark) / `typeCode: 2` (folder)
- Modern format: `type: "text/x-moz-place"` / `type: "text/x-moz-place-container"`
- Fallback: presence of `uri` vs `children` fields

---

## Project structure

```
bmctl.py
  BookmarkNode          Data model for a single bookmark
  UrlNormalizer         URL normalization / deduplication
  BookmarkDatabase      JSON parser + in-memory index
  BookmarkAuditor       Duplicate detection + reporting
  BookmarkComparator    Two-database diff
  BookmarkMerger        Merge + conflict resolution + HTML export
  BookmarkDashboardGen  Interactive HTML dashboard generator
  BookmarkExporter      CSV / Excel / Markdown export
```
