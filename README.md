# Euromnis-OmnisDocRAG-example

Local RAG stack for the Omnis Studio documentation — extracts the PDF manuals (and optionally the
notation reference from the Omnis help), chunks and embeds them, stores them in PostgreSQL with
`pgvector`, and exposes semantic and full-text search as MCP tools for AI-assisted Omnis development.

---

## Quick orientation

Start here: [Project_instructions.md](Project_instructions.md)

It covers the project goal, directory layout, setup steps, runtime startup order, and the available
topologies for local and Docker-based operation.

---

## Bring your own documentation

This repository shows **one way** to build a RAG server from the Omnis documentation, so that a developer
with no RAG experience can see the whole path and build a first running MVP. It is an example, not a
product; other approaches are possible, and the scripts may need adapting to your documents. **The repository contains
no Omnis documentation and nothing generated from it.** The manuals and the help are © Omnis Software
Ltd. You provide them from your own Omnis installation:

- the PDF manuals `CommandRef.pdf`, `FunctionRef.pdf` and `Programming_Omnis.pdf` in `Omnis PDF/`
- optionally a doc pack made from the HTML help of your installation (`OMNISDOC_PACK`)

The step-by-step guide, the problems that occur when extracting these documents and the checks to
run after each step: [Documentation/Building_from_Omnis_Docs_en.md](Documentation/Building_from_Omnis_Docs_en.md).
Have the result validated, preferably by an AI agent: [Documentation/AI_Review_Guide_en.md](Documentation/AI_Review_Guide_en.md)
is written for the agent and describes the problems and what to check.
`output/` (extracted text, chunks, embeddings) is derived from the documentation and is git-ignored.
Do not commit or publish it.

---

## Documentation index

| Document | What it covers |
|---|---|
| [Project_instructions.md](Project_instructions.md) | Entry point — setup, startup order, quick-start commands |
| [Documentation/Building_from_Omnis_Docs_en.md](Documentation/Building_from_Omnis_Docs_en.md) | **Start here for the data:** providing the PDFs and HTML help, extraction pitfalls, checklist |
| [Documentation/AI_Review_Guide_en.md](Documentation/AI_Review_Guide_en.md) | **For an AI agent:** the problems, where the scripts must be adapted, what to check, report format |
| [Documentation/RAG_concept_en.md](Documentation/RAG_concept_en.md) | Why RAG, how retrieval works, architecture overview |
| [Documentation/Pipeline_en.md](Documentation/Pipeline_en.md) | Data build pipeline, platforms, **runbook: new embeddings on a Mac → server** |
| [Documentation/chunking_concept_en.md](Documentation/chunking_concept_en.md) | How the PDFs are extracted and chunked, and why |
| [Documentation/embedding_concept_en.md](Documentation/embedding_concept_en.md) | Embedding model, dimensions, and storage |
| [Documentation/postgres_en.md](Documentation/postgres_en.md) | Database schema, full-text vector, `rag.search_ranked` |
| [Documentation/expected_outcome_en.md](Documentation/expected_outcome_en.md) | What a working system looks like end-to-end |
| [Documentation/retrieval_quality_analysis_en.md](Documentation/retrieval_quality_analysis_en.md) | Retrieval quality analysis, measurements before/after the rework |
| [OmnisRAGServer/README.md](OmnisRAGServer/README.md) | Local stdio MCP bridge contract and tool reference |
| [docker_mcp-rag/README.md](docker_mcp-rag/README.md) | Docker stack — `mcp-server` + `rag-server`, using PostgreSQL on the host |
| [docker_mcp-rag-pg/README.md](docker_mcp-rag-pg/README.md) | Full Docker stack — PostgreSQL 18 + `pgvector` + `rag-server` + `mcp-server`; **deployment on an internal Docker server** |

---

## Three runtime topologies

**Local** — Python RAG server + Node.js stdio MCP bridge:

```
VS Code (stdio) → mcp-bridge (Node.js) → rag-server (Python) → PostgreSQL
```

**Docker with host PostgreSQL** — containerised MCP/RAG runtime, DB remains outside Docker:

```
VS Code (HTTP) → mcp-server (Python, port 3000) → rag-server (Python, port 7071) → PostgreSQL
```

**Full Docker stack** — PostgreSQL 18 + `pgvector` inside Docker:

```
VS Code (HTTP) → mcp-server (Python, port 3000) → rag-server (Python, port 7071) → postgres (Docker, PG18 + pgvector)
```

---

## What is indexed

| Corpus | Source |
|---|---|
| `omnis-commands` | Command Reference (every command, overview sections, error code tables) |
| `omnis-functions` | Function Reference |
| `omnis-programming` | Programming manual, all chapters |
| `omnis-notation` | Notation reference from the HTML help (optional doc pack) |

## MCP tools

| Tool | Use for |
|---|---|
| `search_omnis_docs` | Ranked search over everything. `mode`: `hybrid` (default), `semantic`, `fulltext` (supports `"phrase"`, `OR`, `-word`) |
| `get_omnis_doc` | Full text of result ids (second step) |
| `search_omnis_syntax` | Same search, only commands, functions, notation |
| `search_omnis_concepts` | Same search, only the Programming manual |

Search answers are compact text (one line + snippet per hit, typically 2–4 KB); the full text is
fetched only for the ids that matter.

---

## Minimum quick start (Full Docker stack)

Works the same on macOS, Windows and Linux (`python3` / `py` instead of `python` where needed).

```bash
# 1. Set up and check the machine
python scripts/pipeline.py setup
python scripts/pipeline.py doctor        # shows whether embedding runs on GPU/Apple Silicon (minutes) or CPU (hours)

# 2. Start the stack
cd docker_mcp-rag-pg
cp .env.example .env   # then edit DB credentials
docker compose up -d --build
cd ..

# 3. Build, import and check (optional doc pack: OMNISDOC_PACK in scripts/.env.local)
python scripts/pipeline.py build

# 4. VS Code — .vscode/mcp.json
# { "omnis-rag-docker": { "type": "http", "url": "http://localhost:3000/mcp" } }
```

If you want Docker only for `mcp-server` and `rag-server` while keeping PostgreSQL on the host,
use [docker_mcp-rag/README.md](docker_mcp-rag/README.md) instead.

---

## License

The code and the documentation of this repository are released under the [MIT License](LICENSE).
This does **not** cover the Omnis manuals or the Omnis help, which are © Omnis Software Ltd and are not
part of this repository.
