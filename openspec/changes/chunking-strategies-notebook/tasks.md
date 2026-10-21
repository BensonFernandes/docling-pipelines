# Tasks

## 1. Scaffold and setup [backend]

- [ ] 1.1 Confirm no `08_` notebook already exists in `examples/notebooks/`; if a conflict exists, choose the next available number and update the file name used in all subsequent tasks. Verify by listing `examples/notebooks/`.
- [ ] 1.2 Create `examples/notebooks/08_chunking_strategies.ipynb` with Cell Group 0: imports, `CONFIG` dict (corpus paths, chunk parameters, Ollama host, `FORCE_REEXTRACT` flag, retrieval queries), and `check_ollama()` availability function. Verify the file exists and the notebook opens in Jupyter without kernel errors.

## 2. Cell Group A — Ingest + Extract with parquet cache [backend]

- [ ] 2.1 Implement the parquet cache guard: if `./notebook_cache/chunking_strategies/extracted/` exists and `FORCE_REEXTRACT=False`, load from cache and print a skip message; otherwise run the `ingest → extract` flow against the three corpus files (`docs/operators/functional/chunker_readme.md`, `docs/guides/FLOW_CONFIGURATION_GUIDE.md`, `docs/guides/PYTHON_API_GUIDE.md`) with `storage: "in-memory"` and `force_ingest: True`. Verify by running the cell group twice — second run must print the skip message and produce the same row count as the first.
- [ ] 2.2 After extraction, save the resulting pyarrow table to parquet at `./notebook_cache/chunking_strategies/extracted/`. Add a note cell explaining what the cache contains and how to force re-extraction. Verify the parquet file exists after the first run.

## 3. Cell Group B — Simple and Hybrid chunking + structural metrics [backend]

- [ ] 3.1 Implement the `simple` chunking flow (chunk_size=1000, chunk_overlap=200) against the cached extracted table using `DocpipeFlowManager`. Print chunk count and a two-row excerpt (first and last chunk) for each corpus document. Verify by inspecting printed output — three documents should each show distinct chunk counts > 1.
- [ ] 3.2 Implement the `hybrid` chunking flow (chunk_size=512, chunk_overlap=50, docling_tokenizer="sentence-transformers/all-MiniLM-L6-v2") against the same cached extracted table. Print chunk count and excerpt as above. Verify analogously.
- [ ] 3.3 Compute and display the structural metrics comparison table across `simple` and `hybrid` (and `semantic` placeholder if Ollama unavailable): chunk count, mean chunk size (chars), std deviation of chunk sizes, approximate overlap ratio. Use `pandas.DataFrame` for display. Verify the table renders in Jupyter output with correct column headers and one row per strategy per document.
- [ ] 3.4 Add a side-by-side text cell showing the same passage (first 3 chunks of `chunker_readme.md`) from `simple` vs `hybrid`. Use `IPython.display.Markdown` for readability. Verify the output visibly shows structural differences (hybrid should preserve table rows as units where simple cuts across them).

## 4. Cell Group C — Semantic chunking [backend]

- [ ] 4.1 Implement the `semantic` chunking flow (breakpoint_threshold_type="percentile", breakpoint_threshold_amount=95.0, semantic_embeddings_model="nomic-embed-text") behind an `OLLAMA_AVAILABLE` guard. If Ollama is unavailable, print a clearly formatted skip message explaining which model to pull and skip all remaining cells in this group. Verify: with Ollama running, semantic chunks are produced; with Ollama stopped, the skip message prints and no error is raised.
- [ ] 4.2 Add semantic strategy to the structural metrics table from task 3.3 (or re-render the table with all three strategies when this cell group runs). Verify all three rows appear in the table output.

## 5. Cell Group D — Semantic coherence scores + qualitative retrieval [backend]

