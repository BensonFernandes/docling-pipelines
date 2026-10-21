# Design

## Context

The existing notebooks establish clear conventions: a top-level `CONFIG` dict for all tuneable parameters, a `check_services()` function for graceful service-availability detection, a parquet caching pattern to avoid re-running expensive steps, and `DocpipeFlowManager` as the programmatic entry point. This notebook follows all of those conventions.

Three decisions from the exploration phase shape the approach here and are not restated in the proposal:

1. **Full pipeline (Option A)**: `ingest → extract → chunk` runs end-to-end. Bypassing extract would deprive the `hybrid` strategy of Docling's parsed document structure — the very thing that makes it distinct from `simple`.
2. **`docs/` as corpus**: Three real markdown files from the project's own documentation serve as the corpus. This is self-referential in a way that's compelling for a blog post and avoids synthetic data that would make outputs uninteresting.
3. **Qualitative retrieval signal, not labeled precision/recall**: No ground-truth annotations are required. Cosine similarity ranking + side-by-side display is sufficient to expose strategy differences and avoids the annotation maintenance burden that labeled metrics would introduce.

## Goals / Non-Goals

**Goals:**
- Define the notebook's cell group structure and data flow so implementation is mechanical
- Specify the three corpus documents and the rationale for their selection
- Define the three targeted retrieval queries and the structural property each is designed to expose
- Specify the Ollama availability check pattern and which cell groups it gates
- Specify the parquet cache location and re-run guard logic
- Specify how intra/inter-chunk coherence scores are computed and how they connect to the retrieval section

**Non-Goals:**
- Does not design new operators or modify `ChunkerOperator`
- Does not design a benchmark harness — outputs are for human reading, not automated regression

## Decisions

### D1: Corpus — three docs files, chosen for structural contrast

| File | Size | Structure | Strategy it highlights |
|---|---|---|---|
| `docs/operators/functional/chunker_readme.md` | ~8KB | Headers + parameter tables + JSON blocks | `hybrid` — tables and JSON blocks are structural nodes Docling preserves as units |
| `docs/guides/FLOW_CONFIGURATION_GUIDE.md` | ~21KB | Dense prose + code examples | `semantic` — long narrative paragraphs with topic shifts at paragraph boundaries |
| `docs/guides/PYTHON_API_GUIDE.md` | ~20KB | Mixed: procedures + code + prose | Baseline for all three — none has a clear advantage, good for comparison |

**Why not `extract_operator_readme.md` or `CUSTOM_OPERATORS_GUIDE.md`?** Both are 40-50KB — too many chunks to read side-by-side in a notebook. The chosen files are large enough to produce meaningful chunk counts (15–40 chunks per strategy) without overwhelming cell output.

**Why not synthetic documents?** Real project docs make the blog post self-referential ("I asked docling-pipelines about its own chunking documentation") and produce outputs a reader can independently verify by looking at the source files.

### D2: Three retrieval queries, each targeting a known structural property

| Query | Targets | Expected winner |
|---|---|---|
| `"How do I configure chunk_size and chunk_overlap?"` | A parameter table in `chunker_readme.md` | `hybrid` — table rows are preserved as discrete chunks |
| `"What is the difference between in-memory and filesystem storage?"` | A conceptual paragraph mid-way through `FLOW_CONFIGURATION_GUIDE.md` | `semantic` — finds the paragraph boundary; `simple` may cut mid-sentence |
| `"How do I use DocpipeFlowManager programmatically?"` | A procedure section in `PYTHON_API_GUIDE.md` | All three should surface it; useful as a baseline |

These queries are baked into the `CONFIG` dict so they are visible at a glance and easy to modify.

### D3: Parquet caching pattern — one cache dir, force_ingest guard

The extract step writes parquet to `./notebook_cache/chunking_strategies/extracted/`. A cell at the top of the chunking section checks for the cache:

```python
CACHE_DIR = Path("./notebook_cache/chunking_strategies/extracted")
FORCE_REEXTRACT = False  # set True to re-run extraction

if CACHE_DIR.exists() and not FORCE_REEXTRACT:
    print("Using cached extraction — set FORCE_REEXTRACT=True to re-run")
else:
    # run ingest + extract flow
```

This mirrors the pattern in `06_rag_pipeline.ipynb`. The `FORCE_REEXTRACT` flag is in `CONFIG` so it's visible alongside other tuneable parameters. Cache dir is `.gitignore`-listed (already covered by the existing `notebook_cache/` entry if present; otherwise noted in the notebook's setup cell).

### D4: Ollama availability check gates two cell groups

`check_ollama()` makes a lightweight `GET /api/tags` request to `http://localhost:11434`. The result sets a `OLLAMA_AVAILABLE` boolean that gates:

- **Cell group C**: `semantic` chunking flow (requires Ollama embeddings during chunking)
- **Cell group D**: semantic coherence metrics + retrieval signal (both use `nomic-embed-text`)

Cell groups A (ingest+extract) and B (simple + hybrid chunking + structural metrics) always run regardless of Ollama availability. This means the notebook is fully runnable without Ollama — readers get the structural comparison and side-by-side text excerpts even in a no-Ollama environment.

The Ollama model used for coherence and retrieval is `nomic-embed-text` — the same model already used in `04_embeddings_vectordb.ipynb` and `06_rag_pipeline.ipynb`, so no new `ollama pull` instruction is needed beyond what existing notebooks already document.

