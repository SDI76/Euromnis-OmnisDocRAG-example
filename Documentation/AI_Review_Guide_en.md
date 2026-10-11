# AI Review Guide: Validating the Omnis Documentation Build

**This file is written for an AI coding agent.** Give it to your agent (Claude Code, Codex, Copilot,
…) when you provide your own Omnis documentation to this project, and ask it to carry out the review
below. The scripts were developed against one specific set of Omnis documents. With other files or
another Omnis version they can fail without any error. A second pair of eyes finds that, and an agent
can do it cheaply.

Suggested prompt:

> Read `Documentation/AI_Review_Guide_en.md` and `Documentation/Building_from_Omnis_Docs_en.md`. I put
> my Omnis PDFs in `Omnis PDF/` and (optionally) my doc pack at `<path>`. Run the pipeline step by
> step and carry out the review from the guide after each step. Report findings as the guide
> describes. Do not change scripts before you have shown me the evidence.

---

## 1. Context

The project builds a RAG server from the Omnis Studio documentation:

```text
Omnis PDF/*.pdf  (+ optional doc pack from the HTML help)
  → scripts/extract.py     PDF → units (JSON), output/extracted/ and output/*_extracted.md
  → scripts/chunk.py       units (+ doc pack) → chunks, output/chunks/*_chunks.json
  → scripts/validate.py    structural quality gate, output/validation_report.json
  → scripts/embed_and_store.py   chunks → output/embeddings.jsonl
  → scripts/import_to_*.py       → PostgreSQL + pgvector → rag-server → MCP tools
```

The repository is an **example** of one way to do this. It is not a product, and nothing in it is
guaranteed to work on documents it has not seen. Your job is to find out whether it works on the
documents in front of you, and to say clearly where it does not.

Rules:

- The Omnis documentation is © Omnis Software Ltd. Never commit, upload or paste the PDFs, the doc
  pack, `output/` or text from them into a public place. Quote only what a finding needs.
- Do not change the input files.
- Do not change a script until a check has shown a concrete defect, and show the evidence first (the
  PDF page, the extracted text and the chunk side by side).
- After every change: re-run the affected step and `python scripts/validate.py`.
- Run Python that reads untrusted files with `python -I`.

---

## 2. Where the scripts are tied to the documents

These are the places that most often need adapting when the input differs. Check each one against
your files.

| Where | What is assumed | Check |
|---|---|---|
| `scripts/extract.py`, `SOURCES` | File names `CommandRef.pdf`, `FunctionRef.pdf`, `Programming_Omnis.pdf`; command overview titles ("Command information", "Command Groups", "Error Codes", …); FunctionRef overview page 2; label texts `Command group` / `Function group` | Are these titles and labels present in your PDF? Does page 2 of FunctionRef match? |
| `scripts/extract.py`, `OMNIS_VERSION` | `"11"` | Matches your documents? It ends up in metadata |
| `scripts/extract.py`, fonts | Code font name contains `mono`/`courier`/`code` (`MONO_HINTS`); headings are bold; Programming manual headings at certain sizes (Montserrat-Bold ~11 / 9.2 / 7.7 pt) | Print the fonts and sizes of the first heading, a code line and a metadata label in your PDF (PyMuPDF `get_text("dict")`) |
| `scripts/extract.py`, `REFERENCE_SUBSECTIONS` | Sub-headings "Syntax", "Description", "Options", "Example", … | Same labels in your PDF? |
| `scripts/extract.py`, `LigatureRepair` | Only the ligature characters "fi" and "fl" | Other broken ligatures ("ff", "ffi")? Other languages? |
| `scripts/extract.py`, `document_revision` | Revision printed on the first pages as `Revision NNNNN` | Is it found? |
| `scripts/validate.py` | CommandRef bookmarks at level 3; FunctionRef entries = number of "Function group" labels − 1; chapters numbered consecutively; `LIGATURE_DAMAGE` is a **fixed word list**; `CODE_TITLE` regex; `MAX_CHUNK_WORDS = 600` | These checks pass for the wrong reasons when your PDF has no bookmarks or other damaged words |
| `scripts/chunk.py` | `MAX_WORDS = 450`; the strings `"omnis_version": "11.1"` and "Not listed in the Omnis 11.1 help" are hard-coded; doc pack field names (see `Building_from_Omnis_Docs_en.md`) | Replace the version strings if you use another help version |
| `scripts/eval_queries.json` | Example questions with expected hits (regular expressions) | Replace with questions of your developers; the examples reflect one documentation set |

