---
name: daily-paper
description: Search, filter, summarize, deduplicate, and citation-track top-venue academic papers in a local paper-daily workspace. Use for daily paper discovery, citation updates, or citation-growth reports.
---

# Daily Paper

This skill manages a local academic-paper digest. It supports three modes selected from `$ARGUMENTS`:

- Search mode: a research query such as `LLM agents`, optionally with `--sort=date|relevance|citations` and `--count=N`.
- Citation update mode: `--update` only.
- Citation report mode: `--report` only.

If `--update` and `--report` are both supplied, stop and ask the user to choose one. Search mode requires a non-empty keyword after removing flags. Unknown flags, non-positive counts, and invalid sort values are errors. Optional `--no-push` skips the final Git push; `--dry-run` performs discovery/calculation without writing files or committing.

## Workspace

Resolve the workspace root from `PAPER_DAILY_ROOT` when set; otherwise use `D:\daily-papers`. Read and write only these paths beneath that root:

- `config.json`
- `data/papers.json`
- `data/venues.json`
- `papers/`
- `reports/`
- `assets/`

The bundled `scripts/render_html.py` writes HTML companions and navigation indexes beneath the same workspace root.

Read [references/data-contract.md](references/data-contract.md) before changing JSON. Parse each JSON file strictly before doing any work; if a file is malformed, stop, report its path and parse error, and do not initialize, overwrite, or partially repair it. Preserve unknown fields and existing records. Use an atomic temporary-file replacement for valid JSON writes so an interrupted run does not truncate the database.

## Practical execution profile

Use this order for every search so discovery, verification, and writing do not get mixed together:

1. Read all text and JSON files as UTF-8 explicitly. On Windows PowerShell use `-Encoding UTF8`; for validation, prefer Python `json.loads(Path(path).read_text(encoding='utf-8'))` or an equivalent strict parser. Never diagnose a JSON error from mojibake produced by the system code page.
2. Validate `config.json`, `data/venues.json`, and `data/papers.json` before making network requests. Build the canonical arXiv-ID and venue-alias maps once.
3. Discover a broad candidate set, then verify only a shortlist. Do not spend Semantic Scholar requests on every web-search hit.
4. Resolve formal publication evidence before ranking. Prefer a conference/journal page or DOI/Crossref record; use an arXiv record for the abstract and identifier, not as proof of acceptance by itself.
5. Write the topic digest after the qualifying set is fixed. A digest is an abstract-level search result, not a full-paper analysis.
6. Run the optional PDF full-text pass separately when deep reading is requested. Only then create one `papers/{arxiv_id}.md` analysis file per paper.

If fewer than the requested number of papers qualify, report the shortfall and the discarded/skipped candidates. Do not fill the quota with an unverified venue, a survey/tutorial when original research was requested, or an arXiv-only preprint.

## Shared rules

1. Use the configured `default_count`, `default_sort`, `venue_filter`, and `semantic_scholar_rate_limit`; defaults are 10, `relevance`, and 8 requests/second when fields are absent. Never exceed Semantic Scholar's documented limit of 10 requests/second.
2. Use the local date in `YYYY-MM-DD` (Asia/Shanghai when the host timezone is unavailable) for filenames, `first_seen`, and citation-history entries.
3. Canonicalize an arXiv identifier by removing `https://arxiv.org/(abs|pdf)/`, an optional `arXiv:` prefix, query/fragment text, and a trailing version suffix (`vN`) for duplicate comparison. Preserve the source identifier/version in the stored `arxiv_id` and URL.
4. A paper qualifies only when its resolved venue matches a venue or alias in `venues.json` and its rank is included in `config.json.venue_filter`. A preprint qualifies when reliable metadata or an explicit acceptance/publication statement links it to that venue; do not infer acceptance from topic similarity alone.
5. Never invent a venue, citation count, author, date, method, result, or conclusion. If an abstract/full text does not support a summary field, write `信息不足，未在可用摘要或正文中找到证据。` and retain the source URL.
6. De-duplicate search results against both the current candidate set and existing `papers.json` using the canonical arXiv ID. Existing papers are not printed as new search results, but their citation count may be refreshed when the source response includes it.
7. Respect API failures: retry 429 and transient 5xx responses with bounded exponential backoff, honor `Retry-After` when present, and report skipped IDs instead of fabricating data.
8. Record provenance for every fallback value. A successful Semantic Scholar response may supply citation metadata; if it remains unavailable after retries, use the fallback policy in `references/mode-procedures.md` and store `citation_source` with the retrieval date. Never silently mix sources.

## HTML output

Every non-dry-run output must have a matching HTML file with the same base name. Keep Markdown as the source of record and render it with the bundled `scripts/render_html.py` after writing or updating content. Use `--all` after a batch run to render every Markdown file under `papers/` and `reports/`; it also maintains `papers/index.html`, `reports/index.html`, and the workspace-level `index.html`. Use relative links for local documents so a user can open the HTML files directly from the filesystem. The renderer escapes paper content, supports headings, lists, tables, code blocks, links, and the Markdown math delimiters, and copies a pinned local KaTeX runtime and fonts under each output directory so formula display does not depend on a CDN or network access.

## Search mode

