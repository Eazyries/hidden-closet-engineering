# Hidden Closet: Building a Safety-First AI Catalog Pipeline

## The real problem

Hidden Closet is a catalog system for a resale business: products come in from multiple supplier feeds (referred to here as Supplier A and Supplier B), get enriched with AI-generated captions, and get published to a Telegram channel for customers to buy from. An admin operates the whole thing through a Telegram bot.

The obvious way to describe this is "a Telegram bot with OpenAI captions." That description misses the actual engineering problem. The interesting risk in this system was never uptime or latency — it was publishing a product under the wrong brand.

Supplier feeds contain internal factory, folder, and season metadata that must never leak into customer-facing captions. An AI model asked to write a caption from a photo and a supplier title has every incentive to hallucinate a plausible-sounding luxury model name, or to echo an internal supplier tag straight into the brand line. Either failure mode is a real, external, credibility-destroying mistake, not a bug that just gets logged and retried. That constraint shaped almost every architectural decision below.

## Architecture: five independent controls before anything reaches a customer

```
Supplier feed (multi-source)
      │
      ▼
Title parser           — extracts cleaned_title, model_code, size,
      │                  and forbidden_tokens from the raw supplier title
      ▼
Brand resolver         — the catalog database's stored brand wins over
      │                  any supplier tag; internal factory-coded tags
      │                  never resolve to a literal brand override
      ▼
Routing + payload sanitation
      │                  — the AI call is skipped entirely when forbidden
      │                  tokens are present; when it does run, it receives
      │                  a pre-sanitized payload with forbidden tokens
      │                  excluded before the request is built
      ▼
OpenAI analysis
      │
      ▼
Output validation      — rejects invented model names, forbidden tokens,
      │                  and brand mismatches; falls back to a
      │                  deterministic safe caption when validation fails
      ▼
Publish queue          — composite key per product; lifecycle from
      │                  queued to published
      ▼
Telegram channel
```

Telegram is the last step in this chain, not the center of it. It's the delivery mechanism. Everything that matters engineering-wise happens before a caption is allowed anywhere near a "Publish" button.

## The incident that shaped the design

