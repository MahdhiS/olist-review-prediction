# Phase 1 — EDA Findings & Open Questions

*Living document — updated as each EDA phase completes. Full reasoning lives in `notebooks/01_eda.ipynb`; this is the fast-reference version.*

**Last updated:** 2026-08-13, after Solo phase.

## Problem statement (recap)

Predict binary good/bad `review_score` for an Olist order, using only information knowable at/around delivery time (no post-review leakage). Binary threshold not yet locked — decided in Duo, once feature/target relationships are known.

## Status

| Phase | Status |
|---|---|
| Shape, Quality, Merge, Missing Values | Done (session 7) |
| Solo (univariate) | Done (session 8) |
| Duo (feature vs. target, threshold) | Not started |
| Group (redundancy/multicollinearity) | Not started |

## Key decisions made

- **Target join verified first**, ahead of the rest of Quality — 768 orders have no review at all (dropped, no label), 551 had duplicate reviews (deduped, kept most recent by `review_answer_timestamp`).
- **Aggregation to order level**: `sum` for additive quantities (price, freight, weight), `count`/`nunique` for item/product/seller counts, `mode` for repeating categoricals, `max` (not `sum`) for payment installments.
- **Missing-value handling**: item/payment numeric gaps filled with `0` + a `has_items` flag (not dropped — the 756 no-item, has-review rows are 71% 1-star, real signal). Delivery-date nulls deliberately left untouched — no truthful fill value exists until a `delivery_days` feature is actually built in Duo.
- **Dropped**: `review_comment_title`/`review_comment_message` (free text, out of scope for a numeric/categorical baseline).
- **High-cardinality categoricals** (`item_mode`, `customer_state`) use a data-driven top-N + "other" bucketing (N chosen by cumulative-proportion cutoff, not a round number). Low-cardinality ones (`payment_type_mode`, 6 values) don't need it.

## Key findings (Solo phase)

- `review_score` shows **extreme response bias**: 1-star (11.5%) > 2-star (3.2%) and > 3-star (8.2%). Not a smooth decline — relevant to the threshold decision.
- Money-like features (`total_items_value` and similar) are **right-skewed / log-normal** — need `log_scale=True` to see the real shape; raw equal-width bins collapse under a few extreme outliers. Outliers not deleted — unresolved whether they're genuine large orders.
- `num_unique_products` / `num_unique_sellers` are **mechanically bounded** by `num_items` (can never exceed it) — likely redundant, not just correlated. Multicollinearity candidate for Group.
- `item_mode` (74 categories): heavily fragmented long tail, `other` bucket is the single largest bar. Its `unknown` placeholder (2,164 orders) is actually **two different populations** — no-items-at-all orders (71% 1-star) and items-with-missing-category orders — not yet split apart.
- `payment_type_mode`: `credit_card` (76.6%) and `boleto` (19.9%) dominate; `not_defined`/`unknown` combined are ~4 rows, disregarded as noise.
- `customer_state`: `SP`/`RJ`/`MG` = ~67% of orders. Geographic concentration — untested hypothesis that distant/low-volume states may see worse delivery outcomes.
- `has_items`: confirms the 756-row (~0.77%) no-items group from Quality phase, 71% 1-star.

## Open questions / TODO

- [ ] Lock the binary good/bad threshold for `review_score` — **Duo**
- [ ] Derive `delivery_days` / `is_delivered` — the project's core motivating hypothesis (late delivery → lower review) — **Duo, do this first**
- [ ] Test whether `item_mode`'s no-items subset of `unknown` drives the 1-star skew — **Duo**
- [ ] Test `customer_state` (distant states) against delivery lateness / review score — **Duo**
- [ ] Check `num_unique_products`/`num_unique_sellers` vs. `num_items` redundancy — **Group**
- [ ] Backlog: `seller_state` isn't in `df_clean` — add back if buyer-seller distance becomes worth testing
- [ ] Backlog: verify boleto vs. credit card is really about installments (`max_payment_installments`), not just guessed
- [ ] Backlog: rewrite Claude-drafted markdown prose (sessions 7 & 8) in Mahdhi's own words

## Next steps

1. Duo: derive delivery-lateness feature(s) first, then test every Solo candidate against `review_score`, then lock the threshold.
2. Group: redundancy check, starting with the `num_unique_products`/`sellers` flag above.
3. Update this doc after each phase — keep it short, keep the notebook detailed.