### D5: Embedding model — `nomic-embed-text` for consistency

The embedding model for semantic chunking, coherence scoring, and retrieval is `nomic-embed-text` (768 dimensions). Higher-quality alternatives exist (`mxbai-embed-large`, `granite-embedding`) but `nomic-embed-text` is already the standard across `04_embeddings_vectordb.ipynb` and `06_rag_pipeline.ipynb`. A reader who has run any existing notebook already has it pulled — adding a new model pull to prerequisites raises the barrier to entry for a notebook whose primary purpose is comparison, not embedding quality benchmarking. The coherence and retrieval differences between strategies are structural and large enough to be visible at `nomic-embed-text` quality.

### D6: Coherence score computation — pure numpy, no new dependencies

Intra-chunk similarity for a single chunk is computed as the mean pairwise cosine similarity of sentence-level embeddings within that chunk. Inter-chunk similarity for adjacent chunk pairs is their direct cosine similarity.

```
intra_sim(chunk)  = mean(cosine(embed(s_i), embed(s_j)) for all sentence pairs i≠j)
inter_sim(c_i, c_{i+1}) = cosine(mean_embed(c_i), mean_embed(c_{i+1}))
```

Sentence splitting uses Python's `str.split('. ')` — deliberately simple. The goal is a readable score, not linguistic precision. A markdown-aware sentence splitter would add a dependency and complexity without meaningfully changing the comparison story.

All embedding and cosine similarity work uses `numpy` (available in the dev environment via `uv sync --extra notebooks`). No new packages are introduced.

### D7: Cell group structure

```
Cell Group 0: Setup + imports + CONFIG dict + check_ollama()
Cell Group A: Ingest + Extract flow  [cache check → run if needed → save parquet]
Cell Group B: Simple chunking + Hybrid chunking + Structural metrics table
              [always runs; no Ollama required]
Cell Group C: Semantic chunking  [skipped with message if OLLAMA_AVAILABLE=False]
Cell Group D: Semantic coherence scores (intra + inter) for all three strategies
              Qualitative retrieval: 3 queries × 3 strategies → top-3 chunks side-by-side
              [skipped with message if OLLAMA_AVAILABLE=False]
Cell Group E: Strategy decision guide table + config quick-reference
              [always runs; pure markdown]
```

The decision guide table in Cell Group E is the "take-home" section — it maps document type to recommended strategy and is the most directly useful output for the blog post's audience.

### D8: Pre-commit compliance for notebook code cells

Four pre-commit hooks run on `.ipynb` files:

- **`check-json`**: The notebook file must be valid JSON. Generated automatically by Jupyter — no special action needed.
- **`nbstripout`**: Strips all cell outputs before staging. Workflow: run all cells, then `git add` (hook strips outputs), then `git commit -s`. Never commit with outputs present.
- **`nbqa-ruff-check --nbqa-dont-skip-bad-cells`**: Ruff lints every Python code cell, including cells that would raise at runtime. Implications:
  - All imports must be used in the cell they appear in, or must live in the shared setup cell (Cell Group 0)
  - No bare `except:` clauses — always catch a specific exception type
  - No unused variables (assign results to `_` if intentionally unused)
  - The graceful-skip pattern for Ollama-unavailable cells must use `if/else` blocks, not `raise SystemExit` or magic commands that bypass linting
- **`check-added-large-files` (3000KB limit)**: The stripped notebook (no outputs) must stay under 3MB. At ~20–40 chunks per strategy × 3 strategies, the notebook JSON will be well under this limit.

The canonical Ollama-skip pattern (Ruff-safe):
```python
if not OLLAMA_AVAILABLE:
    print("Skipping — Ollama unavailable. Run: ollama pull nomic-embed-text")
else:
    # actual cell logic here
```

### D9: Notebook numbering — `08_`

Existing notebooks are `01` through `07`. The new file is `08_chunking_strategies.ipynb`. If a future notebook is added at `08` before this is merged, the number should be bumped — noted as a minor merge-time check in tasks.

## Risks / Trade-offs

| Risk | Mitigation |
|---|---|
| `hybrid` chunking on `.md` files produces same output as `simple` if Docling's markdown parser treats all content as flat text | Verify during implementation by printing chunk metadata; `chunker_readme.md` has explicit tables that should parse as structured nodes. If hybrid and simple produce identical chunks on all three docs, replace one corpus file with a richer structured doc (e.g., an operator readme with more table density). |
| Intra-chunk similarity is low across all strategies because `docs/` prose is already fairly homogeneous | The comparison is still valid — the point is relative difference, not absolute score. Even if all three score 0.7+, hybrid should consistently score higher on `chunker_readme.md` than simple. If scores are indistinguishable, increase corpus size or switch to a more diverse doc. |
| `nomic-embed-text` embedding calls are slow for large chunk sets (30–40 chunks × 3 strategies = 90–120 embed calls) | Each embed call is ~50–100ms on a warm Ollama instance — total ~5–12 seconds. Acceptable for a notebook. If slow, add a progress indicator. |
| Notebook numbering conflict at merge time | Tasks include a check: confirm no `08_` notebook was added since this change was branched. |

## Open Questions

None — all decisions required before implementation are resolved above.
