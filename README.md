# Market Basket Analysis – removing the Item × Location association

Analysis of ~517k retail transaction lines (Dec 2010 – Dec 2011, 19,371 baskets, 3,928 products, 30 countries).
Goals: (1) measure and **remove the statistical association between `Itemname` and `Location`**, (2) extract
basket insights (frequent items, combinations, rules, time/location patterns), (3) ship it as a reproducible repo.

> Column mapping from the brief: `TransactionID` = **`BillNo`**, `Location` = **`Country`**. Extra columns: `Quantity`, `Date`, `Time`, `Price`, `CustomerID`, `Itemname_Last2` (last two words of the item name, used as a coarse "product family").
> Currency is not stated in the data; revenue is assumed to be £.

## Repository layout
```
data/raw/            market_basket_raw.csv.gz   input (as supplied)
data/processed/      clean_lines / decorrelated_lines (regenerated, git-ignored)
scripts/             01_clean.py  02_decorrelate.py  03_insights.py  common.py  run_all.sh  make_notebook.py
notebooks/           market_basket_analysis.ipynb   (executed: EDA + results)
reports/figures/     10 PNG charts
reports/tables/      CSV outputs (rules, itemsets, association tables, cleaning log, ...)
requirements.txt
```

## Usage
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
bash scripts/run_all.sh                       # ~30 s: clean -> decorrelate -> insights
jupyter notebook notebooks/market_basket_analysis.ipynb
```

## 1. Preprocessing
The file is described as cleaned, but the audit found problems. `01_clean.py` removes them and logs each rule (`reports/tables/cleaning_log.csv`):

| Step | Rows dropped |
|---|---|
| Missing `Itemname` | 1,455 |
| Non-product rows (POSTAGE, Manual, CARRIAGE, fees, "damaged", "found", ...) | 2,623 |
| `Quantity` ≤ 0 | 176 |
| `Price` ≤ 0 | 466 |
| Exact duplicate lines | 5,166 |
| **Rows kept** | **507,522** |

`CustomerID` is missing on 26% of rows; it is not needed for basket analysis so it is left untouched. No normalisation was applied: basket analysis works on item presence/absence, so scaling numeric columns would add nothing.

## 2. Removing the Item × Location correlation

**Setup.** Unit = basket. Table *T[item, location]* = number of baskets of that location containing the item. Countries with < 80 baskets are pooled into "Other" (7 groups); only items in ≥ 100 baskets are tested (1,511 items).

**Measuring.**
* Chi-square statistic and **bias-corrected Cramér's V** (Bergsma 2013) – plain V is inflated on sparse tables.
* A textbook chi-square p-value would be invalid (one basket fills many cells; many cells have tiny expected counts), so significance is calibrated by a **permutation test**: shuffle location labels across baskets (baskets are exchangeable) and recompute. The same permutations give per-item p-values, adjusted with Benjamini–Hochberg.

**Removing.** Observed data can't be "decorrelated" without changing it, so we change *which items are analysed*: repeatedly drop the 3% of items with the largest chi-square excess over its permutation expectation, re-testing each round, until the global permutation test no longer rejects independence and V ≤ 0.05.

| | Items | Cramér's V (corrected) | Permutation p |
|---|---|---|---|
| Before | 1,511 | **0.106** | 0.002 (association is real) |
| After | 694 | **0.027** | 0.070 (independence not rejected) |

817 items were removed (list: `reports/tables/removed_location_dependent_items.csv`). They are mostly children's lunch boxes, plasters tins, snack boxes and cutlery sets that sell several times more often than average to Belgium, France, Germany and the Netherlands but under-sell in the UK (figure 08).

**Trade-offs you should know about**
* Only 26% of lines (130k) survive, and the retained set is *not* the bestsellers: the UK-skewed top sellers (e.g. White Hanging Heart T-Light Holder) are location-dependent relative to the 7 groups, so they are pruned. Because of that, rules are reported for **both** datasets (`rules_full.csv`, `rules_decorrelated.csv`); only 20 of the 80 decorrelated rules also appear in the full-data list.
* 93% of baskets are UK, so "location" is mostly "UK vs. everyone else". Small-country effects rest on few baskets (Belgium 98, Netherlands 93, Spain 88).
* The 0.07 permutation p-value (300 permutations) is a "fail to reject", not proof of independence.
* After pruning, countries are re-pooled from the remaining baskets, so figure 08 (right) shows fewer country columns.

## 3. Insights
(Figures in `reports/figures/`; tables in `reports/tables/`.)

* **Most frequent items (full data):** White Hanging Heart T-Light Holder (2,198 baskets), Jumbo Bag Red Retrospot (2,061), Regency Cakestand 3 Tier (1,904), Party Bunting (1,655), Lunch Bag Red Retrospot (1,541). Among location-independent items the leader is Spotty Bunting (1,113).
* **Combinations are mostly colour/variant siblings.** Full data: Jumbo Bag Pink Polkadot + Jumbo Bag Red Retrospot (4.6% of baskets); Green + Roses Regency Teacup (4.1%). Decorrelated data: Regency Tea Plate Green + Pink (1.9%).
* **Strongest rules (decorrelated, lift 35–56):** Set/6 Collage Paper Cups ⇄ Plates (confidence 89%), Christmas Gingham Star ⇄ Tree (72%), Happy ⇄ Monsters Stencil Craft, Glass Apothecary Bottle Perfume ⇄ Tonic, Apple ⇄ Strawberry Bath Sponge. Cross-sell implication: bundle matching sets / variants.
* **Time.** Peak trading 10:00–15:00 (noon is the busiest hour, 3,146 baskets); Thursday is busiest (4,118) and **there is no Saturday trading** in the data. Revenue builds from September and peaks in November (≈ £1.41M vs. £0.5–0.7M in a typical month); December 2011 is partial (data ends 9 Dec).
* **Location.** UK = 92% of baskets and 88% of revenue. Average basket ≈ £470 in the UK/Germany/France, but the Netherlands (£3,039) and Australia (£2,435) place a few very large (wholesale-style) orders.
* **Baskets** are skewed: median 15 distinct items, mean 26, max 1,095 – so a few wholesale baskets dominate item co-occurrence counts.

## 4. Challenges and how they were handled
| Challenge | Handling |
|---|---|
| "Cleaned" data still contained noise rows | Audited and filtered with a logged rule per step |
| Chi-square assumptions violated (non-independent cells, sparse table) | Permutation test + bias-corrected Cramér's V + BH correction |
| Raw V looked "small" (0.106) yet was highly significant | Used a stopping rule on the permutation test, not just an effect-size cut-off (a V ≤ 0.10 rule stopped immediately) |
| Per-item significance vanished while the global test still rejected (many tiny effects) | Kept pruning until the global test passed |
| Heavily UK-dominated data | Pooled small countries; documented that results rest on few non-UK baskets |
| mlxtend memory/time with 4k items | Basket matrix built sparsely, FP-Growth with `max_len=3`, min support 1% |

## Limitations / next steps
Stratified (per-country) rule mining; country-balanced weighting instead of item removal; seasonality-aware rules (Q4 gift items); customer-level analysis using `CustomerID`.