This is the clearest example of the system actually being tested by reality. A catalog item (internal incident ID **#475302**) had its internal tag set to a folder label rather than a clean brand tag, and its supplier photo produced OCR/leet noise from a vision pass. The result: fragments of that noise and the folder label leaked into the customer-facing caption as if they were part of the brand or model name.

Root cause turned out to be two independent bugs stacked on top of each other:

1. A structural bug in the fast-path used for high-confidence supplier data: it built a fallback caption from raw brand/model fields instead of the sanitized caption path, so garbage model text could reach the output.
2. A gap in the gibberish filter: it didn't yet reject *short* OCR/leet residual strings — it only handled longer noise patterns.

Neither bug alone was catastrophic. Together, they let junk output slip past every earlier layer. The fix touched four different points in the pipeline, not one:

- **Data repair**: corrected the item's stored brand/tag/category directly.
- **Output-level suppression**: the brand formatter now runs incoming brand strings through a blocklist that catches model/folder/promo labels and returns an empty string rather than let them pass as a brand.
- **Pipeline-level classification**: the brand-resolution module gained an explicit "non-brand tag" classifier, so folder/tag labels are never eligible to become a brand hint in the first place.
- **A dedicated OCR/leet detector**: rejects short garbled strings while explicitly preserving legitimate short luxury abbreviations that look similar in length — the naive version of this filter would have thrown out real data along with the noise.

A permanent regression fixture was added to the test harness with explicit "must not contain" assertions, so this exact failure mode can't silently come back.

## When the safety net itself was lying

Early in building the test harness, it reported several "critical" safety regressions. Investigating them individually showed the harness itself had a bug: it was scanning the *entire* generated caption — including the raw supplier description text it echoes as a stopgap — for forbidden tokens. A forbidden token showing up inside an untouched supplier-description line isn't a safety failure; it's a style problem, because the brand line and structured fields were still clean.

The fix wasn't a patch to a handful of test cases — it was splitting one overloaded metric into two:

- **A hard safety gate** — did an internal supplier tag escape into the *structural* fields (brand, labels, code, order line)? This must always read zero.
- **A style/quality gate** — would we actually want to show this full caption to a customer? This is allowed to be non-zero while formatting work is in progress.

That distinction — "did we ship something dangerous" vs. "did we ship something we're not proud of" — is the difference between a metric that's technically green and a metric that's actually meaningful. Conflating them either causes false alarms that erode trust in the safety gate, or worse, lets real regressions hide inside a noisy "expected" bucket.

## Catalog sync: idempotent imports without a staging environment

Supplier catalogs get re-imported over time as new source folders are added or refreshed. The sync layer handles this as branch-scoped, idempotent synchronization rather than a full destructive reload:

- Products are identified by a stable internal supplier ID, not by the customer-facing product code, because the code isn't guaranteed stable the way a supplier's internal ID is.
- Each sync run is tracked through an explicit session state machine (started → completed / failed / partial), backed by a dedicated sessions table.
- Unchanged rows are detected and skipped so partial re-imports don't generate noisy, meaningless writes.
- Branch-scoped imports deliberately skip the global "mark anything not seen in this run as inactive" step. Importing one source folder shouldn't be able to silently deactivate products from every other folder that simply weren't part of this run.

This is backed by a dedicated schema and its own test suite, part of a broader automated test suite covering the project overall.

## Developer tooling: inspecting a production system without a staging copy

There's no separate staging environment here — the only data that matters is the live catalog. To make changes to the AI pipeline safely, the project has a family of read-only inspection scripts that exercise real logic against real data without ever calling OpenAI, writing to a database, or importing the bot process itself: dry-running the exact payload that would be sent to the model, showing whether a product would take the fast path or go to the AI call and why, and running the validator/fallback logic end-to-end without touching the database or the delivery channel.

There's also a small static-analysis script that greps the live source for function definitions and generates an up-to-date line-number map of the entire pipeline — used internally to keep documentation honest against a large, actively-changing codebase.

The point of this tooling isn't cleverness — it's that a large production file with no staging environment needs some way to answer "what will this change actually do" before it ships, and building that tooling was treated as part of the job rather than optional polish.

## Engineering decisions behind the stack

None of these were defaults — each one was a call made against a specific constraint.

**Why SQLite, not Postgres.** One VPS, one bot process as the primary writer, no multi-region or multi-writer requirement. WAL mode, busy-timeout, and foreign keys are configured explicitly rather than reaching for a client-server database that would add an operational surface (auth, networking, backups) this system doesn't need yet.

**Why systemd, not Docker.** Single VPS, single deploy target, no need to schedule across hosts. Git pull → sanity check → service restart is the entire deploy path. Systemd already provides process supervision and restart-on-failure; containerizing a single-host service would add a build/registry step without buying anything this deployment target needs.

**Why Telegram as the admin interface, not a custom web panel.** The admin already lives in Telegram day-to-day. Building a bot meant zero custom auth, zero custom UI, and a mobile-friendly interface on day one — at the cost of the UI being whatever Telegram's message/keyboard model allows.

**Why the monolith, not services.** The core application is a large, growing single file. That's a real cost — but splitting it has its own cost: there isn't yet test coverage that would make a split safe, and there's one deployment target that doesn't need independent scaling of parts. The decision not to refactor without a stability plan and test coverage first is a sequencing decision, not an oversight.

**Why no separate staging environment.** *(Inferred from the tooling that exists, not an explicitly stated policy.)* There's no dedicated staging database or environment. Instead, changes to the AI pipeline are validated with read-only inspection scripts against real (snapshotted) data, and catalog imports go through production-gated runner scripts with a hard rule: on any failed command or unexpected condition, stop and report rather than self-fix. The read-only tooling is effectively standing in for a staging environment.

## Lessons learned

Building the safety layer for this pipeline, not just calling an API, was the actual work. A few things that came out of it directly:

- **Model output is a hostile input, not a trusted one.** The incident above happened because two different acceptable-looking outputs (OCR noise, a folder label) were treated as safe by default in one code path. Every AI output now goes through validation before it's allowed to reach a caption.
- **A deterministic fallback has to exist and has to be boring.** When validation fails, the system doesn't retry the AI or degrade gracefully into "probably fine" — it falls back to a fixed, minimal, safe caption. Boring and safe beats clever and occasionally wrong.
- **Test your safety metrics, not just your code.** The harness itself produced false positives before it produced real signal. A green dashboard that's measuring the wrong thing is worse than no dashboard, because it creates false confidence.
- **Tooling before features.** The read-only inspection scripts were built specifically so pipeline changes could be checked against real data without an AI call, a DB write, or a bot restart. That's infrastructure investment with no visible feature attached to it, and it's what made it possible to debug the incident above without guessing.
- **Say what's still broken.** The style-level gate is intentionally allowed to stay non-zero while the safety gate is not. Knowing the difference between "not done" and "not safe" is the entire point of the two-gate split.

## Trade-offs

The system intentionally remains a monolith. At its current size and with one deployment target, keeping the existing catalog/publish flow reliable has been worth more than the readability gained by splitting it apart — and a split done without test coverage would risk the exact kind of regression this whole safety layer exists to prevent. Refactoring is a planned step, gated on test coverage, not a rejected idea.

The same logic applies to the caption formatter: the hard safety gate shipped before the style gate did, on purpose. Shipping "safe but not pretty yet" was the correct order — shipping "pretty but occasionally wrong" would not have been.

None of this is hidden — it's tracked, prioritized, and sequenced on purpose. The system that exists today does the thing that actually mattered first: the system is designed to prevent products from being published under the wrong brand, with deterministic fallbacks when validation fails.
