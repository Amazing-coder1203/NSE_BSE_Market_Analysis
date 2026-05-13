# NSE Market Analysis Dashboard

End-to-end quantitative finance project analysing 3 years of Indian equity market data across **180 NSE stocks** and **10 NIFTY sectors**. Built a Sharpe-optimised portfolio using Modern Portfolio Theory that generated **120.9% cumulative return vs NIFTY 50's 27.7%** - an alpha of **+93.2%**.

Check out the web version of dashboard here: https://nsemarketanalysis.pages.dev/

---

## Tech Stack

| Layer | Tools |
|---|---|
| Data Ingestion | Python, yfinance |
| Data Processing | Pandas, NumPy |
| Database | PostgreSQL 16 (WSL2), SQLAlchemy, psycopg2 |
| Analysis & Visualisation | Pandas, NumPy, Matplotlib, Seaborn |
| Dashboard | Power BI Desktop (4 pages) |

---

## Project Structure

```
nse_bse_analysis/
│
├── 01_analysis.ipynb            # Full pipeline: fetch, clean, analyse, save CSVs
├── 03_db_load.ipynb             # Load all outputs to PostgreSQL
│
├── master_clean.csv             # 133,380 rows - clean OHLCV + returns + volatility
├── sector_summary.csv           # CAGR, volatility, Sharpe per sector
├── sector_daily_cumulative.csv  # Daily cumulative returns per sector (7,410 rows)
├── correlation_matrix.csv       # 10×10 sector correlation matrix
├── portfolio_vs_nifty.csv       # Portfolio vs NIFTY 50 daily comparison (739 rows)
├── portfolio_weights.csv        # Optimal MPT portfolio weights
│
└── NSE_BSE_Market_Analysis.pbix # Power BI dashboard
```

---

## Data Pipeline

**Universe:** 200 stocks attempted across 10 NIFTY sectors (20 per sector). 20 tickers dropped due to delisting or insufficient history (>30 days missing). Final universe: **180 stocks**.

**Cleaning steps:**
- Fetched 3 years daily OHLCV via `yfinance` with `auto_adjust=True` - handles splits and dividend adjustments automatically
- Dropped 20 tickers with >30 days missing data (new listings, delisted stocks)
- Forward-filled isolated gaps from trading halts and market holidays (Holi, Makar Sankranti, Labour Day)
- Replaced zero-volume rows on market holidays with forward-filled values
- Capped outliers using 1st/99th percentile IQR method per ticker - modifies values not rows
- Validated OHLC integrity: High ≥ Close ≥ Low for every row - zero corrupt rows

**Final dataset:** 133,380 rows × 10 columns, zero nulls in price data

---

## Analysis

### Sector Returns - 3 Year Cumulative

| Rank | Sector | Cumulative Return | CAGR | Ann. Volatility | Sharpe Ratio |
|---|---|---|---|---|---|
| 1 | Metal | 162.2% | 37.9% | 34.3% | 0.900 |
| 2 | Pharma | 108.1% | 27.7% | 25.7% | 0.804 |
| 3 | Auto | 90.9% | 24.0% | 28.0% | 0.609 |
| 4 | FinServ | 95.5% | 25.0% | 32.6% | 0.554 |
| 5 | Energy | 88.8% | 23.6% | 31.4% | 0.528 |
| 6 | Realty | 77.8% | 21.1% | 39.1% | 0.361 |
| 7 | Banking | 52.6% | 15.1% | 30.3% | 0.268 |
| 8 | FMCG | 37.3% | 11.1% | 24.7% | 0.168 |
| 9 | IT | 23.4% | 7.3% | 31.1% | 0.008 |
| 10 | Media | -1.3% | -0.5% | 39.4% | -0.189 |

### Volatility Analysis
- Computed 30-day rolling volatility per ticker: `daily_returns.rolling(30, min_periods=10).std()`
- Annualised: `daily_vol × √252` (252 NSE trading days per year)
- Most volatile: Media (39.4%), Realty (39.1%) - high risk, poor returns
- Most stable: FMCG (24.7%), Pharma (25.7%) - defensive sectors

### Correlation Matrix
- Computed 10×10 Pearson correlation matrix on sector average daily returns
- Highest correlation: Energy-Metal (0.79) - both commodity/cyclical, poor diversification pair
- Lowest correlation: FMCG-IT (0.44) - best diversification pair. IT driven by US tech spending, FMCG by Indian domestic consumption - independent macro drivers

### Efficient Frontier (Modern Portfolio Theory)
- Simulated 5,000 random portfolios using Dirichlet weight distribution
- Computed proper portfolio volatility using the **full covariance matrix**: `σ_p = √(wᵀ Σ w)`
- Covariance matrix built from correlation matrix and annualised volatilities: `Σᵢⱼ = ρᵢⱼ × σᵢ × σⱼ`
- Portfolio volatility range: **21.98% - 33.34%** (minimum variance at 21.98% - below any individual sector, proving diversification benefit)
- Best Sharpe from random simulation: **0.812**

---

## Portfolio Construction

