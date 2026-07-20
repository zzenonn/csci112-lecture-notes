# CSCI 112 Slides

[Slidev](https://sli.dev/) presentation decks for CSCI 112/212 — Contemporary
Databases. This is a **pnpm workspace monorepo**: each lecture module is a
self-contained Slidev project in its own folder, sharing one `node_modules`.

Each deck is numbered to match its companion note in [`../notes/`](../notes/). See the
[top-level README](../README.md) for the full module ↔ note mapping and the philosophy
behind the slides/notes split.

## Prerequisites

- **Node.js** (LTS)
- **pnpm** — pin the version automatically with
  [corepack](https://nodejs.org/api/corepack.html): `corepack enable`. The workspace
  root pins the version via the `packageManager` field, so no global pnpm install is
  needed.

Install shared dependencies once from this directory:

```bash
pnpm install
```

## Decks

Each folder is a workspace package. Note that most package names follow
`csci112-<folder>`, **but `08a`/`08b` differ** — so the most reliable way to target a
deck is by its **folder path** (`--filter ./<folder>`) rather than its package name.

| Folder | Package name |
|--------|--------------|
| `01-databases-real-world` | `csci112-01-databases-real-world` |
| `02-nosql-intro` | `csci112-02-nosql-intro` |
| `03-document-oriented-db` | `csci112-03-document-oriented-db` |
| `04-mongodb-data-structures` | `csci112-04-mongodb-data-structures` |
| `05-mongodb-aggregation` | `csci112-05-mongodb-aggregation` |
| `06-mongodb-sharding-replication` | `csci112-06-mongodb-sharding-replication` |
| `07-mongodb-indexing` | `csci112-07-mongodb-indexing` |
| `08a-modeling-inheritance-computed-approximation` | `csci112-08a-modeling-patterns-1` |
| `08b-modeling-reference-versioning-single-collection` | `csci112-08b-modeling-patterns-2` |
| `08c-modeling-subset` | `csci112-08c-modeling-subset` |
| `09-dynamodb-intro` | `csci112-09-dynamodb-intro` |
| `10-dynamodb-data-modeling` | `csci112-10-dynamodb-data-modeling` |
| `11-dynamodb-design-patterns` | `csci112-11-dynamodb-design-patterns` |

## Running a Deck

Target one deck by its folder path. From this `slides/` directory:

```bash
# Live dev server with hot reload (opens in browser)
pnpm --filter ./10-dynamodb-data-modeling dev

# Build a static site (outputs to that deck's dist/)
pnpm --filter ./10-dynamodb-data-modeling build

# Export the deck to PDF (outputs to <deck>/dist/slides.pdf)
pnpm --filter ./10-dynamodb-data-modeling export
```

PDF export uses `playwright-chromium` (already a dev dependency of each deck). If a
fresh machine is missing the browser binary, install it once with
`pnpm exec playwright install chromium`.

## Generating All PDFs

To export every deck and collect the PDFs into one directory, named per module, run
this from the `slides/` directory:

```bash
OUT=./pdfs                          # or any target directory
mkdir -p "$OUT"
for d in [0-9]*/; do
  d="${d%/}"
  pnpm --filter "./$d" export && cp "$d/dist/slides.pdf" "$OUT/$d.pdf"
done
```

Each PDF lands as `$OUT/<deck-folder>.pdf` (e.g. `pdfs/10-dynamodb-data-modeling.pdf`).

## Verifying Slides Fit

`slidev build` only checks that a deck **compiles** — it does not catch content that
**overflows** a slide. After editing, export to PDF and inspect the pages visually to
confirm nothing is clipped, especially slides with large tables or code blocks.

## Conventions

- Build artifacts (`<deck>/dist/`) are gitignored; there is no aggregate `dist/`.
- Slides carry the *pattern* (small examples, templates); the matching note carries the
  *full instance* (complete sample data, runnable scripts, command output).
- Every deck ends with a **Further reading** slide linking to its companion note.
- When a slide references a note, use the full GitHub URL to the note file.
