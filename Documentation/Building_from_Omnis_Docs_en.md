# Building the RAG Server from Your Omnis Documentation

## What this project is, and what it is not

This repository shows **one way** to turn the Omnis documentation into a RAG server that AI coding
assistants can query. It is written so that a developer with no RAG experience can get a picture of
the whole path (documents → text → chunks → embeddings → database → search → MCP tools) and build a
**first running MVP** from it. It is an example, not a product:

- The scripts were developed against specific Omnis documents. With other files or another version
  they may have to be adapted. They do not announce it when they fail quietly.
- Other approaches are valid (other extractors, chunk sizes, embedding models, vector stores). The
  documents in this folder explain why this project made the choices it made, so you can make your own.
- **Have the result validated and reviewed, ideally with an AI agent.** Extracting technical
  documentation is error-prone, and automatic checks only catch the damage they were written for.
  [AI_Review_Guide_en.md](AI_Review_Guide_en.md) is a document written for an AI agent: it describes
  the problems, where the scripts are tied to the documents and what to check. Give it to your agent
  when you provide your documents.

Suggested path for a beginner: read [RAG_concept_en.md](RAG_concept_en.md), then this guide, then run
the pipeline on the PDFs alone (no doc pack) to get a first working server. Add the doc pack and
your own evaluation questions afterwards.

---

This repository contains **no Omnis documentation** and nothing generated from it. The manuals and
the HTML help are © Omnis Software Ltd. Every developer builds the corpus from their own, licensed
Omnis installation. This guide walks through that build, names what goes wrong when the
documentation is extracted, and lists what to check after each step.

> Do not publish the PDFs, the doc pack, `output/` (extracted Markdown, chunks, embeddings) or a
> database dump. Embeddings and chunk text are derived from the documentation. Share them only where
> your Omnis licence allows it, for example inside your team. The `.gitignore` of this repository
> excludes all of them.

---

## 1. What you need

| Input | Used for | Where it comes from | Where it goes |
|---|---|---|---|
| `CommandRef.pdf` | `omnis-commands` | Omnis PDF manual "Command Reference" | `Omnis PDF/` |
| `FunctionRef.pdf` | `omnis-functions` | Omnis PDF manual "Function Reference" | `Omnis PDF/` |
| `Programming_Omnis.pdf` | `omnis-programming` | Omnis PDF manual "Programming with Omnis Studio" | `Omnis PDF/` |
| **Doc pack** (optional), made from the HTML help | `omnis-notation`, deprecation data, entries missing from the PDFs | The HTML help in the installation folder of your Omnis Studio; you convert it yourself | any folder, set `OMNISDOC_PACK` |

The three file names are fixed in `scripts/extract.py`. The doc pack is optional. Without it the
pipeline builds the three PDF corpora only: no notation corpus (`$line`, `$search`, …), no deprecation
flags and none of the entries that exist only in the HTML help.

