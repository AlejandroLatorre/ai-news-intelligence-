# AI News Intelligence Platform

A production-grade intelligence platform that turns the open web into structured, deduplicated, geographically-scoped news briefings about a target company, sector, or topic — and packages them as ready-to-deliver newsletters.

The platform was built as part of an enterprise consulting toolkit; the source code is not public. This repository documents what the product does, the design choices behind it, and the engineering problems it solves.

---

## What it does

Given a company (or sector / topic), a date range, a country and an output language, the platform produces a curated news brief in a single run:

1. **Discovers** candidate news from the open web using LLM-driven web search, scoped to the requested geography and time window.
2. **Filters** the raw set deterministically: drops the wrong company (homonyms, brand collisions), the wrong country, the wrong dates, opinion pieces, press-release noise and duplicates.
3. **Deduplicates** semantically across languages — the same event reported in three languages collapses to one entry.
4. **Enriches** each surviving item with a normalized headline, summary, source, date and category.
5. **Clusters** items into events (one acquisition, one product launch, one regulatory move) so the brief reads as a story, not a feed.
6. **Generates** a newsletter: hero highlight, categorized sections, trends section, and a closing "opportunities" angle tailored to the consulting context.
7. **Exports** to CSV (analyst use) and to a styled HTML newsletter (delivery use).

The full pipeline runs unattended in a few minutes per company.

---

## Why it exists

Generic news APIs and LLM web-search wrappers fail in three ways that matter for client delivery:

- **Wrong entity.** "Iberdrola" matches Avangrid, ScottishPower and Neoenergia in the US/UK/Brazil press; a naive search drops them. A naive *strict* search misses sub-brands. The platform supports configurable brand variants per company.
- **Wrong geography / language mix.** A Spanish-language search for a Spanish multinational returns mostly Latin American press; a brief about *Spain* needs Spanish news that *affects Spain*, not news that happens to be in Spanish. Geo-scope is enforced post-retrieval, not via query tricks.
- **Same event, many sources.** Reuters, Bloomberg, El País and Handelsblatt cover the same earnings release. Without semantic dedup the brief reads like a feed reader. The platform clusters cross-source, cross-language reports of the same event into one entry.

The result is a brief an analyst can hand to a client without rewriting it.

---

## Architecture (functional view)

The pipeline is deliberately split into deterministic and probabilistic stages, with the LLM confined to the steps where it adds the most value (drafting, summarizing, deduping by meaning) and kept out of the steps where it drifts (date filtering, geography filtering, exact-string filtering).

| Stage | Nature | Responsibility |
|---|---|---|
| Discovery | LLM web search | Generate candidate URLs and snippets within a country + date window |
| Surface filters | Deterministic | Drop wrong-date, wrong-domain, opinion, duplicates by URL |
| Entity filter | Deterministic + LLM-assisted | Resolve homonyms, accept configured brand variants, exclude protagonist tokens |
| Geo filter | Deterministic | Verify the news *affects* the target country, not just that it is *written in* its language |
| Semantic dedup | Embeddings | Cluster items by meaning, cross-language threshold tuned separately from same-language |
| Event clustering | LLM (constrained) | Group items into events with `{keep, drop, event}` decisions |
| Enrichment | LLM | Normalized headline, 60-word summary, category, date, source |
| Brief assembly | Deterministic templating | Hero, categorized sections, trends, opportunities, footer |
| Export | Deterministic | CSV (4 cached columns + run-time fields) and styled HTML newsletter |

A second phase (in progress) adds a vector store for cross-run trend comparison ("what changed about company X between this run and last month").

---

## Engineering choices that mattered

- **LLM-as-aggregator is unreliable.** Prior versions used an LLM to "decide which items are duplicates" — it drifted erratically run to run. The current pipeline moves surface-pattern dedup into deterministic code and reserves embeddings + clustering for semantic dedup.
- **Reasoning budgets eat output budgets.** Aggregation calls on reasoning models silently truncate when reasoning tokens consume the response cap. The pipeline pins minimal reasoning effort (or raises the cap explicitly) for aggregation steps.
- **Prompts that shrink lose quality faster than prompts that grow.** Aggressive simplification of LLM-filter prompts produced erratic behavior; the codebase keeps the verbose, example-rich filter prompts and only optimizes the deterministic surface around them.
- **Cross-language same-day dedup needs its own threshold.** Same-event articles in two languages on the same day score lower on cosine similarity than same-language same-event articles. The pipeline uses three thresholds (same-language `0.82`, cross-language `0.85`, same-day cross-language `0.78`).
- **Geographic scope is a post-filter, not a query parameter.** The platform fetches broadly and rejects later, because LLM web-search providers honor language hints far more reliably than country hints.

---

## Output

- **Newsletter (HTML).** Hero card with company + date range + logo, categorized news sections, trends panel (forward-looking, not retrospective), opportunities section, EN-only footer.
- **CSV.** One row per news item with seven columns including normalized headline, summary, source, date, category, language, URL.
- **Per-company logo endpoint** for embedding in delivered briefs.

---

## Status

Active. Phase 1 (deterministic pipeline, semantic dedup, brand variants, CSV export, styled newsletter) is shipped to production. Phase 2 (cross-run trend vectors via pgvector) is pending infrastructure approval.

---

## What this repository contains

This README. The code lives in a private corporate repository and is not redistributable.
