# PropertyComper — case study

**Estimating after-repair value for Cook County, Illinois, and then checking whether the estimate was any good.**

This is a write-up, not the source. The engine is private; what follows is the problem, the method, what
the validation actually found, and what I changed because of it.

---

## The problem

Automated home valuations disagree by six figures on the same house and none of them show their working.
For a renovation buyer that gap is the whole decision: purchase price is knowable, but the exit — the
after-repair value (ARV) — is a forecast, and every published estimate is a single number with no error
bars and no comparable sales attached.

I wanted three things no consumer tool provides:

1. **Traceability.** Every valuation shows the comparable sales behind it and why each one scored as it did.
2. **Two numbers, not one.** As-is value and after-repair value are different questions; blending them
   into a single "estimate" hides the spread that the renovation case depends on.
3. **A measured error.** Not "our model is accurate" — an error distribution, segmented, from outcomes.

## The data

Everything comes from public records, with no paid feed:

- **Cook County Assessor** (Socrata): parcel sales, addresses, and characteristics — beds, baths, square
  footage, year built, property class.
- **PTAX-203 declarations**: the statewide transfer declarations filed on every Illinois sale. These carry
  seller-declared flags (distressed sale, recent renovation) that the Assessor's file does not, and they
  publish months sooner.
- **Census geocoder** for address → parcel resolution.

Two findings shaped the pipeline more than any modelling choice:

- **The Assessor's sales file lags by three to five months.** It is not the county's records that are slow;
  it is that feed. PTAX is the faster spine, so the ingest was redesigned around it.
- **18.5% of the parcel roll is not housing.** Parking spaces and common areas sit in the same file as
  homes and quietly poison any $/sqft distribution that includes them.

## The method

For a subject property, the engine pulls recent sales within a radius, scores each on similarity
(distance, size, age, property type, recency), drops outliers, and adjusts for market drift between the
comp's sale date and today. The valuation then reads **two ends of the comparable $/sqft distribution**:
a lower percentile as the as-is value, an upper percentile as the after-repair value.

That mechanism rests on a testable claim, so I tested it: declared distressed sales land in the bottom
half of their own comp set **85%** of the time, and declared renovations in the top half **76%** of the
time. The ends of the distribution really do correspond to condition.

## The validation, and what it found

PTAX records every transfer on a parcel, so ground truth is already in the data. A property bought and
resold higher within two years is a flip, and its resale price is what the ARV actually turned out to be.

For each of **728 real flips**: compute the valuation *as of the purchase date*, using only comps that had
already sold by then, and compare the predicted ARV to the realised resale price.

The headline error was unremarkable. Segmenting it was not:

| Resale value | Flips | Median absolute error | Median dollar error |
|---|---:|---:|---:|
| under $100k | 28 | **208%** | $162,708 |
| $100k–200k | 116 | 62% | $95,952 |
| $200k–350k | 310 | 18% | $51,288 |
| $350k+ | 250 | 16% | $70,866 |

My first version of this table keyed on what each flip was **bought** for and reported "40% median error
under $100k". That framing was wrong, and holding the acquisition price constant shows why:

| Segment | Flips | Median absolute error |
|---|---:|---:|
| bought under $100k, resold under $150k | 65 | 136% |
| bought under $100k, resold $150k+ | 102 | **21%** |

**Buying cheap is not the problem.** A $50k purchase that resells at $250k errs 21%, indistinguishable
from the overall average. Cheap *property* is the problem.

**The mechanism of the failure:** the upper percentile reads the renovated end of the comp $/sqft range.
In a cheap, mixed neighbourhood that end can be three to four times the distressed end. Multiplying
$180/sqft by 1,000 square feet returns $180k for a house in an area that caps out near $80k — the model
assumes a renovation-quality exit the neighbourhood cannot support.

**What changed because of it:**

- Comps are matched to the subject's own property type rather than mixed across types.
- The app refuses to show an ARV it cannot support, and says why, rather than printing a confident number.
  The reliability signature turned out to be a low as-is value combined with a wide as-is/ARV spread —
  exactly the fingerprint of a cheap mixed area.
- Absolute dollar error is reported next to percentage error, because percentages flatter expensive homes
  arithmetically: the $350k+ segment has the *best* percentage error and the *second worst* dollar error.

## Is 16% any good?

No — not on its own. It is worth being blunt about that, because the segmented table above is the point
of the project and it only means something next to a benchmark.

Zillow publishes a median error of roughly **1.7–1.9% for on-market homes and about 7% for off-market
homes** ([Zillow's published accuracy](https://www.zillow.com/z/zestimate/)). This model's best segments
sit at 16–18%. That is materially worse, and some of the gap is simply a harder question:

- An AVM estimates what a house is worth **today, as it stands**. This predicts what a house will sell for
  **after an unknown renovation, six to twenty-four months out**.
- The renovation scope is not in public records. Two buyers paying the same price produce different exits
  depending on what they spend, and nothing in the data distinguishes them.
- Commercial AVMs train on listing data: photos, condition notes, list-price history. This has assessor
  records and transfer declarations.

But the harder question does not make the number good enough. On a $300k exit, 16% is roughly $48k, and a
renovation margin is often $40–60k. **The point estimate is directionally useful and not decision-grade.**

That is why the work went into the error distribution rather than the headline: a single accuracy figure
would imply uniform reliability. Segmented, the tool can say "in this segment expect about 16%" and, in
the segments where it cannot, decline to answer and say why. A valuation that knows where it fails is
more useful than one that is confidently wrong in the places that matter most.

## What I would tell a reviewer

- **The validation is the project.** The scoring model is ordinary; knowing where it breaks, and refusing
  to answer there, is what makes it usable.
- **The error distribution is not uniform, and the average hid that.** A single headline accuracy number
  would have been technically true and practically useless.
- **Segmenting on the wrong variable produced a plausible, wrong conclusion** — "cheap purchases are
  unreliable" — that survived until the data was cut the other way.
- **Known limitations are written down**, not discovered by users: a running log of unverified surfaces and
  open work, plus a record of ideas that were measured and deliberately not built, with the evidence for each.

## Stack

Next.js (App Router) and TypeScript · Supabase/Postgres · Mapbox GL · Vitest · ingest scripts in TypeScript
against Socrata and PTAX.

---

*Valuations are estimates from public records, for research only — not an appraisal and not investment advice.*

**Copyright (c) 2026 Omar Rayan. All rights reserved.** This document may be read and linked; the
underlying source is not published and no licence to it is granted.
