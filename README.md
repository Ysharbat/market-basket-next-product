# Market Basket Analysis & Next Product to Buy

**Who are our most valuable customers, which products are bought together, and what should we offer each customer next?**

This project answers these three questions on 1 million real online-retail transactions, using RFM segmentation, association rules (FP-Growth) and a next-product recommender that is tested honestly on future purchases. The same methods are used by banks for customer segmentation, cross-selling and *Next Best Offer* models (see [How this maps to banking](#how-this-maps-to-banking)).

### Key results

- **Recommender:** suggesting 5 new products per customer, the association-rule model reached a **Hit Rate@5 of 25.5% vs 20.8% for a best-seller baseline** (+4.7 points, 95% CI +2.5 to +6.8), tested on 2,150 customers over the 3 months after the training period.
- **Segmentation:** **15% of customers (Champions) generate 53% of revenue**; together with Loyal customers, 34% of customers bring in 80%.
- **Cross-sell:** the strongest product link (sugar bowl → matching milk jug) has **64% confidence and lift 23.5**, but the biggest opportunity is a weaker link reaching **1,089 baskets**.
- **Data quality:** removing purchases that were cancelled minutes later turned January 2011 from **+9.5% growth into −7.1%**. Cleaning changed the business conclusion.

---

## Contents

1. [Data](#data)
2. [Data cleaning](#1-data-cleaning)
3. [Exploratory analysis](#2-exploratory-analysis)
4. [RFM customer segmentation](#3-rfm-customer-segmentation)
5. [Market basket analysis](#4-market-basket-analysis)
6. [Next product to buy](#5-next-product-to-buy)
7. [How this maps to banking](#how-this-maps-to-banking)
8. [Limitations](#limitations)
9. [How to run](#how-to-run)

---

## Data

[Online Retail II](https://archive.ics.uci.edu/dataset/502/online+retail+ii) (UCI Machine Learning Repository): all transactions of a UK-based online gift wholesaler between **December 2009 and December 2011**. Many customers are small shops buying for resale.

| | Raw | After cleaning |
|---|---|---|
| Transaction lines | 1,067,371 | 997,099 |
| Invoices | | 39,251 |
| Customers (with ID) | | 5,839 |
| Products | | 4,716 |
| Countries | | 43 |

Columns: Invoice, StockCode, Description, Quantity, InvoiceDate, Price, Customer ID, Country. The data is not included in this repository (see [How to run](#how-to-run)).

---

## 1. Data cleaning

📓 [`notebooks/01_cleaning_eda.ipynb`](notebooks/01_cleaning_eda.ipynb)

Every rule was checked on the data before it was applied, and each one is logged with its row count. Rules are applied in order, so each count excludes rows already removed.

| # | Rule | Rows removed | Reason |
|---|---|---:|---|
| 1 | Sheet overlap (1–9 Dec 2010) | 22,523 | The same days appear in both Excel sheets as exact copies |
| 2 | Exact duplicate rows | 11,812 | Double entries (checked after standardising StockCode case) |
| 3 | Cancellation invoices (C…) | 19,104 | Returns, not purchases |
| 4 | Bad-debt adjustments (A…) | 6 | Accounting entries |
| 5 | Quantity ≤ 0 | 3,393 | Warehouse adjustments ("damages", "lost"), no customer |
| 6 | Price ≤ 0 | 2,621 | Not sales; mostly blank descriptions |
| 7 | Non-product codes | 4,634 | Postage, fees, discounts, vouchers, test items |
| 8 | Purchases reversed by a matching cancellation | 6,179 | Same customer, product and quantity cancelled later |
| | **Total** | **70,272 (6.6%)** | |

Rule 8 matters most for the results. The two largest orders in the data (80,995 and 74,215 units) were cancelled about 15 minutes after being placed. Without this rule, one customer who actually spent £169 would have appeared as a top customer with £77k of spending.

Other steps: StockCodes were uppercased (173 codes existed in two spellings), and each product was given its most frequent description (621 codes had several names).

---

## 2. Exploratory analysis

- **Seasonality:** revenue peaks in **November** in both years (£1.40M in 2010, £1.44M in 2011). Wholesale customers stock up *before* Christmas.
- **Growth:** January–November 2011 revenue was only **+2.2%** above 2010.
- **Countries:** the UK accounts for **85.4% of revenue** and 91.6% of invoices, so the basket analysis uses UK invoices only.
- **Guests:** orders without a Customer ID are 7.4% of invoices but 13.5% of revenue. They are kept for basket analysis but cannot be used for RFM or recommendations.
- **Basket size:** median 15 different products per invoice; only 8% of invoices contain a single product.

---

## 3. RFM customer segmentation

📓 [`notebooks/02_rfm.ipynb`](notebooks/02_rfm.ipynb)

Each registered customer is described by **R**ecency (days since last purchase), **F**requency (number of orders) and **M**onetary value (total spending), scored 1–5 by quintile and mapped to named segments with the standard R × F map. Monetary is not used in the map because it moves closely with frequency (Spearman ρ = 0.86); it is used to measure each segment's value instead. All one-time buyers (27.6% of customers) get F = 1.

| Segment | Customers | % customers | % revenue | Action |
|---|---:|---:|---:|---|
| Champions | 852 | 14.6% | **52.6%** | Reward and retain; early access, referrals |
| Loyal | 1,131 | 19.4% | 27.6% | Upsell and cross-sell (basket rules) |
| At Risk | 751 | 12.9% | 5.7% | Win-back campaign |
| Hibernating | 1,511 | 25.9% | 3.9% | Cheap automated email only |
| Potential Loyalists | 637 | 10.9% | 3.6% | Encourage a repeat order |
| Can't Lose | 73 | 1.3% | 3.0% | Personal outreach: high value, but no purchase for ~10 months |
| Need Attention | 266 | 4.6% | 1.8% | Personalised reactivation |
| About to Sleep | 380 | 6.5% | 1.2% | Low-cost reminder |
| Promising | 161 | 2.8% | 0.3% | Build the habit |
| New Customers | 77 | 1.3% | 0.2% | Onboarding |

**Takeaway:** a small group carries the business, so retention budget should go to Champions and Loyal first. *Can't Lose* (73 customers with a median of 11 orders and £3,575 spent) is the clearest win-back target.

---

## 4. Market basket analysis

📓 [`notebooks/03_market_basket.ipynb`](notebooks/03_market_basket.ipynb)

**Method:** UK invoices → basket matrix (invoice × product) → **FP-Growth** (minimum support 1%) → association rules with **support, confidence and lift**.

Decisions along the way:
- Baskets with **more than 200 products** were removed: only 1% of baskets, but 14% of all lines. Such wholesale stock-up orders link almost every product with every other.
- Products below 1% support were removed first (Apriori property): 4,706 → 595 products, with the same result much faster.
- FP-Growth found 595 frequent products, 327 pairs and 85 triples, giving **1,178 rules with lift > 1** (median lift 9).
- **89% of the rules link the same product in different designs** (e.g. red lunch bag → black lunch bag): wholesalers buy assortments. These were tagged using product-name similarity, so that cross-sell analysis focuses on links between *different* products (63 unique links).

**Strongest cross-sell links:**

| Rule | Confidence | Lift | Baskets with A but not B |
|---|---:|---:|---:|
| Red retrospot sugar bowl → small milk jug | 64% | 23.5 | 227 |
| Toy tidy → recycling bag (retrospot) | 55% | 15.5 | 411 |
| Painted metal pears → bird ornament | 73% | 10.2 | 181 |
| White wooden cabinet → white picture frame | 42% | 8.3 | 603 |
| White picture frame → white hanging heart T-light holder | 40% | 2.9 | **1,089** |

**Takeaway:** the strongest link is not the biggest opportunity. High-lift pairs suit bundles and "frequently bought together" suggestions; the weaker but wide link reaches almost five times more baskets and suits a low-cost email campaign. A rule shows that products are bought together, not that an offer will cause a sale, so any action should first be tested with a control group.

---

## 5. Next product to buy

📓 [`notebooks/04_next_product.ipynb`](notebooks/04_next_product.ipynb)

**Task:** for each customer, recommend **5 products they have never bought**, then check whether they buy them.

**Honest evaluation:**
- **Time split:** models learn only from data **before 1 September 2011** and are tested on purchases from **September to November 2011**.
- Only **first-time purchases** count as correct, because recommending what a wholesaler re-orders anyway is not cross-selling.
- Association rules were **rebuilt from pre-cutoff data only**, to avoid data leakage.
- 2,150 customers were active in both periods and bought at least one new product.

| Model | Hit Rate@5 | Precision@5 | Recall@5 |
|---|---:|---:|---:|
| Random | 2.6% | 0.5% | 0.1% |
| Best-sellers (all history) | 18.6% | 4.5% | 1.0% |
| Best-sellers (last 3 months) | 20.8% | 5.5% | 1.1% |
| **Association rules** | **25.5%** | 6.7% | 1.7% |
| **Item similarity (cosine)** | 24.9% | **6.9%** | **1.9%** |

*Hit Rate@5: share of customers who bought at least one of the 5 recommended new products.*

**Is the difference real?** A paired bootstrap (2,000 resamples of customers) gives:

| Comparison | Difference | 95% CI |
|---|---:|---|
| Rules vs best-sellers | +4.7 pts | [+2.5, +6.8] |
| Similarity vs best-sellers | +4.1 pts | [+1.8, +6.3] |
| Rules vs similarity | +0.6 pts | [−1.4, +2.5] |

**Takeaway:** both personalised models clearly beat the best-seller list, and are equal to each other within the margin of error. The rule-based model is easier to explain to business users ("recommended because you bought X").

---

## How this maps to banking

| This project | In a bank |
|---|---|
| Basket = one invoice | Product holdings of a customer (current account, card, deposit, loan…) |
| Recency / Frequency / Monetary | Days since last transaction, transactions per month, balances or income |
| Champions | Primary-bank customers (salary paid in, daily card use) |
| At Risk / Can't Lose | Early attrition signals: the population a churn model would score |
| Rule: sugar bowl → milk jug | Rule: salary account + debit card → credit card |
| "Baskets with A but not B" | Customers holding product A but not B: the campaign target list |
| Next-product recommender | **Next Best Offer** model |
| Time split + Hit Rate vs baseline | Standard validation of a bank NBO model: train on past months, test on the next |
| Bootstrap CI, A/B test before claiming impact | Campaign test vs control group |

---

## Limitations

- **One test period** (September–November 2011, the pre-Christmas season); results may differ in other months.
- **15% of new purchases were products launched after the cutoff**, which no model trained on history can recommend.
- **No product categories** in the data: "same product family" is estimated from names (about 10% of tags were wrong in a manual check of 20 rules).
- **Guest orders (13.5% of revenue)** cannot be followed over time, so they are excluded from RFM and recommendations.
- **Partial cancellations** are not netted out (only exact reversals are removed).
- **Correlation, not causation:** a hit means the customer bought the product, not that a recommendation caused the purchase. Measuring that needs a live A/B test.

---

## How to run

```bash
git clone https://github.com/Ysharbat/market-basket-next-product.git
cd market-basket-next-product
pip install -r requirements.txt
```

1. Download **Online Retail II** from the [UCI repository](https://archive.ics.uci.edu/dataset/502/online+retail+ii), unzip it, and place `online_retail_II.xlsx` in `data/raw/`.
2. Run the notebooks in order:
   - `01_cleaning_eda.ipynb`: loads the Excel file once (a few minutes), cleans it, saves `data/processed/online_retail_clean.parquet`
   - `02_rfm.ipynb`: RFM scores and segments
   - `03_market_basket.ipynb`: FP-Growth and association rules
   - `04_next_product.ipynb`: recommenders and evaluation

### Project structure

```
market-basket-next-product/
├── data/                 # not in the repository
│   ├── raw/              # online_retail_II.xlsx
│   └── processed/        # clean data, segments, rules, results
├── notebooks/
│   ├── 01_cleaning_eda.ipynb
│   ├── 02_rfm.ipynb
│   ├── 03_market_basket.ipynb
│   └── 04_next_product.ipynb
├── requirements.txt
└── README.md
```

**Tools:** Python, pandas, NumPy, SciPy, mlxtend (FP-Growth), Matplotlib.

---

*Author: Yusif Sharbatov, data analyst with a finance and accounting background, building a data science portfolio focused on banking.*