- [ ] 5.1 Implement `embed_chunks(chunks, model="nomic-embed-text")` helper that calls Ollama's embedding API and returns a numpy array of shape `(n_chunks, embedding_dim)`. Embed all chunks from all three strategies in a single pass (reuse embeddings across coherence and retrieval steps). Verify by printing embedding array shape — should be `(n_chunks, 768)` for `nomic-embed-text`.
- [ ] 5.2 Compute intra-chunk similarity: for each chunk, split on `'. '` into sentences, embed each sentence, compute mean pairwise cosine similarity. Average per strategy per document and display as a table alongside inter-chunk similarity (cosine similarity between adjacent chunk mean embeddings). Guard with `OLLAMA_AVAILABLE`. Verify the table shows a meaningful score difference between `hybrid` and `simple` on `chunker_readme.md` (hybrid expected higher intra-chunk score due to structural boundary preservation).
- [ ] 5.3 Implement the qualitative retrieval cell: for each of the 3 queries in `CONFIG`, embed the query, compute cosine similarity against all chunks from each strategy, and display a formatted table showing top-3 retrieved chunks per strategy side-by-side. Use `pandas` for the table layout. Guard with `OLLAMA_AVAILABLE`. Verify the output renders — the query targeting the parameter table (`"How do I configure chunk_size and chunk_overlap?"`) should show `hybrid` returning a more coherent chunk for `chunker_readme.md`.
- [ ] 5.4 Add a narrative summary cell (markdown) below the retrieval table that ties the retrieval results back to the coherence scores from task 5.2 — one sentence per strategy explaining why its retrieval result reflects its coherence score. This is the "punchline" cell for the blog post. Verify the cell contains substantive prose (not placeholder text).

## 6. Cell Group E — Strategy decision guide + README update [backend]

- [ ] 6.1 Add the strategy decision guide as a markdown cell: a table mapping document type (dense prose, structured docs with headers/tables, short FAQ/mixed) to recommended strategy with a one-line rationale per row. Add a config quick-reference cell showing the minimal JSON config for each strategy. Verify the markdown renders cleanly in Jupyter.
- [ ] 6.2 Update `examples/notebooks/README.md` to add `08_chunking_strategies.ipynb` to the "Available Notebooks" section following the same format as existing entries (heading, description, "What you'll learn" list, prerequisites). Verify the README entry is consistent with the notebook's actual prerequisites (Ollama with `nomic-embed-text` for full run; no Ollama for structural metrics only).

## 7. Validation [backend]

- [ ] 7.1 Before committing, audit every Python code cell for Ruff compliance: all imports used in their cell or consolidated in Cell Group 0; all `except` clauses catch a specific exception type; no unused variables; all Ollama-gated cells use the `if not OLLAMA_AVAILABLE: ... else: ...` pattern (no `raise SystemExit`, no `%%script false`). Verify by running `pre-commit run nbqa-ruff-check --files examples/notebooks/08_chunking_strategies.ipynb` and confirming exit 0.
- [ ] 7.2 Run `pre-commit run nbstripout --files examples/notebooks/08_chunking_strategies.ipynb` and confirm it exits 0 and the file has no cell outputs (check `"outputs": []` in the raw JSON). This is the state the file must be in when committed.
- [ ] 7.3 Run `pre-commit run check-json end-of-file-fixer trailing-whitespace --files examples/notebooks/08_chunking_strategies.ipynb` and confirm all three exit 0.
- [ ] 7.4 Execute the full notebook end-to-end with Ollama running (`Kernel → Restart & Run All`) and confirm: no cell raises an uncaught exception, all five cell groups produce visible output, the structural metrics table shows all three strategies, and the retrieval table renders for all three queries.
- [ ] 7.5 Execute Cell Groups 0, A, B, and E with Ollama stopped (or `OLLAMA_AVAILABLE = False` forced in Cell Group 0) and confirm: Cell Groups C and D print the skip message without raising an error, and all other cells complete successfully.

## Workflow follow-up

- Stage changes with `git add examples/notebooks/08_chunking_strategies.ipynb examples/notebooks/README.md` and commit with `git commit -s`.
- Run `pre-commit run --from-ref origin/main --to-ref HEAD` and fix any remaining hook failures before pushing.
- Archive the change after review: `/opsx-archive`.