---

## 3. Known problems and how to detect them

For each problem: what you see, why it happens, how to check. Use counts and samples, not impressions.

### 3.1 PDF extraction

| # | Problem | Detection |
|---|---|---|
| P1 | **Headings are not tagged in the PDF.** Converters guess headings from the font size. The code font is larger than the section titles, so code lines become headings. | In `output/extracted/Programming.json` list the titles that contain `$`, `;`, `=`, `Returns #`, or begin with `Calculate`, `Quit method`. Expected: none. Do the same for titles in the chunk files |
| P2 | **Entry titles look like labels.** A command's title uses the same bold font as the label "Syntax". An entry is merged into the previous one and disappears. | Count entries in `CommandRef.json` against the PDF bookmarks (`fitz.open(...).get_toc()`, level 3) and for `FunctionRef.json` against the number of "Function group" labels in the PDF. Any difference is a lost or invented entry. List the names that are missing |
| P3 | **Bookmarks or structure missing** (re-saved PDF). `validate.py` skips the bookmark check silently when there are none. | Print `len(get_toc())` for each PDF. If it is 0, the bookmark check proves nothing: verify the entry count some other way (the alphabetical order of titles, page ranges without gaps) |
| P4 | **Ligatures fi/fl lost in tables** (`specifed`, `frst`, `fag`). | Search all chunk texts for words that become valid English when an "i" or "l" is inserted after an "f". `validate.py` only knows a fixed word list: extend the check yourself with a dictionary |
| P5 | **Code without spaces** (`Calculatexas1`). Glyphs are positioned without space characters. | Open 10 code blocks (long, short, with strings, with operators) and compare to the PDF page. Search the code blocks for very long tokens without a space |
| P6 | **Metadata tables rendered inconsistently.** The Markdown conversion has several table variants; fields such as "Flag affected" or "Platform(s)" go missing. | For 10 entries compare the metadata line in the chunk with the table in the PDF. Count entries without `Command group`/`Function group`. Check keys for clipped names (`Pla`) |
| P7 | **Multi-column pages** (obsolete commands, function group lists) are scrambled or dropped. | Locate those pages, extract their text with `page.get_text()` and compare to the chunks. Anything that is missing there is lost |
| P8 | **Page furniture** (headers, footers, page numbers) is part of the text. | Find lines that occur many times across chunks |
| P9 | **Oversized or empty chunks.** | Distribution of word counts per corpus; the largest 10 and smallest 10 chunks read by a human |
| P10 | **Tables split across pages** lose their header or merge with the following text. | Find chunks containing `|` rows without a header line |
| P11 | **Wrong version mix.** PDF and help describe different Omnis versions. | The informational part of `validate.py` lists entries that exist on one side only. A long list means different versions. Say which entries are affected |

### 3.2 HTML help and the doc pack

