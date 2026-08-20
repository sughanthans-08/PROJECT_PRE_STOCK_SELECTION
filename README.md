# Pre_Stock_Analysis

**A two stage quantitative screener for Indian equities (NSE / BSE)**

Narrows the ~7,800 listings on the Indian exchanges to a short list of companies
that are simultaneously high quality, high growth, and reasonably valued, using
the TradingView scanner API for market wide filtering and pandas + yfinance for
valuation analysis.

## The two stage funnel

The screen is split by capability, not convenience. Stage 1 asks *is this a good
business growing quickly?* Stage 2 asks the harder question, *and is it cheap
right now, both against its peers and against its own history?*

```
              ~7,800 NSE / BSE listings
                        |
   STAGE 1   Market cap, ROE, ROCE, sales growth (1Y / 5Y), PEG
             Filtered server side in a single API request
                        |
              collapse NSE / BSE dual listings
                        |
   STAGE 2a  Relative valuation:   PE < industry benchmark PE
                        |
   STAGE 2b  Historical valuation: PE < own long run average PE
             Derived metrics:      3Y CAGRs, average ROE
                        |
                   Final shortlist -> results.csv
```

**Stage 1 (TradingView).** Absolute fundamental thresholds are pushed to the
scanner so the entire market is filtered server side in one request, with no
bulk download.

**Stage 2 (pandas + yfinance).** Handles what the scanner cannot express:
cross sectional comparison against industry peers, time series comparison
against a stock's own trading history, and growth horizons the API does not
publish.

### Representative run

| Filter | Listings remaining |
|---|---:|
| NSE / BSE universe | 7,815 |
| Market cap > 4,000 Cr | 1,860 |
| ROE > 20% | 460 |
| ROIC > 20% | 337 |
| Sales growth 1Y > 20% | 137 |
| Sales growth 5Y > 16% | 97 |
| PEG < 1.3 | 58 |
| Deduplicated to unique companies | 31 |
| Below industry PE | 20 |
| Below own historical PE + derived growth tests | **5** |

## Repository layout

```
Pre_Stock_Analysis/
├── Quantitative_Equity_Screener.ipynb   # Main analysis notebook
├── requirements.txt
├── README.md
└── .gitignore
```

The notebook is committed with outputs saved, so the full analysis is readable
on GitHub without running anything.

## Running it

```bash
python -m venv venv
source venv/bin/activate        # Windows: .\venv\Scripts\Activate.ps1
pip install -r requirements.txt
jupyter notebook Quantitative_Equity_Screener.ipynb
```

Run all cells in order. Stage 2 is network bound and issues several requests per
surviving company, so a full run takes roughly a minute. The shortlist is
written to `results.csv`.

## Configuration

Every threshold lives in the `CONFIG` dictionary in section 1. Setting any value
to `None` disables that filter.

If the shortlist comes back empty, read the Stage 1 funnel table before changing
anything, because it reports how many companies each filter removes and therefore which
constraint is binding. In practice sales growth and PEG bind first.

| Parameter | Default | Suggested relaxation |
|---|---|---|
| `MIN_REVENUE_GROWTH_TTM` | 20 | 12 to 15 |
| `MIN_ROE` / `MIN_ROCE` | 20 | 15 |
| `MAX_PEG` | 1.3 | 2.0, or `None` |
| `STAGE2_METRICS` | 20 / 17 / 20 | `None` to skip derived growth tests |
| `APPLY_HISTORICAL_PE_FILTER` | `True` | `False` |
| `MIN_MARKET_CAP` | 4,000 Cr | 1,000 Cr for small caps |

## Implementation notes

Four data quality problems shaped the design, each of which fails silently
rather than loudly:

**Unrecognised API fields return `NULL`, not an error.** A misspelled field is
indistinguishable from a universally unreported metric and empties the screen
without warning. Field names were validated against the exchange metadata
endpoint and against live population rates before use.

**Field variants differ enormously in coverage.** On the Indian large cap
universe, `return_on_equity` and `return_on_equity_fq` are populated for ~14% of
companies against ~98% for `return_on_equity_fy`. Filtering on the sparser
variant discards most valid candidates while appearing to work.

**Every company is listed twice**, once per exchange. Uncorrected, this
double counts the universe and biases every industry aggregate. Dual listings
are collapsed before any aggregation.

**Ticker symbols are not portable between providers.** TradingView normalises
punctuation to underscores, so `BAJAJ_AUTO` is `BAJAJ-AUTO.NS` on Yahoo and
`M_M` is `M&M.NS`. The original character is unrecoverable from the symbol, so
each plausible spelling is attempted until one resolves.

Two analytical choices are also worth noting:

**The industry PE benchmark is drawn from the full market**, not from the Stage 1
survivors. Benchmarking against survivors is degenerate: the growth filters are
strict enough that most industries retain a single company, and a lone stock can
never sit below its own average, so valid candidates get eliminated by an
artefact of sample size. The median is used rather than the mean, since PE
distributions carry extreme outliers.

**Historical PE is reconstructed point in time.** Each month's close is paired
with the most recently *reported* fiscal year EPS, offset by a reporting lag so
no observation uses earnings that were not yet public. Dividing historical
prices by present day EPS would be simpler and meaningless. Since the provider
exposes roughly four annual EPS observations, the reconstructable window is
typically three to four years. The realised depth is reported per stock in
`pe_history_months`, and shallow histories are flagged rather than silently
judged.

## Reading the output

Beyond the headline metrics, `results.csv` carries the reasoning behind each
decision:

| Column | Meaning |
|---|---|
| `industry_pe`, `industry_peers` | Benchmark PE and how many peers formed it |
| `avg_pe_hist`, `pe_history_months` | Historical average PE and its depth |
| `industry_pe_status`, `hist_pe_status`, `stage2_growth_status` | Per test verdicts |
| `waiver` | Non empty means the stock was **retained because a test could not be evaluated**, not because it passed |
| `yf_status` | Whether financial data was retrieved cleanly |

## Limitations

- Screens on reported fundamentals only: no leverage, cash flow quality,
  promoter pledging, or governance analysis.
- Reported figures may lag or be restated, and consolidated versus standalone
  treatment varies by company.
- PEG is populated for roughly two thirds of the universe. Companies without it
  are excluded by the `MAX_PEG` filter rather than passed silently.
- This is a candidate generation tool, not investment advice.