The PDFs and the HTML help are different versions of the documentation (see
[Version drift](#version-drift)). Take both from the Omnis version your team writes code for.

---

## 2. Step by step

### 2.1 Put the PDFs in place

Copy the three PDFs into `Omnis PDF/`.

### 2.2 Provide the doc pack (optional)

The HTML help ships with your Omnis Studio installation (a folder `idehelp/omnis` with `commands/`,
`functions/` and `notation/`). `scripts/chunk.py` does not read HTML. It reads a **doc pack**: the
help converted to Markdown plus JSONL catalogs. This repository contains no converter. You provide
the pack, and the pipeline only specifies what it expects:

```text
<pack>/
├── catalogs/
│   ├── commands.jsonl      one JSON object per line: name, markdown_path, deprecated, modern_equivalent, scope_category
│   ├── functions.jsonl     name, markdown_path
│   └── notation.jsonl      name, markdown_path, container, container_canonical, deprecated
└── md/
    ├── commands/…          one Markdown page per help page
    ├── functions/…
    └── notation/…
```

- `markdown_path` is relative to the pack root (`md/commands/…`). `name` is the exact entry name as
  shown in the help (`Do method`, `mid()`, `$search`).
- Command and function pages: optional front matter, a `# Title`, a `## Metadata` section as a
  two-column Markdown table (field, value), and the remaining sections as `## Heading` (Syntax,
  Description, Example, …). Code is fenced. A `Source:` line is ignored.
- Notation pages: front matter with `title`, a description, and per section (`## Properties`,
  `## Methods`, …) a Markdown table whose first column is the member name and the other columns are
  its description. Members that are not named `$…`, `#…` or `k…` are treated as plain text.
- Only fields listed above are used. Everything else in the catalogs is ignored.

Point the pipeline to the pack in `scripts/.env.local` (git-ignored) or with `--omnisdoc`:

```env
OMNISDOC_PACK=/path/to/omnisdoc
```

`chunk.py` prints `Doc pack: <path>` as its first line; check it. Without a pack use
`chunk.py --without-docpack`.

**Use the complete help.** Some installations contain a second, reduced help folder next to the full
one (here: `jsidehelp` next to `idehelp`) with the same layout but only a subset of the pages. Nothing
fails, the corpus is just incomplete. Compare the page counts of the folders before you convert.

### 2.3 Run the pipeline

```bash
python scripts/pipeline.py setup
python scripts/pipeline.py doctor     # shows whether embedding runs on GPU/Apple Silicon (minutes) or CPU (hours)
python scripts/pipeline.py build      # extract → chunk → validate → embed → import → eval
```

Or step by step: `extract.py` → `chunk.py` → `validate.py` → `embed_and_store.py` → import → `eval_retrieval.py`.
Details: [Pipeline_en.md](Pipeline_en.md). Database and server setup: the READMEs of the Docker
folders.

**`validate.py` is a gate, not a proof.** It exits non-zero on structural damage it was written to
find. Whether the content is right still needs the checks below, and some of its checks cannot fail
for certain inputs (for example the bookmark check on a PDF without bookmarks).

**Adapting the scripts.** If the documents differ from the ones the scripts were written for, the
scripts have to change. Fonts, sizes, label texts and file names are hard-coded in `scripts/extract.py`,
and `validate.py` contains fixed word lists. The table "Where the scripts are tied to the documents" in
[AI_Review_Guide_en.md](AI_Review_Guide_en.md#2-where-the-scripts-are-tied-to-the-documents) lists
every such place.

---

## 3. Problems when extracting the PDFs

These are the traps found in the Omnis manuals. Expect the same classes of problem in any PDF that is
not tagged. Background and the fixes implemented in `scripts/extract.py`:
[chunking_concept_en.md](chunking_concept_en.md).

| # | Problem | Symptom in the output | Why | What to check |
|---|---|---|---|---|
| 1 | **Headings are not tagged** | Wrong titles; code lines used as section titles | PDF converters guess headings from font size; the code font is larger than the section titles | No chunk title looks like code (`Calculate x as …`, `$ctask.$x()`) |
| 2 | **Entry title looks like a label** | A command or function is merged into the previous entry and disappears | Entry titles use the same bold font as "Syntax", "Description" | Count of entries = count of bookmarks (CommandRef) or of metadata tables (FunctionRef) |
| 3 | **No bookmarks or structure** | Nothing to anchor sections to | Re-saved PDFs (e.g. via macOS Preview) lose bookmarks | Does the PDF have a table of contents (`fitz.open(...).get_toc()`)? Which extraction path does the log report? |
| 4 | **Ligatures "fi" / "fl"** | `specifed`, `frst`, `fag`, `usequalifers` | Table extraction drops the second character of a ligature | No ligature damage left; sample words with "fi"/"fl" in tables |
| 5 | **Code without spaces** | `Calculatexas1` | Glyphs are positioned, no space characters in the PDF | Open the code blocks of a few entries and read them |
| 6 | **Metadata tables render inconsistently** | Missing "Flag affected", platform, group | The Markdown conversion produces three table variants and sometimes plain text | Compare the metadata line of 5–10 entries against the PDF page |
| 7 | **Multi-column index pages** | Lists such as obsolete commands or function groups are missing or scrambled | Column order is lost in the Markdown conversion | Anything that comes from index pages is checked separately |
| 8 | **Page-edge clipping** | Header cells like `Pla` instead of `Platform` | Text clipped at the page border | Metadata keys must come out complete |
| 9 | **Page furniture** | Headers, footers and page numbers inside text | Repeated on every page | Grep for repeated lines across chunks |
| 10 | **Oversized entries** | Chunks far above the size limit | Long tables or code listings | `validate.py` fails above 600 words (`chunk.py` splits at 450); check how long entries were split |

Rules of thumb that apply to every PDF:

- **The heuristics are tuned to specific files.** Font names and sizes (Montserrat-Bold at certain
  point sizes, LMMono10 for code) are hard-coded in `extract.py`. Another manual version, a re-saved
  file or a different export can break them silently. After any change of input, re-check by hand,
  do not rely on the exit code alone.
- **Compare against the source.** Take ten entries spread over the document (first, last, long, short,
  one with a table, one with code) and compare PDF page and chunk side by side.
- **Count, don't read.** Entries per PDF, chunks per corpus, bookmarks per command. A count that
  changes by a few percent between runs is a finding.

---

## 4. Problems with the HTML help and the doc pack

The help is generated HTML with many irregularities. Whatever converter you use, check the result
against the help in a browser.

| Problem | What to check |
|---|---|
| Incomplete help folder | Page counts: commands, functions and notation pages against the full help |
| Wrong Omnis version | A pack is version-specific. Convert the help of the version you develop for |
| Method signatures with broken parentheses or stray spaces | Search the pack for signatures whose `(` never closes |
| Tables with merged or empty cells (notation members, parameters) | Spot-check five notation pages with tables |
| Empty placeholder rows in notation tables | Sample constants and hash-variable pages |
| Links that point nowhere | Count them during conversion; keep the number small and explainable |
| Deprecation information | Pick known legacy commands and confirm that `deprecated` is set |
| Catalog and pages disagree | Every `markdown_path` exists; every page has a catalog row |

### Version drift

| Source | Version | Consequence |
|---|---|---|
| PDF manuals | whatever you have (the revision number is printed on the first pages and read by `extract.py`) | May lack newer commands and functions |
| HTML help | the installed Omnis version | May contain entries the PDF lacks (and the reverse) |

`chunk.py` merges both: entries that exist only in the help are added (`source: OmnisDocPack`), PDF
entries that the help no longer lists get `in_help_11_1: false` (the key name is historical). Read the
informational section of the `validate.py` output: it lists the entries that exist in only one source.
A large list means the PDF and the help come from different versions.

---

## 5. Review with an AI agent

Give [AI_Review_Guide_en.md](AI_Review_Guide_en.md) to your coding agent together with your documents
and ask it to run the pipeline and the review step by step. It contains the problems above in
detail, the detection method for each, a procedure and a report format. Read its findings yourself:
the agent can be wrong too, which is why it has to show evidence (page, extracted text and chunk side
by side).

---

## 6. Checklist

After every build:

1. The doc pack (if used) passes your own checks from section 4.
2. `python scripts/validate.py` exits with 0. Read its informational section, not only the exit code.
3. Entry counts match the sources: commands = PDF bookmarks, functions = PDF metadata tables.
4. No chunk title looks like code; no ligature damage; no chunk is empty or oversized.
5. Ten PDF entries and (with a pack) five notation pages compared by hand with the source.
6. The doc pack is the one you intend: `chunk.py` prints `Doc pack: <path>` at the start.
7. `python scripts/eval_retrieval.py --compare` gives plausible numbers with **your own** queries:
   `scripts/eval_queries.json` is an example set; replace or extend it with questions your developers
   actually ask, and name the expected hit per query.
8. The MCP tools answer from the VS Code client: search for a command you know, then fetch its full
   text with `get_omnis_doc` and read it.
9. Nothing derived from the documentation is staged for git: `git status` shows no PDFs, no
   `output/`, no pack.

When the Omnis version changes: replace the PDFs, rebuild the doc pack from the new help and
run `pipeline.py build` again. `embed_and_store.py` re-embeds only changed chunks.