| # | Problem | Detection |
|---|---|---|
| H1 | **Incomplete help folder.** An installation can contain a reduced copy of the help next to the full one. Nothing fails, content is missing. | Compare page counts of the folders; compare the counts of commands, functions and notation pages with the counts in the catalogs |
| H2 | **Wrong version.** A pack is version-specific. | Sample five names that were added or removed in the version you develop for |
| H3 | **Broken signatures.** Parentheses that never close, stray spaces (`( x`, `bFromStart= kTrue`). | Parse every signature for balanced parentheses and look for double spaces |
| H4 | **Tables with merged or empty cells** (notation members, parameter tables). | Spot-check 5 notation pages against the help in a browser. Count members per page against the help |
| H5 | **Empty placeholder rows** in notation tables. | Find member rows with an empty description |
| H6 | **Catalog and pages disagree.** | Every `markdown_path` exists; every page has a catalog row; names are unique |
| H7 | **Deprecation flags wrong or missing.** | Take 10 commands you know to be legacy and confirm `deprecated` is set. Take 10 you know are current and confirm it is not |
| H8 | **Notation names lose their `$`.** The catalog drops `$` in some `$root` paths. | Compare `container` with the page title for ten nodes |

### 3.3 Chunks, embeddings and retrieval

| # | Problem | Detection |
|---|---|---|
| C1 | **Duplicate or unstable ids.** | `validate.py` reports duplicates; also check that two runs give the same ids |
| C2 | **`embed_text` and `text` diverge** (the embedding text carries a context line). | Read 5 chunks: does the context line state the right entry, group and corpus? |
| C3 | **Stale embeddings** (chunks changed, vectors did not). | `python scripts/pipeline.py doctor` must show `missing/stale: 0`; the importer refuses stale embeddings |
| C4 | **Model/token limit mismatch.** Chunks longer than the model's limit are truncated when embedded. | Check the token limit stored per record in `embeddings.jsonl` against the longest chunk |
| C5 | **Retrieval looks fine on the examples but not on real questions.** | Write 20–40 queries from real developer questions with the expected entry, run `python scripts/eval_retrieval.py --compare`, and read the misses. Include exact names, German questions, notation members (`$search`), hash variables (`#ERRCODE`) |

---

## 4. Procedure

Work step by step. At the end of each step write down what you checked, the numbers, and the
samples that you compared by hand.

1. **Inputs.** List the files in `Omnis PDF/` with page count, bookmark count and the revision number.
   Note the doc pack path and its catalog counts (or that none is used).
2. **Fit check.** Go through the table in section 2 and report for each row whether it fits your
   files. Do this *before* the first run.
3. **extract.** Run `python scripts/extract.py`. Read the warnings it prints. Apply P1–P8 to
   `output/extracted/*.json` and the `output/*_extracted.md` files.
4. **chunk.** Run `python scripts/chunk.py`. Confirm that the first output line reports the doc pack
   you intended. Apply P9–P11 and H1–H8.
5. **validate.** Run `python scripts/validate.py`. Report which checks passed *and which could not
   fail for your input* (for example the bookmark check without bookmarks).
6. **Hand comparison.** Pick 10 PDF entries and (with a pack) 5 notation pages, spread over the
   document: first, last, longest, shortest, one with a table, one with code, one deprecated. Put PDF
   page, extracted text and chunk side by side.
7. **embed and import.** Run the remaining steps. Apply C1–C4.
8. **Retrieval.** Apply C5 with your own queries.
9. **Report** (format below).

### Report format

```text
Inputs:       <files, pages, bookmarks, revision, doc pack>
Fit check:    <rows of section 2 that do not fit>
Counts:       <entries per source, chunks per corpus, expected vs. actual>
Findings:     <ID> <severity> <what> — evidence: <page / chunk id / sample> — proposed fix
Unverified:   <what could not be checked and why>
Changes made: <script, reason, evidence, re-run result>
```

Severity: **high** = content missing or wrong (lost entries, scrambled code), **medium** = degraded
quality (damaged words, missing metadata), **low** = cosmetic.

Do not report "looks good" without numbers and samples. If a check cannot be performed, say so.

---

## 5. When a script has to change

- Make the smallest change that fixes the defect, in the place named in section 2.
- Prefer a rule that follows from the document structure (bookmarks, fonts, table positions) over a
  list of special cases. Special cases break on the next version.
- Add the check that would have caught the defect to `scripts/validate.py`.
- Re-run the step and `validate.py`, then re-compare the entries that motivated the change.
- Describe the change in your report.
