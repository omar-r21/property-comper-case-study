# PropertyComper — case study

Estimating after-repair value in Cook County, Illinois, and then checking whether the estimates were
any good.

This is a write-up, not the source. The code is private.

## Why

Automated valuations disagree by six figures on the same house, and none of them show their work. For
a renovation buyer that gap is the whole decision — the purchase price is known, but the exit is a
forecast, and every published estimate is one number with no error bar and no comps attached.

So: show the comps behind every valuation, report as-is and after-repair separately instead of blending
them, and measure the error against real outcomes rather than claiming accuracy.

## Data

All public, no paid feed. Cook County Assessor records via Socrata for sales, addresses and
characteristics; PTAX-203 transfer declarations, which every Illinois sale files, for the seller's
declared distress and renovation flags; Census geocoding for address to parcel.

Two things I found in the data mattered more than any modelling choice. The Assessor's sales file lags
three to five months — the county isn't slow, that particular feed is — so the ingest was rebuilt
around PTAX instead. And 18.5% of the parcel roll isn't housing: parking spaces and common areas sit in
the same file as homes and wreck any $/sqft distribution that includes them.

## Method

Pull recent sales near the subject, score them on distance, size, age, type and recency, drop outliers,
adjust for market drift since each comp sold. Then read two ends of the comps' $/sqft distribution: a
low percentile for as-is, a high one for after-repair.

That rests on a claim worth testing, so I tested it. Declared distressed sales land in the bottom half
of their own comp set 85% of the time; declared renovations land in the top half 76% of the time. The
ends of the distribution really do track condition.

## Backtest

PTAX records every transfer, so the ground truth was already there. A property bought and resold higher
within two years is a flip, and the resale price is what the ARV turned out to be. For 728 of them:
value the property as of its purchase date using only comps that had already sold, then compare to what
it actually resold for.

The overall number was unremarkable. Cutting it up wasn't:

| Resale value | Flips | Median error | Median $ error |
|---|---:|---:|---:|
| under $100k | 28 | 208% | $162,708 |
| $100k–200k | 116 | 62% | $95,952 |
| $200k–350k | 310 | 18% | $51,288 |
| $350k+ | 250 | 16% | $70,866 |

My first version of this table cut on what each flip was *bought* for and concluded that cheap
purchases were unreliable. That was wrong. Holding acquisition constant:

| | Flips | Median error |
|---|---:|---:|
| bought under $100k, resold under $150k | 65 | 136% |
| bought under $100k, resold $150k+ | 102 | 21% |

A $50k buy that exits at $250k errs 21%, which is the overall average. Cheap *property* is the problem,
not cheap purchases.

The reason is mechanical. The high percentile reads the renovated end of the local $/sqft range, and in
a cheap mixed neighbourhood that end runs three or four times the distressed end. $180/sqft on 1,000
square feet says $180k for a house on a street that tops out near $80k — it assumes an exit the area
can't support.

What changed: comps are matched to the subject's property type instead of mixed; the app refuses to
show an ARV in the segments where it knows it can't (low as-is value plus a wide as-is/ARV spread is
the fingerprint of a cheap mixed area); and dollar error is reported next to percentage error, since
percentages flatter expensive homes — the $350k+ segment has the best percentage error and the second
worst dollar error.

## Is 16% good?

Not on its own, no. Zillow publishes roughly 1.7–1.9% median error on-market and about 7% off-market.
This is several times worse.

Some of that gap is a harder question. An AVM values a house as it stands today; this predicts what it
sells for after an unknown renovation, six to twenty-four months out, and the renovation scope isn't
in public records. Two buyers paying the same price get different exits depending on what they spend,
and nothing in the data separates them. Commercial AVMs also train on listing data — photos, condition
notes, price history — where this has assessor records and transfer declarations.

But a harder question doesn't make the number good enough. On a $300k exit, 16% is about $48k, and the
whole margin on a flip is often $40–60k. The point estimate is directionally useful and not something
you'd trade on.

Which is why the segmentation is the actual output. One accuracy figure implies uniform reliability.
Segmented, it can say "expect about 16% here" and, where it can't, decline and explain why. A valuation
that knows where it fails beats one that's confidently wrong exactly where it costs the most.

## Stack

Next.js and TypeScript, Supabase/Postgres, Mapbox GL, Vitest. Ingest scripts in TypeScript against
Socrata and PTAX.

---

Estimates from public records, for research. Not an appraisal, not investment advice.

Copyright (c) 2026 Omar Rayan. All rights reserved. This write-up can be read and linked; the source
isn't published and no licence to it is granted.
