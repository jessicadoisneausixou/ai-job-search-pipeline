# ai-job-search-pipeline

AI-Powered Job Search Pipeline

An automated low-code pipeline that aggregates project-manager jobs across France, Germany, and the Netherlands, scores each for fit using an LLM, stores them in Postgres, and emails a daily ranked shortlist. Built with n8n, the Adzuna API, the Anthropic API, and Supabase.

What it does

A scheduled workflow queries Adzuna for jobs in three countries, normalises and deduplicates them, asks an LLM to score each 0–100 for fit (with a one-line reason), and upserts them into Supabase keyed on job URL. A second workflow reads the database, filters for high-fit roles, and emails a formatted digest.

Architecture

Fetch:  Schedule → 3× Adzuna requests → Split → Clean fields → Dedupe
        → LLM scoring → Parse → Postgres (upsert on URL)

Digest: Schedule → Postgres (select fit_score >= threshold) → Build HTML → Gmail

Key design decisions

Adzuna API over scraping LinkedIn — LinkedIn's ToS prohibit scraping and risk account bans; Adzuna is a sanctioned aggregator. Tradeoff: coverage is limited to Adzuna's network.
Dedupe + upsert on URL — url is a unique column, so re-runs never create duplicates.
Two separate workflows — the digest reads all accumulated data, not just today's fetch, and each is independently testable.
Structured LLM output — the prompt forces a JSON-only reply ({"score", "reason"}) that a Code node parses and merges back onto each job.

Difficulties solved

Supabase connection — the direct host is IPv6-only (unreachable from n8n Cloud); fixed by switching to the IPv4 Session pooler. Then hit a self-signed certificate error, resolved by allowing the encrypted connection without strict cert validation.
Data lost between nodes — the LLM node overwrites the item's fields, so job data vanished before Postgres. Fixed with a Code node that merges the model's reply with the original fields pulled from an earlier node.
Nested API fields — Adzuna returns company/location as objects; mapping the object instead of .display_name produced blobs.

Known limitations

Adzuna's API is for building/testing and display, not ongoing aggregation without consent — this is a personal demo.
Two minor mapping bugs remain: Salary_min points at Adzuna's prediction flag, and company stores the object rather than company.display_name (scoring unaffected).
Built on the n8n Cloud trial, which caps executions; self-hosting is the long-term option.

Security: Adzuna keys sit in the request URLs as query params — always replace with placeholders before pushing. n8n stores other credentials separately from the workflow JSON.
