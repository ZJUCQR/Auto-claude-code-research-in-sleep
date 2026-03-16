---
name: paper-citation-refiner
description: "Generate and refine .bib files from existing LaTeX documents, review Related Work sections, and perform De-AI text polishing. Use when user says \"生成参考文献\", \"完善引用\", \"De-AI\", or \"check citations\"."
argument-hint: [tex-directory-or-main-file]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, WebSearch, WebFetch, mcp__codex__codex, mcp__codex__codex-reply
---

# Paper Citation Refiner: BibTeX Generation and Text Polish

Refine citations and text based on existing LaTeX source files in: **$ARGUMENTS**

## Constants

- **REVIEWER_MODEL = `gpt-5.4`** — Model used via Codex MCP for section review.
- **DBLP_BIBTEX = true** — Fetch real BibTeX from DBLP/CrossRef to eliminate hallucinated citations.

## Inputs

1. **Existing LaTeX files** — `.tex` files in the target directory (e.g., `main.tex`, `sections/*.tex`).
2. **Existing `.bib` file** — Optional. The current bibliography file, or the skill will create a new one.

## Workflow

### Step 1: Scan and Extract Citations

1. Scan all `.tex` files in the project for citation commands: `\cite{}`, `\citep{}`, `\citet{}`, `\citealp{}`, `\citealt{}`.
2. Extract all unique citation keys.
3. Handle multiple citations within a single command (e.g., `\citep{key1, key2}`).

### Step 2: Generate and Filter BibTeX

**CRITICAL: The final `.bib` file must ONLY contain papers that are actually cited.**

1. Parse the existing `.bib` file (if present). Keep only entries whose keys match the extracted set.
2. Scan all `\citep{}` and `\citet{}` references in the drafted sections
3. Build a citation key list
4. For each citation key:
   - Check existing `.bib` files in the project/narrative docs
   - If not found and **DBLP_BIBTEX = true**, use the verified fetch chain below
   - If not found and **DBLP_BIBTEX = false**, search arXiv/Scholar for correct BibTeX
   - **NEVER fabricate BibTeX entries** — mark unknown ones with `[VERIFY]` comment
5. Write `.bib` file containing ONLY cited entries (no bloat)

#### Verified BibTeX Fetch (when DBLP_BIBTEX = true)

Three-step fallback chain — zero install, zero auth, all real BibTeX:

**Step A: DBLP (best quality — full venue, pages, editors)**
```bash
# 1. Search by title + first author
curl -s "https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json&h=3"
# 2. Extract DBLP key from result (e.g., conf/nips/VaswaniSPUJGKP17)
# 3. Fetch real BibTeX
curl -s "https://dblp.org/rec/{key}.bib"
```

**Step B: CrossRef DOI (fallback — works for arXiv preprints)**
```bash
# If paper has a DOI or arXiv ID (arXiv DOI = 10.48550/arXiv.{id})
curl -sLH "Accept: application/x-bibtex" "https://doi.org/{doi}"
```

**Step C: Mark `[VERIFY]` (last resort)**
If both DBLP and CrossRef return nothing, mark the entry with `% [VERIFY]` comment. Do NOT fabricate.

**Why this matters:** LLM-generated BibTeX frequently hallucinates venue names, page numbers, or even co-authors. DBLP and CrossRef return publisher-verified metadata. Upstream skills (`/research-lit`, `/novelty-check`) may mention papers from LLM memory — this fetch chain is the gate that prevents hallucinated citations from entering the final `.bib`.

**Automated bib cleaning** — use this Python pattern to extract only cited entries:

```python
import re
# 1. Grep all \citep{...} and \citet{...} from all .tex files
# 2. Extract unique keys (handle multi-cite like \citep{a,b,c})
# 3. Parse the full .bib file, keep only entries whose key is in the cited set
# 4. Write the filtered bib
```

This prevents bib bloat (e.g., 948 lines → 215 lines in testing).

**Citation verification rules (from claude-scholar + Imbad0202):**
1. Every BibTeX entry must have: author, title, year, venue/journal
2. Prefer published venue versions over arXiv preprints (if published)
3. Use consistent key format: `{firstauthor}{year}{keyword}` (e.g., `ho2020denoising`)
4. Double-check year and venue for every entry
5. Remove duplicate entries (same paper with different keys)

### Step 3: De-AI Polish (去 AI 痕迹)

Scan all `.tex` files (especially abstract, introduction, and related work) and rewrite to eliminate AI writing signatures.

**Strictly Avoid (Lexical):**
- delve
- pivotal
- landscape
- tapestry
- underscore
- groundbreaking
- revolutionary
- paradigm shift

**Strictly Avoid (Templates/Phrasing):**
- "In this section, we..." (and variations)
- Significance inflation (overstating the impact of the work)
- Rule-of-three lists without substantive meaning
- Repetitive transitions (e.g., "It is worth noting that", "Importantly")

### Step 4: Model Review & Iteration (Related Work)

Send the all the files to the REVIEWER_MODEL for critical analysis:

```
mcp__codex__codex:
  model: gpt-5.4
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    Review all citations throughout the manuscript to verify their correctness and ensure that no hallucinated, fabricated, or incorrect references are included.

    Focus strictly on:
    1. Are the citations factually accurate and appropriately contextualized?
    2. Is the literature coverage sufficient for a top-tier venue? What key lines of work are missing?
    3. Are there any logical errors in how prior work is categorized or compared to the current method?
    4. Are there any remaining AI-generated stylistic tics (delve, pivotal, groundbreaking, etc.)?

    Provide specific critiques. If the coverage is insufficient or contains errors, output an action plan to iterate and expand the section.

	If the model identifies insufficient citations or logical errors, **iterate**: 
    1. Research the missing literature.
    2. Draft additions to the cite part.
    3. Add the new citation keys to the extraction list and repeat Step 2 to fetch their BibTeX.
    [paste full draft text]
```

### Step 5: Final Sanity Check

Before completing the task, aggressively verify the following constraints:

- [ ] **No `TODO` markers** exist in the `.tex` or `.bib` files.
- [ ] **No `VERIFY` markers** exist in the `.tex` or `.bib` files. All citations must be resolved to real entries.
- [ ] **Zero Redundancy**: `.bib` contains exactly the number of unique keys extracted from the text. No uncited entries.
- [ ] **No Stale Files**: Ensure no orphaned `.tex` files exist in the directory (files not `\input` or `\include` in `main.tex`).
- [ ] All citation keys compile cleanly (no undefined references).

## Key Rules

- **Never fabricate BibTeX**. If a citation cannot be found via DBLP/CrossRef/Scholar, alert the user rather than faking metadata.
- **Aggressive filtering**. Delete any existing `.bib` entry that is not actively cited in the current text.
- **Silent overwrites**. Once backed up, aggressively replace the old `.bib` and update `.tex` files with De-AI'd text. Do not ask for permission for each small word change during the De-AI phase.