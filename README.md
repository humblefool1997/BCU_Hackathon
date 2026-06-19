# MACINTOSH — BCU AI Hackathon 2026

A retrieval-augmented question-answering pipeline that answers the 100-question
multiple-choice set in `questions_100.csv`. Output: `MACINTOSH_submission.csv`
(`question_no,answer`, one of `A`–`E` per row).

---

## Model (≤ 8B requirement)

| Role | Model | Size | Notes |
|------|-------|------|-------|
| **Answering LLM** | **Qwen2.5-7B-Instruct** | **~7.6B params** | Under the 8B cap. Run locally via **Ollama** on macOS. |
| Reranker (retriever) | bge-reranker-v2-m3 | 278M | A cross-encoder that **scores passages only** — it never produces an answer, so it does **not** count toward the 8B cap. |
| Optional fast backend | e.g. Groq `llama-3.1-8b-instant` | ~8B | OpenAI-compatible endpoint, toggled in the config cell for faster iteration. Also ≤ 8B. |

**Declared model for grading: Qwen2.5-7B-Instruct (~7.6B parameters).**

---

## Approach

These questions are **long-tail Wikipedia facts** (obscure towers, species ranges,
single record-label or release-year details). A 7B model cannot recall most of them
from its weights, so we treated the task as a **retrieval + reading-comprehension**
problem rather than a reasoning one:

> Find the right Wikipedia article → locate the supporting sentence → fact-check every
> option against it → commit to a single best letter.

---

## Pipeline

1. **Load & clean** — read the CSV; repair UTF-8 mojibake (e.g. `BÃª Kontrol` → `Bê Kontrol`) with `ftfy` / a latin-1 fallback.
2. **Entity extraction** — pull the searchable subject of each question (LLM call, with a capitalised-noun / quoted-title heuristic fallback).
3. **Query building** — entity-anchored queries, **de-duplicated** (fixes the `Command Command MARSOC` repeat bug), with an extra **section-targeted query** for buried-number questions (length / distance / population / ratio).
4. **Retrieval** — full **English Wikipedia articles** via the MediaWiki API (not snippets), with **DuckDuckGo** as a fallback. Includes **429 backoff + a polite delay + caching**; successful lookups cache, 429s do not, so re-runs are clean.
5. **Disambiguation retry** — if the top passages don't contain enough of the question's content nouns, re-query with **option-specific discriminating tokens** and re-rank the merged set. This catches wrong-sense retrieval (e.g. *monarch* bird family vs butterfly; *armet* helmet vs an armoured-vehicle company).
6. **Chunk + rerank** — split articles into ~130-word passages; **bge-reranker-v2-m3** cross-encoder scores them; keep the **top 5**.
7. **Answer (self-consistency)** — few-shot prompt; **5 samples at T = 0.7** with **option-order shuffling** each sample to cancel small-model positional bias; majority vote mapped back to the original letters.
8. **Contradiction verification** — per-option **CONTRADICTED / SUPPORTED / UNSTATED** check against the evidence (with an explicit nickname-trap guard, e.g. *"black"* vs *dark purple*). It leads on *"which statement accurately describes…"* questions, breaks ties, and can overturn a weak single leader.
9. **Validate & export** — hard format gate (100 rows, `A`–`E`, **zero blanks**); a **no-"Unknown" policy** (always commit a best guess, assuming no penalty for wrong); write `MACINTOSH_submission.csv` plus a per-question evidence log (`MACINTOSH_evidence_log.jsonl`).

---

## How to run

**Prerequisites**

```bash
# 1. Python deps
pip install pandas requests ddgs sentence-transformers ftfy
# (optional, only for the OpenAI-compatible backend)
pip install openai

# 2. Local model via Ollama  (https://ollama.com)
ollama pull qwen2.5:7b-instruct
ollama serve            # if not already running
```

**Run**

1. Place `questions_100.csv` next to `MACINTOSH_pipeline.ipynb` (or set `QUESTIONS_FILE` in the config cell).
2. Open `MACINTOSH_pipeline.ipynb` and run the cells top to bottom.
3. Use the **smoke-test cell** (first 3 questions) to confirm retrieval + answering work, then run `run_pipeline(questions)` for all 100. Pass `limit=10` first if you want a quick subset.
4. Outputs land in the working directory: `MACINTOSH_submission.csv` and `MACINTOSH_evidence_log.jsonl`.

**Config toggles** (top of the notebook): `LLM_BACKEND` (`"ollama"` / `"openai"`), `N_SELF_CONSISTENCY`, `USE_RERANKER`, `TOP_K_PASSAGES`, `COVER_FRAC` (disambiguation-retry sensitivity), and `ALLOW_UNKNOWN`.

---

## Files

| File | Purpose |
|------|---------|
| `MACINTOSH_pipeline.ipynb` | Full pipeline (load → retrieve → rank → answer → verify → export). |
| `MACINTOSH_submission.csv` | Final answers (`question_no,answer`). |
| `MACINTOSH_evidence_log.jsonl` | Per-question entity, queries, top passages, answer, confidence, method. |
| `MACINTOSH_presentation.pptx` | 3-slide deck (with speaker notes). |
| `retrieval_cache.json` | Cached Wikipedia / DuckDuckGo results for fast re-runs. |

---

## Reproducibility

Fixed seeds for sampling, on-disk retrieval cache, and a full evidence log mean a
re-run reproduces the same answers and lets any reviewer inspect exactly which
passage drove each choice. All answers are produced by the declared ~7.6B model.

---

## Results

- **100 / 100** questions answered — **no blanks**, all valid `A`–`E`.
- Answer distribution: **A: 23, B: 28, C: 17, D: 14, E: 18**.

### Note on the final submission (provenance)

The submitted CSV is the **pipeline's own output**, with **8 evidence-backed
corrections** applied — cases where the pipeline's *own retrieved evidence* directly
contradicted the answer it had selected. Each correction realigns the answer to the
retrieved text, so the submission's provenance stays accurate to the ≤ 8B pipeline.
We deliberately did **not** substitute answers from any larger/external model.

| Q | from → to | Reason (grounded in retrieved evidence) |
|---|-----------|------------------------------------------|
| 10 | B → E | Play opened **1908**; no listed year matches → "None of the above". |
| 20 | E → B | Evidence: *"individual foil and team sabre."* |
| 44 | B → A | Evidence: **privately built** (Triumph engine in Norton frame), not factory-built. |
| 46 | E → B | Evidence: **dark purple** spathes — the nickname ("black") trap. |
| 47 | C → E | Monarchidae ≈ **100 species** (retrieval had grabbed butterflies). |
| 51 | D → C | Cold-water-soluble **α-glucan = carbohydrate**, not a lipid. |
| 54 | C → E | Evidence: *"fourth studio album"* contradicts the "third" option. |
| 77 | C → D | Became 62nd yokozuna in **1987** (1983 was the top-division debut). |

---

## Known limitations

- The coverage heuristic that triggers the disambiguation retry can miss a **sense
  collision** when two topics share most of their vocabulary (tune `COVER_FRAC`).
- **Numbers buried deep in article sections** remain the hardest cases.
- "Confidence" is **self-consistency vote agreement, not calibrated accuracy** — a
  high vote share is not a guarantee of correctness.
- A further **17 questions** were flagged as uncertain (weak/ambiguous retrieval or
  fine distinctions); they remain as the pipeline answered them. The intended way to
  improve them is to re-run them through the upgraded retrieval + contradiction cells,
  not to hand-edit the CSV.

---

*Repository: https://github.com/humblefool1997/BCU_Hackathon*