1. Read `config.json`, `data/papers.json`, and `data/venues.json`. Treat a missing `papers.json` as an empty database only in search mode; do not silently recreate missing configuration or venue data.
2. Run web searches with these queries, replacing `{keywords}` and using the current/recent years as appropriate:
   - `{keywords} site:arxiv.org NeurIPS ICML ICLR 2025 2026`
   - `{keywords} site:arxiv.org CVPR ACL EMNLP 2025 2026`
   - `{keywords} site:semanticscholar.org`
   Prefer official conference/journal pages, DOI/Crossref, arXiv records, and Semantic Scholar metadata. Extract candidate arXiv IDs, DOI/venue evidence, and the source URL rather than trusting search-result prose.
3. For the shortlist, query Semantic Scholar's paper endpoint (or its documented equivalent) with at least `title,authors,year,citationCount,venue,publicationDate,abstract,externalIds,url,openAccessPdf`. Use the arXiv ID when available. Space requests according to the configured rate limit; do not run repeated broad searches after a 429.
4. If Semantic Scholar returns 429/5xx after the bounded retries, continue metadata verification with arXiv (title/authors/date/abstract), Crossref or the official proceedings/DOI (formal venue/date/authors), and OpenAlex (citation fallback). Mark the exact source and retrieval date. A candidate is still discarded when formal venue evidence cannot be established.
5. Resolve the venue case-insensitively against names and aliases in `venues.json`, assign the matching rank, and discard non-qualifying papers. Normalize common forms such as `NeurIPS 2024`, `Advances in Neural Information Processing Systems 37`, and official DOI container titles to the configured venue. Use the formal publication date for the date window when available; retain the arXiv submission date separately. Exclude titles/abstracts that clearly identify a survey, review, tutorial, overview, or position paper when original research is requested.
6. Sort by the requested key: relevance uses search rank/semantic relevance, date uses formal publication date descending, and citations uses the verified citation count descending; break ties by title then canonical ID.
7. For the first `N` new papers, write a grounded Markdown summary with the exact structure in [references/mode-procedures.md](references/mode-procedures.md). Use the abstract, paper text, or official page for the contribution, method, experiments, and conclusion. Include source links, venue evidence, citation count, and any unavailable-evidence note.
8. Unless `--dry-run`, merge new records into `data/papers.json`. Store at least the schema fields in [references/data-contract.md](references/data-contract.md), initialize `citation_history` with today's count, and set `first_seen` only once. Save the digest as `papers/YYYY-MM-DD-{sanitized-keywords}.md`; replace path-invalid characters and trim the filename without changing the displayed research direction.
9. Render the digest immediately after saving it, then refresh `papers/index.html`. Do not report the search as complete until the Markdown and HTML paths both exist.

## Deep-reading pass

Search mode does not imply full-text analysis. When a full-paper reading pass is requested, download the official/open-access PDF for each selected paper, extract text, and create one `papers/{arxiv_id}.md` file with evidence-grounded sections for motivation, contribution, method, experiments, conclusion, limitations, and reproducibility. Store `analysis_source: "pdf_fulltext"` and `fulltext_chars` when the extraction succeeds. If PDF retrieval or extraction fails, keep the digest and record the skipped ID; do not invent details from the title.

After each successful deep-reading file is written, render its HTML companion. A failed PDF pass must not produce a placeholder HTML analysis.

In deep-reading Markdown, format mathematics consistently: inline formulas must use `$formula$`; standalone formulas must use a separate paragraph delimited by `$$` on its own lines. Do not emit `\\(...\\)` or `\\[...\\]`, and do not place a display formula inside a sentence. Preserve the paper's symbols and label any reconstruction or inference explicitly.

## Post-run checks

Before reporting completion, re-parse every changed JSON file as UTF-8, verify the digest section count equals the number of qualifying papers, verify canonical arXiv IDs are unique, and check that each selected record has a resolvable venue rank, date, source URL, and citation provenance. For every generated Markdown file, verify the same-base-name `.html` exists, the HTML contains the document title and major headings, and each local `.html` link target exists. Verify the indexes exist when their parent directories exist. Report API failures, discarded candidates, and any missing full-text analyses explicitly.

## Citation update mode

Read all records from `data/papers.json`, query Semantic Scholar once per unique canonical arXiv ID, and update `citation_count`/`last_updated` when metadata is returned. Append one `{date,count}` observation per paper per day; if today's observation already exists, replace its count rather than adding a duplicate. Keep going after individual failures and print a concise summary including the largest absolute increases and skipped IDs. Read [references/mode-procedures.md](references/mode-procedures.md) for the update and retry details.

## Citation report mode

Read `data/papers.json` without changing citation history. For each paper, calculate current count, 7-day growth, and 30-day growth against the latest observation on or before each cutoff (not against the earliest record and not by summing duplicated same-day rows). Generate `reports/YYYY-MM-citation-report.md` using the tables and sections in [references/mode-procedures.md](references/mode-procedures.md), including growth rate only when a non-zero baseline exists. Group topic statistics by normalized keyword and preserve the paper's venue and source link. Save the report as Markdown and render the matching `reports/YYYY-MM-citation-report.html`; refresh `reports/index.html` in the same run.

## Git handoff

After a non-dry run, show the files changed, including HTML companions and index updates, and the proposed commit message `Daily papers: {keywords}` or `Citation update/report: YYYY-MM`. If the workspace is a Git repository and the user has not supplied `--no-push`, run `git add` for only the generated/updated files, commit, then push the current branch. If there is no repository, no upstream, or a commit/push fails, retain the files and report the exact Git error; never reset or discard unrelated changes.
