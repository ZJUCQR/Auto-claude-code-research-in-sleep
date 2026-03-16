---
name: paper-citation-adder
description: "Analyze existing LaTeX documents, identify claims requiring references, insert appropriate citation commands (e.g., \\cite{}), and generate a complete, verified .bib file in a single pass. Use when user says \"添加参考文献\", \"增加cite\", or \"generate citations\"."
argument-hint: [tex-directory-or-main-file]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch
---

# Paper Citation Adder: Inline Citation Insertion and BibTeX Generation

Analyze existing LaTeX source files in **$ARGUMENTS**, insert missing citations for factual claims, and generate a corresponding `.bib` file in a single pass.

## Constants

- **DBLP_BIBTEX = true** — Fetch real BibTeX from DBLP/CrossRef to eliminate hallucinated citations.

## Inputs

1. **Existing LaTeX files** — `.tex` files in the target directory (e.g., `main.tex`, `sec/*.tex`).

## Workflow

### Step 1: Analyze Text and Insert Citations

1. Read all `.tex` files in the project.
2. Identify sentences, claims, or technical statements that require academic referencing.
3. Use `WebSearch` / `WebFetch` (e.g., Google Scholar, arXiv) to find the most relevant, authoritative, and up-to-date papers supporting these claims.
4. Insert citation commands (e.g., `\cite{firstauthor2024keyword}`) directly into the `.tex` files at the appropriate locations. 
5. **Strict Rule:** Do NOT alter the user's original writing, vocabulary, or sentence structure. Your only job is to insert citation commands.

### Step 2: Generate and Filter BibTeX

**CRITICAL: The final `.bib` file must ONLY contain papers that are actually cited in the text.**

1. Compile a list of all citation keys you just inserted.
2. For each citation key, fetch the exact BibTeX metadata using the verified chain below:
   
   **Step A: DBLP (Primary - best quality)**
   
   ```bash
   # 1. Search by title + first author
   curl -s "[https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json&h=3](https://dblp.org/search/publ/api?q=TITLE+AUTHOR&format=json&h=3)"
   # 2. Extract DBLP key from result (e.g., conf/nips/VaswaniSPUJGKP17)
   # 3. Fetch real BibTeX
   curl -s "[https://dblp.org/rec/](https://dblp.org/rec/){key}.bib"

​	**Step B: CrossRef DOI (fallback — works for arXiv preprints)**

```bash
# Search CrossRef by article title and first author (if available) to obtain DOI
# Then fetch the BibTeX entry via the DOI resolver
curl -sLH "Accept: application/x-bibtex" "https://doi.org/{doi}"
```

​	**Step C: Mark `[VERIFY]` (last resort)**
If both DBLP and CrossRef return nothing, mark the entry with `% [VERIFY]` comment. Do NOT fabricate.

**Step D: Unresolved citations**
- If a citation cannot be resolved via DBLP or CrossRef, insert a `TODO` comment next to the `\cite{}` in the `.tex` file indicating manual review is required.
- Example: `\cite{missing2024}` → `\cite{missing2024} % TODO: verify citation`.


1. **NEVER fabricate BibTeX entries**. If an entry cannot be verified via DBLP or CrossRef, find an alternative paper instead of faking metadata.
2. Write a new `.bib` file containing ONLY the verified entries.
3. Ensure the `.bib` file is properly linked in the main `.tex` file (e.g., `\bibliography{refs}`).

### Step 3: Final Sanity Check

Perform a single, strict check before completing the task:

- [ ] All newly inserted `\cite{}` commands have a matching, perfectly formatted entry in the `.bib` file.
- [ ] `.bib` contains exactly the number of unique keys present in the text. No uncited bloat.

## Key Rules

- **Aggressive Verification:** Rely entirely on DBLP/CrossRef for metadata to prevent hallucinated citations.