# Add a "Rate Limits" column to the Models table

> Closes #204
>
> Status: **DRAFT — not yet opened as a public PR.** Local-only branch
> `feat/rate-limits-column-204` for review before sharing upstream.

---

## Summary

Surface **per-model rate limits** (request- and/or token-throughput caps over a
time window) directly in the public Models table so developers can compare
providers, not just by cost and context window but also by the throughput a
given tier actually permits.

This PR introduces:

1. An **optional `[rate_limit]` block** in the model TOML schema, sufficient
   to capture the three most common shapes providers publish today
   (single tier, tiered, or undisclosed).
2. A new **"Rate Limits" column** in `ModelTable` (canonical models page)
   and `ProviderModelsTable` (provider / model pages), with sort, tooltip
   breakdown, and a mobile-collapsible treatment.
3. A **hand-authored starter set** for the providers whose public docs list
   stable, verifiable limits (OpenAI, Anthropic, Google) so the column ships
   with real values rather than entirely placeholder rows.

Everything is additive — no existing TOML needs editing to keep `bun validate`
green, and there are no breaking JSON output changes for downstream SDK users.

## Motivation

Cost and context window answer *"how much can I send and what does it cost?"*
but they don't answer *"how fast can I send it?"* A model with a generous
1M-token context is useless if a tier-level RPM/TPM cap means a single
backfill job takes hours.

Issue #204 explicitly calls this out: rate-limit visibility is the missing
third dimension (cost + latency + throughput) when choosing a provider. We
already know this is a recurring pain point — users hit timeouts only after
they wire a model into a batch job, then realize "the docs say X TPM" but
their tier gets Y.

Today the data isn't on `models.dev` at all, so every consumer has to scrape
provider pages themselves.

## Proposal

### 1. Data shape — extend `packages/core/src/schema.ts`

Add an optional `RateLimit` block on `ModelBase` so both inline `Authored`
metadata under `models/` and provider entries under `providers/` accept it:

```toml
[rate_limit]
requests_per_minute = 60               # nullable when only TPM is published
tokens_per_minute   = 100_000          # nullable when only RPM is published
window              = "minute"        # default if omitted -> "minute"
```

For providers that publish tiered limits (e.g. Anthropic's
"Free / Build / Scale / Enterprise"):

```toml
[[rate_limit.tiers]]
label = "Build"
requests_per_minute = 50
tokens_per_minute   = 30_000

[[rate_limit.tiers]]
label = "Scale"
requests_per_minute = 1_000
tokens_per_minute   = 400_000
```

For providers that do not publish limits, the block is **omitted entirely**.
The renderer treats absence as `"Undisclosed"` — the same sentinel already
used by other dimensions — rather than `0`, which would mislead sort.

Schema rules (all in `.strict()`):

- `requests_per_minute`, `tokens_per_minute` — `z.number().int().min(1)`,
  either one is sufficient, but at least one must be set.
- `window` — default `"minute"`; enum currently `"second"|"minute"`. Per-hour
  supported through `window = "minute"` + `60 * N` formula is **not** in the
  first cut; we keep the unit human-readable until we see demand.
- `tiers[]` — at most one of `tiers` and the top-level numbers may be set;
  duplicates are rejected by the existing `tier sizes must be unique`
  refinement, adapted for tier `label` rather than `size`.

`AuthoredModel` and the rendered `Model` type carry the new field; the
generated `models.json` and `catalog.json` keep their existing contracts
(just one new optional key per model).

### 2. UI — `packages/web/src/render.tsx`

- New `SortableTh` `Rate Limits` column in **`ModelTable`** (canonical models
  view) and **`ProviderModelsTable`** (provider / model views).
- `data-sort` uses a synthetic key: `requests_per_minute ?? 0` so high-RPM
  models surface first when sorting descending.
- Cell renders the **most prominent value** first:
  - Tiered: `<N> req/min · <M> tok/min · +<k> tiers →`
  - Single: just the published number + unit.
  - Missing: `"Undisclosed"`.
- **Tooltip** (`title=` attr) lists every tier on hover so the column stays
  a single token of width on desktop.
- **Responsive collapse**: extend the existing `index.css`
  `@media (max-width: 52rem)` rule (currently enforces
  `table { min-width: 62rem }` + horizontal scroll) to add the new column to
  the existing "context + output + price" priority group that is **always
  visible**, while the new column joins the **second-priority group** (gets
  the same horizontal scroll treatment as Reasoning/Tool Call/etc.). No
  additional `media` query is needed in this PR.
- `EmptyRow` `columns` counts in `ModelTable` and `ProviderModelsTable` are
  bumped by 1 (and so are the matching `columns={N}` props on `TableSection`
  in `ModelPage`, `ProviderPage`, `LabPage`, `HomePage`).

### 3. Refresh process

We **do not** try to scrape limits in the existing GitHub Actions sync
workflow (`sync-models.yml`) in this PR. Limits change too often and too
unevenly across providers for a daily cron to be authoritative. Instead:

- Hand-author the value where the provider publishes a stable number
  (`issue #204`'s "Best Practices" section is the editorial bar).
- A follow-up issue will track adding a per-provider `sync.md`-style guide
  for limits and any endpoint that exposes them programmatically (e.g.
  OpenAI's `GET /v1/organization/usage/tokens`).

This avoids the trap of publishing stale numbers that look authoritative.

## Files touched

| File | Why |
|------|-----|
| `packages/core/src/schema.ts` | Add `RateLimit`, `RateLimitTier`, refinement; allow `ModelBase` to carry `rate_limit` |
| `packages/core/test/schema.test.ts` | Unit tests for new schema (valid, missing-either-side, both-set, tiers + scalar mutual exclusion) |
| `packages/core/src/describe.ts` | Optional: include the rate cap in auto-generated descriptions when present (gated on a new flag so existing descriptions aren't rewritten) |
| `packages/web/src/render.tsx` | `Rate Limits` column in `ModelTable` + `ProviderModelsTable`; tooltip; column counts |
| `packages/web/src/index.css` | Tweak `index.css` mobile block only if `min-width` needs to grow, otherwise no change |
| `providers/openai/models/<tier-1-and-2>.toml` | Sample authored data for OpenAI tier 1 + tier 2 (TPM + RPM where applicable) |
| `providers/anthropic/models/claude-*.toml` | Sample authored data for Anthropic Build / Scale |
| `providers/google/models/gemini-*.toml` | Sample authored data for Gemini tier 1 |
| `docs/pr-204-rate-limits.md` | This file |

> Note: the schema adds **no required** fields. Existing TOMLs that do not
> declare `[rate_limit]` continue to validate and render `"Undisclosed"`.

## Best Practices the PR enforces (per issue #204)

- **Primary value first**: `requests_per_minute` is shown ahead of
  `tokens_per_minute` when both are set, because users who batch typically
  think in TPM first; both are shown together when known.
- **Units literal in column**: `req/min`, `tok/min`. We deliberately do not
  invent a fuzzy "RPM/TPM" one-letter abbreviation to keep screen-reader and
  copy-paste consumers accurate.
- **Tiered breakdown in tooltip, not in cell**: cell stays one line so the
  table doesn't balloon horizontally.
- **Undisclosed sentinel**: missing block == `"Undisclosed"`, never `"0"`,
  never `"-"`.
- **No inferred limits from docs copy**: if we don't have a primary number
  from the provider's own docs, we don't author by analogy.

## Acceptance criteria

- [ ] `bun validate` passes against `providers/` and `models/` with the new
      schema block added but only the starter providers populated.
- [ ] New column renders on `/models`, `/providers/<id>`, and
      `/labs/<id>` (verified via static build).
- [ ] Sorting by Rate Limits descending surfaces the highest-RPM / TPM
      models first.
- [ ] Tooltip on tiered rows shows every tier with its label and numbers.
- [ ] Existing 52rem breakpoint keeps the new column readable inside the
      existing horizontal-scroll region without further media queries.
- [ ] `packages/core/test/schema.test.ts` covers: scalar, tiered, both-set,
      missing-both, mutual exclusion, `Undisclosed` empty-cell rendering.
- [ ] No change to `models.json`/`catalog.json` **shape** beyond the new
      optional `rate_limit` key (verified by `git diff models.json | head`).

## Out of scope (explicit non-goals for this PR)

- Auto-scraping limits in `sync-models.yml` (deferred — see "Refresh
  process" above).
- Embeddings / image / audio quotas — these are not currently listed on
  `models.dev` and would need a separate schema discussion.
- Per-endpoint caps (chat completion vs. batch API) — first cut is one
  compact rate per model.
- Regional variance within a single provider (eu./us./global. on Bedrock,
  Vertex) — the existing model file naming already supports per-region
  files; if data shape diverges, that's a follow-up issue.

## Test plan

1. **Schema**: `bun test packages/core` — new cases in `schema.test.ts`.
2. **Static build**: `cd packages/web && bun run build` — verify column
   appears in rendered HTML on three pages.
3. **Visual smoke**: load `/models`, `/providers/anthropic`, and
   `/labs/anthropic` in a browser at desktop (≥1024px) and mobile (≤768px)
   widths; verify column placement, tooltip, and scroll behavior.
4. **CI**: `bun validate` (already wired) and the existing `validate.yml`
   workflow must pass with new TOML samples.

## Risk / rollback

Low risk:

- New schema field is **optional**; worst case a downstream consumer sees
  an unknown JSON key. Engines that already do strict parsing on
  `catalog.json` see one extra key, which is additive.
- Column adds a column to tables; if a maintainer objects to the order,
  the column is a single component touch-up.
- Rollback path: revert commit; no data migration needed because
  `[rate_limit]` blocks were never present before this PR.

## Follow-ups (tracked, not in this PR)

- Open issue: "Sync rate limits from provider APIs where available"
- Open issue: "Per-endpoint rate limits (chat vs. batch vs. realtime)"
- Open issue: "Regional rate-limit variance on Bedrock / Vertex"

---

### Source citations

- Issue #204 — Add "Rate Limits" Column to Models Table.
- `packages/core/src/schema.ts` — existing `Cost`/`CostTier` patterns reused
  for the optional `RateLimit` block.
- `packages/web/src/render.tsx` — existing `SortableTh` + responsive
  pattern reused for the new column.
- `.github/workflows/sync-models.yml` — concurrency model preserved; limits
  are intentionally **not** added to the daily sync in this PR.
