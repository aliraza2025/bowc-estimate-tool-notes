---
layout: article
title: Building the Data Plane Behind a Phone-Ready Window-Cleaning Quote
description: How I turned 11,000+ Housecall Pro jobs, San Diego parcel records, and a small set of tracked jobs into an explainable pricing engine that office staff can use on a live call.
author: Ali Raza
date: 2026-10-26
read_time: 11 min read
permalink: /data-plane/
note_label: Engineering note
note_number: "01"
tags:
  - Next.js
  - TypeScript
  - Data Pipelines
  - Regression
  - Pricing
toc:
  - id: the-quoting-problem
    label: The quoting problem
  - id: two-planes
    label: Two-plane architecture
  - id: step-1-sync
    label: Syncing job history
  - id: step-2-clean
    label: Cleaning window-only jobs
  - id: step-3-enrich
    label: Parcel enrichment
  - id: step-5-train
    label: Training the model
  - id: the-quote-path
    label: The live quote path
  - id: feedback-loop
    label: The feedback loop
  - id: validation
    label: Validation
  - id: lessons
    label: Lessons learned
---

Explore my complete [product, AI, analytics, and strategy portfolio](https://alirazaportfolio2026.netlify.app/).

A caller asks a simple question: "How much to clean the windows on my house?" Answering it well is harder than it sounds. The office has a phone number, an address, and maybe a rough description of the home. The price needs to be fair, consistent across agents, and ready before the caller loses interest.

I built the **Blue Ocean Estimate Tool** during my summer 2026 Business Operations Internship at Blue Ocean Window Cleaning, a residential and commercial cleaning company in San Diego. It is an internal Next.js app that produces phone-ready quotes for new customers from phone and Angi leads. This article explains the part that does most of the work but never appears on screen: the **data plane** that turns raw job history into the files the quote engine reads.

> The code excerpts below are shortened from the working project for readability. Figures are aggregates from the project's own validation reports. No customer-level data appears in this article.

## The quoting problem {#the-quoting-problem}

Before the tool, quoting relied on individual judgment and spreadsheets. Two agents could quote the same house differently, and neither could easily explain why. The business already had the raw material for something better: years of completed jobs in Housecall Pro (HCP), its field-service platform. That history was messy, though.

- **Mixed tickets.** A single job might combine window cleaning with solar panels, gutters, screens, or pressure washing. Its total says little about the price of windows alone.
- **Missing property data.** Jobs record an address, not the square footage, bedrooms, or stories that drive effort.
- **Unstructured hours.** Time on site often lives in free-text descriptions rather than clean fields.
- **Scope ambiguity.** "Inside only," "outside only," "select windows," and "partial" all mean very different amounts of work.

The goal was a quote an agent could give in under a minute, with a range, a confidence signal, and a plain-language explanation of how the number was reached.

## Two-plane architecture {#two-planes}

I split the system into two planes with very different jobs.

The **data plane** is a set of offline Node.js scripts. It pulls job history, cleans and enriches it, fits models, validates them, and *promotes* the results into versioned JSON files.

The **quote path** is the Next.js app. It never touches raw history. On each request it reads the promoted JSON, looks up the property, and computes a price.

```text
Housecall Pro API ──> sync ──> clean ──> enrich ──> calibrate ──> train
                                                                    |
                         office corrections ──┐                     v
                                              ├──> validate ──> promote
                                              |                     |
                                              |                     v
                                              |        data/*.json artifacts
                                              |                     |
Caller address ──> parcel lookup ──> quote engine <─────────────────┘
                                          |
                                          v
                        price + range + confidence + explanation
                                          |
                                          └──> office log ──> corrections
```

This boundary keeps expensive preparation outside the request path. API pagination, text parsing, and regression fitting happen offline, so a failed preparation run can be inspected before its artifacts replace the files used for quotes. Artifact promotion and concurrent writes still need explicit controls.

## Step 1: sync completed jobs from Housecall Pro {#step-1-sync}

The pipeline starts with the HCP Public API. The sync script pulls completed jobs for a time window, caches the raw responses, and merges them into the same CSV shape as a manual HCP export. Keeping one export format meant every later stage worked whether the data came from the API or a spreadsheet download.

```bash
node ml/sync-hcp-api.mjs --days=180 --with-line-items   # incremental pull
node ml/sync-hcp-api.mjs --comps-only                   # rebuild comps offline
node ml/sync-hcp-api.mjs --max-pages=5 --dry-run        # safe test run
```

The `--comps-only` flag rebuilds downstream files from the cached export without calling the API again. That made iteration cheap: I could change cleaning rules and regenerate everything in seconds. The most recent sync covered **11,482 jobs**, with on-site hours recovered for **7,595** of them and technician counts for **8,971**.

## Step 2: keep only window-cleaning work {#step-2-clean}

The single most important cleaning decision was to train only on jobs whose price reflects window cleaning. A classifier reads each job description and keeps three pure categories: inside and out, outside only, and inside only.

```js
export const NON_WINDOW_KEYWORDS = [
  { key: "solar_panel", patterns: [/solar\s*panel/i, /\bsolar\b/i] },
  { key: "gutter", patterns: [/gutter/i, /ledges?\s+cleaned/i] },
  { key: "screen_repair", patterns: [/screen\s*repair/i] },
  { key: "pressure_wash", patterns: [/pressure\s*wash/i, /power\s*wash/i] },
  // ...about thirty categories in total
];

export const PURE_WINDOW_CATEGORIES = new Set([
  "window_inside_and_out",
  "window_outside_only",
  "window_inside_only",
]);
```

Jobs are not simply dropped when they don't fit. They are routed into separate files for **mixed**, **ambiguous** ("select windows," "west-facing only," "TBD"), **unspecified**, and **excluded** work. For mixed jobs, the cleaner tries to extract the window-only amount from line items, so a window-plus-gutter ticket can still contribute its window portion. Keeping the rejects visible made the filter auditable: when a rule looked too aggressive, I could open the excluded file and see exactly what it had removed.

## Step 3: enrich with San Diego parcel records {#step-3-enrich}

A job's address becomes useful only when it is joined to property features. The enrichment step queries the public SanGIS parcel service for living area, bedrooms, and bathrooms. Two details mattered more than expected.

First, the parcel data contains two size fields. One describes the lot, not the house. Using it inflated large-lot homes dramatically, so the lookup prefers assessor living area:

```ts
/** Prefer assessor living area; USABLE_SQ_FEET is lot/usable land, not house size. */
function inferSqft(attrs: ParcelAttributes): number | undefined {
  return parseNumericField(attrs.TOTAL_LVG_AREA) ?? parseNumericField(attrs.USABLE_SQ_FEET);
}
```

Second, bathroom counts are sometimes stored without a decimal point, so "35" means 3.5 baths:

```ts
const n = parseInt(text, 10);
if (n >= 10 && n < 100) return n / 10;
```

Address matching is fuzzy by nature, so the lookup tries a sequence of query strategies, from the strictest (house number, street, community, and ZIP) to looser fallbacks. A final sanity check caps square footage that is implausible for the bedroom and bathroom count, which catches bad parcel matches before they reach the price.

Parcel enrichment succeeded for **3,323 of 11,464** validated jobs (29%). That coverage limit shaped the design: the quote engine has to work well both with and without a parcel match.

## Step 4: recover hours and rates from descriptions {#step-4-signals}

Price alone does not explain effort. A parser extracts price-relevant signals from job descriptions and API job objects: on-site hours, technician count, and per-pane notes. Durations outside a plausible range (under 15 minutes or over 16 hours) are discarded rather than trusted.

From these signals the pipeline computes **dollars per hour by scope**. That rate becomes an independent pricing signal for brand-new addresses: predicted hours multiplied by the historical hourly rate.

## Step 5: train the exterior model and derive the rest {#step-5-train}

My first instinct was to fit a separate regression for each scope. The data argued otherwise. The model for "inside and out" jobs was far noisier (mean absolute error around **$174**) than the exterior-only model (around **$61**), largely because interior work varies so much from home to home.

So I made exterior pricing the single regression and derived the other scopes from it with the company's established rule of thumb: exterior is 60% of a full inside-and-out job.

```ts
export const EXTERIOR_SHARE_OF_BOTH = 0.6;
export const INTERIOR_SHARE_OF_BOTH = 0.4;

export function bothFromExterior(exteriorPrice: number): number {
  return exteriorPrice / EXTERIOR_SHARE_OF_BOTH;
}

export function interiorFromBoth(bothPrice: number): number {
  return bothPrice * INTERIOR_SHARE_OF_BOTH;
}
```

The exterior model is an ordinary least squares regression on square footage, bedrooms, bathrooms, property type, and lead source. Its training set blends two sources. A capped base sample of cleaned historical jobs carries modest weights, with parcel-matched rows weighted higher than unmatched ones. A small **gold set** of 18 jobs from a manually maintained job-tracking sheet, with clean paid amounts and durations, carries a much heavier weight so it can steer the fit. On that gold set the model reached a mean absolute error of about **$58**, with two-thirds of quotes within 25% of the actual price.

## Step 6: promote artifacts, not databases {#step-6-promote}

Every stage writes plain files. The final step promotes validated outputs into `data/`: model coefficients, calibration tables, price lookups, address comps, and sync metadata. The app reads them through a small cache keyed on file modification time:

```ts
export function readJsonData<T>(relativePath: string): T {
  const file = path.join(process.cwd(), relativePath);
  const mtime = fs.statSync(file).mtimeMs;
  const hit = caches.get(relativePath) as CacheEntry<T> | undefined;
  if (hit && hit.mtime === mtime) return hit.data;
  const data = JSON.parse(fs.readFileSync(file, "utf8")) as T;
  caches.set(relativePath, { data, mtime });
  return data;
}
```

When the pipeline promotes a new comps file, the next quote picks it up automatically, with no restart and no migration. For a small internal tool, that tradeoff was worth it: the "database" is a folder that can be diffed, backed up, and versioned, and every model carries a version string that appears on each quote.

## The live quote path {#the-quote-path}

With the artifacts in place, a quote request is fast and deterministic. The engine layers several signals, each with its own weight:

1. **Exterior regression**, adjusted for stories with a ladder premium.
2. **Lookup tables** of historical prices by property segment, blended in proportion to how many jobs support the matching cell.
3. **Address comps** from prior jobs at the same address, weighted heavily when there are several visits.
4. **Hours times rate** for new addresses, using predicted hours and the historical hourly rate.
5. **Price-ladder snapping** up to the next standard price point the office already uses.
6. **Guardrails**: a floor and ceiling per scope, plus a range around the recommended price.

```ts
const blendedScoped = lookupWeight * scopedLookup.price + (1 - lookupWeight) * regScoped;

if (addressHit) {
  mergedScoped = addrWeight * addressHit.price + (1 - addrWeight) * blendedScoped;
} else if (hoursRateHit && scope !== "interior") {
  mergedScoped = hoursRateWeight * hoursRateHit.price + (1 - hoursRateWeight) * blendedScoped;
}

const snapped = snapToPriceLadderUp(mergedScoped);
const price = applyPriceGuardrails(snapped, exteriorBase, scope, propertyKind, {
  accurate: tightlyComped,
});
```

Two design choices made the output usable on a call. First, every quote returns a step-by-step explanation ("Step 1: exterior regression $X; Step 4: HCP lookup, n jobs, 40% blend…"), shown in a derivation panel, so an agent can defend the number. Second, a confidence score rises when the parcel matched, when the caller confirmed square footage, or when many historical jobs back the lookup cell. Low confidence prompts the agent to confirm details instead of guessing.

## The feedback loop {#feedback-loop}

A pricing model drifts unless reality flows back into it. The app has two surfaces: a full estimate workspace and a simplified front-desk view. Every desk quote is written to a shared office log. When a job's real price differs, staff record a "should-be" price on the Corrections page.

A single command folds those corrections back into the live tables:

```js
if (!skipSync) await run("node", ["ml/sync-correction-log.mjs"]);
await run("node", ["ml/build-price-lookup.mjs"]);
await run("node", ["ml/build-address-lookup.mjs"]);
await run("node", ["ml/promote-customer-data.mjs"]);
```

This provides a path for staff to correct a price and supply labels for later preparation runs. It does not establish adoption, correction quality, or improved accuracy; corrections still need scope checks and review before reuse.

## Validation {#validation}

Each pipeline run ends with validation against historical jobs, using the live TypeScript predictor rather than a separate reimplementation, plus bootstrap resampling to put confidence intervals on the error.

| Measure | Early validation | Live predictor |
|---|---|---|
| Jobs evaluated | 8,362 | 11,464 |
| Mean absolute error | $239 | $73 (95% CI $70–$76) |
| Median absolute % error | 47% | 10% |
| Quotes within 10% | 11% | 49% |
| Quotes within 25% | 26% | 73% |

The two columns use different job sets, so this is not a controlled experiment. The early column is the first validation pass. The live-predictor column, three days later, scored the enriched training set with the full blended engine, including lookups and address comps. The later run combines cleaning, enrichment, and comparable-job signals, but these evaluations do not isolate the contribution of each change.

The numbers also need an honest caveat. The live predictor uses address comps built from the same history it is scored on, so repeat addresses look better than a truly new customer would. The reported mean bias was about $23 under the reference price. This is an observed error, not evidence that underpricing is desirable. A separate check on 17 known townhome units from an HOA project landed within 10% for 82% of units; the small, specific sample does not establish generalization.

## Tradeoffs and limits {#tradeoffs}

The architecture is deliberately small, and that has costs.

**Files instead of a database.** JSON artifacts are easy to inspect and version, but they assume a single writer and a modest data size. A larger operation would move job history to PostgreSQL and schedule the pipeline.

**Address matching is heuristic.** Normalized street strings and fuzzy parcel queries require match-quality checks. Parcel coverage was 29% in the reported run; the evaluation does not isolate its effect on accuracy.

**Evaluation is optimistic for repeat addresses.** The next step is a time-based holdout that scores only addresses first seen after the training cutoff.

**The gold set is small.** Eighteen heavily weighted jobs can strongly affect the fit. More representative labels and held-out evaluation are needed to assess stability.

**Customer data needs a separate home.** Training inputs contain customer-level records. In a next iteration I would keep raw exports in private storage and commit only aggregated, anonymized artifacts.

## Lessons learned {#lessons}

The biggest lesson was to make data assumptions inspectable: which jobs count as window cleaning, which size field describes the house, and how hours are recovered from free text. The retrospective comparisons do not isolate the effect of those choices from model changes or repeat-address evidence.

The second lesson was to treat explainability as a product feature, not a nice-to-have. An agent will not trust a number they cannot explain to a customer. Showing each pricing step, with its source and weight, gives staff a basis for reviewing and correcting a quote.

The resulting design uses scripts to write versioned JSON artifacts for a small Next.js app. That makes the calculation inspectable and identifies future storage and promotion work. It does not, by itself, establish business savings, adoption, or a reduction in quoting time.

---

**Project:** Blue Ocean Estimate Tool (internal Next.js app)  
**Author:** Ali Raza, Business Operations Intern, Blue Ocean Window Cleaning  
**Topic:** Data-plane architecture for service-business price estimation

Explore my complete [product, AI, analytics, and strategy portfolio](https://alirazaportfolio2026.netlify.app/).

