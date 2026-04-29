# AI News Intelligence Platform

This is an internal tool used in a consulting practice to produce weekly news briefings about specific companies. You give it a company name, a date range, a country, and a target language; it gives you back a curated newsletter that an analyst can send to a client without rewriting it.

The code lives in a private corporate repository. This README explains what the tool does and why it ended up built the way it is.

## The job

The starting point is always the same. A consultant needs a brief about, say, Iberdrola in Brazil over the last month, in English, by tomorrow. The expectation is somewhere between fifteen and thirty news items, deduplicated, organised into sections (M&A, regulation, products, leadership, ESG, and so on), with a short executive paragraph at the top and a forward-looking trends section at the bottom. The whole thing has to be delivered as both a styled HTML newsletter and a CSV that the analyst can pivot.

Done by hand, this is roughly half a day of reading. The platform brings it down to a few minutes of unattended runtime per company.

## Why generic tools were not enough

We tried the obvious shortcuts first. None of them held up.

A direct call to a news API returns far too much noise on entity matches. Iberdrola in the US press is mostly about Avangrid; in the UK, ScottishPower; in Brazil, Neoenergia. A naive "must contain Iberdrola" filter throws all of those away. A relaxed filter lets in every "Iberdrola Móvil" customer service complaint that happens to share a brand prefix. The platform handles this with a per-company list of accepted brand variants and a separate list of protagonist tokens to exclude.

LLM web search wrappers ignore geography. Asking a model for "news about company X in Spain" returns mostly Latin American press, because the model honours the language hint far more reliably than the country hint. Geography has to be enforced after the fetch, by reading where the news happened, not what language it is written in.

Semantic deduplication only works if you stop trusting the model to do it. Earlier versions of the pipeline asked the LLM to "decide which items are duplicates," which produced different answers on every run. The current version uses embeddings with three separately tuned thresholds: 0.82 for same-language pairs, 0.85 for cross-language pairs, and 0.78 for cross-language pairs that landed on the same day. Those numbers came out of a battery of test runs against hand-labelled clusters; they are not interchangeable.

## How the pipeline is structured

The pipeline alternates between deterministic Python and LLM calls, with the model confined to the steps where it actually adds value (drafting, summarising, judging meaning) and kept out of the steps where it drifts (date math, geography checks, exact-string matching).

Discovery is the first LLM call. It runs a web search scoped to a country and a date window, and returns a list of candidate URLs and snippets. From there, the pipeline applies a series of deterministic filters: dates outside the window are dropped, opinion pieces and press releases are dropped, exact URL duplicates are dropped, and entity collisions are resolved against the brand-variant list.

The surviving items go into the embeddings stage, which clusters them by meaning. A second, much more constrained LLM call walks the clusters and emits a structured decision per cluster: keep one item as the canonical version, drop the rest, optionally tag the cluster with an event label (one acquisition, one regulatory move, one product launch). Replacing a free-form "summarise these duplicates" prompt with this constrained shape is what stopped the phantom-drop bug that plagued earlier versions.

Each surviving item is then enriched: a normalised headline, a sixty-word summary, a category, a date, a source. Enrichment runs with a hard retry policy because partial enrichment is worse than no enrichment at all.

Finally, deterministic templating assembles the newsletter. The hero card carries the company name, the date range, and the company logo (served from a per-company logo endpoint). Below it sit the categorised sections, the trends panel (forward-looking, never retrospective), and a closing section called Opportunities that frames the news in terms of where the consulting practice could plausibly help. Footer text is hardcoded English, regardless of the body language.

## Things that surprised us along the way

A reasoning model with a generous output cap can still truncate silently if the reasoning tokens eat the cap. The aggregation step pins minimal reasoning effort and bumps the cap to sixteen thousand tokens; without those two settings, briefings would lose their last few items at random.

Aggressively simplifying an LLM filter prompt makes it less reliable, not more. Verbose, example-rich filter prompts beat terse ones in every battery of tests we ran. The deterministic surface around the prompt is the right place to optimise; the prompt itself wants to stay long.

Cross-language same-day deduplication is its own problem. Same-event coverage in two languages on the same day scores lower on cosine similarity than same-event coverage in one language on the same day. Treating those two cases with the same threshold either over-merges or under-merges. The third threshold exists to handle exactly that case.

## Output

The HTML newsletter ships as a self-contained file with the hero card, the categorised sections, the trends panel, the opportunities section, and the footer. The CSV carries seven columns per row: normalised headline, summary, source, date, category, language, URL. Four of those columns are cached in the database so re-runs are cheap; the other three are recomputed.

## Status

Phase one is in production. The deterministic pipeline, the semantic deduplication, the brand-variant handling, the CSV export, and the styled newsletter are all shipped and used weekly.

Phase two adds a vector store for cross-run trend comparison, so a consultant can ask "what changed about company X between this run and last month." It is currently waiting on infrastructure approval to deploy pgvector inside our network.

## About this repository

You will only find this README here. The code is not public.