Re-ran simulation with `np.random.seed(42)` for reproducibility. Extracted weights from the max Sharpe portfolio:

| Sector | Weight | Rationale |
|---|---|---|
| Pharma | 39.6% | Best risk-adjusted return (Sharpe 0.804), low volatility (25.7%) |
| Metal | 27.8% | Highest raw return (162%), strong CAGR justifies allocation |
| Energy | 10.0% | Moderate Sharpe, diversifies away from Pharma |
| FinServ | 8.2% | Decent CAGR, low correlation with Pharma (0.59) |
| Auto | 6.4% | Solid Sharpe, adds cyclical exposure |
| Realty | 3.7% | Small allocation despite high volatility |
| Banking | 2.1% | Low Sharpe, minimal allocation |
| IT | 1.8% | CAGR barely above risk-free rate |
| Media | 0.4% | Negative Sharpe - essentially excluded |
| FMCG | 0.1% | Low return despite low volatility |

**Optimal portfolio:** Max Sharpe 0.845 | Expected CAGR 28.67% | Expected Vol 25.65%

### Portfolio vs NIFTY 50

| Metric | Optimized Portfolio | NIFTY 50 Benchmark |
|---|---|---|
| Cumulative Return | **120.9%** | 27.7% |
| Alpha | **+93.2%** | - |

Portfolio daily return: `portfolio_return = sector_returns_wide @ optimal_weights`

---

## Database Schema (PostgreSQL 16)

Normalised dimensional model with 7 tables following `dim_` / `fact_` naming conventions:

```
dim_ticker              (180 rows)   ticker, sector
fact_prices         (133,380 rows)   date, ticker, OHLCV, daily_return, volatility_30d
fact_analysis         (7,410 rows)   date, sector, cumulative_return
fact_sector_summary      (10 rows)   sector, CAGR, volatility, Sharpe
fact_correlation        (100 rows)   sector pairs in long format (10×10 matrix)
fact_portfolio          (739 rows)   date, portfolio vs nifty cumulative returns
fact_weights             (10 rows)   sector, optimal_weight
```

---

## Power BI Dashboard (4 Pages)

Connected to PostgreSQL via WSL2 port proxy. Bidirectional cross-filtering across all pages via table relationships.

**Page 1 - Market Overview**
Sector cumulative return bar chart, cumulative return curves over time, annual volatility bar chart, 4 KPI cards, date range slicer

**Page 2 - Sector Deep Dive**
Dropdown sector slicer driving all visuals - cumulative return line, per-stock rolling volatility chart, sector metrics table

**Page 3 - Correlation & Risk**
Interactive 10×10 correlation heatmap with conditional formatting, risk vs return scatter with Sharpe-sized bubbles, 7% risk-free rate reference line

**Page 4 - Portfolio vs NIFTY 50**
Portfolio vs benchmark line chart, allocation donut chart, 3 KPI cards (120.9% / 27.7% / +93.2% alpha), 5 rebalancing insights

---

## Key Findings

1. **Metal dominated** - 162% cumulative return driven by India's infrastructure boom and global commodity demand
2. **Pharma is the efficiency champion** - second-best Sharpe (0.804) at low volatility (25.7%)
3. **IT disappointed** - CAGR of 7.3% barely above the 7% risk-free rate; US rate hikes froze tech spending
4. **Media destroyed value** - only sector with negative return (-1.3%) and negative Sharpe (-0.189)
5. **FMCG-IT best diversification pair** - lowest correlation (0.44), independent macro drivers reduce portfolio volatility
6. **+93.2% alpha** - MPT-based sector allocation significantly outperformed passive NIFTY 50 index investing

---

## Setup & Reproduction

```bash
# 1. install dependencies
pip install yfinance pandas numpy matplotlib seaborn sqlalchemy psycopg2-binary

# 2. set up PostgreSQL in WSL2
sudo apt install postgresql
sudo service postgresql start
sudo -u postgres psql
```

```sql
CREATE DATABASE nse_analysis;
CREATE USER nse_user WITH PASSWORD 'nse_pass123';
GRANT ALL PRIVILEGES ON DATABASE nse_analysis TO nse_user;
\q
```

```bash
# 3. run notebooks in order
jupyter lab
# → 01_analysis.ipynb   (fetch, clean, analyse, save CSVs - ~5 min)
# → 03_db_load.ipynb    (load to PostgreSQL)

# 4. Power BI - WSL2 port proxy setup (run in Windows PowerShell as Admin)
# netsh interface portproxy add v4tov4 listenport=5432 listenaddress=0.0.0.0 connectport=5432 connectaddress=<WSL_IP>
# Connect Power BI: Server=127.0.0.1, Database=nse_analysis
# Open NSE_BSE_Market_Analysis.pbix
```

> Re-running `01_analysis.ipynb` fetches live data from Yahoo Finance. Results may differ slightly due to price updates. Portfolio weights are fixed via `np.random.seed(42)` for reproducibility.

---

## Disclaimer

For educational and portfolio demonstration purposes only. Portfolio optimisation uses in-sample historical data (look-ahead bias present by design). Not financial advice. Past performance does not guarantee future results.
