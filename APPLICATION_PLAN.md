# ITR Schedule FA — Section A3 Helper Tool
## Complete Application Plan (Portable for Any Agent)

---

## 1. OVERVIEW

**Purpose**: A local web tool to help fill Section A3 (Foreign Equity & Debt Interest) of Schedule FA in Indian Income Tax Return.

**Stack**: Python 3.10+ / Flask / vanilla HTML+CSS+JS / openpyxl / yfinance

**Portability**: Runs on Mac & Windows via `python app.py` → opens `http://localhost:5000`

**Status Key**: ✅ Done | 🔨 In Progress | ⬜ Not Started

---

## 2. REQUIREMENTS (CONFIRMED BY USER)

1. **SBI TT Rate**: Use SBI TT Buying Rate of **last working day of the month preceding** the transaction date — for ALL conversions (buy, sell, dividend, peak, closing).
2. **Year selector**: User picks the calendar year. Option to import previous year's saved JSON.
3. **Dividends**: Auto-detect via `yfinance` `ticker.dividends`. Skip if none. Allow manual override.
4. **Partial sells**: FIFO. Fractional shares supported.
5. **Save/Load**: JSON files for persistence.
6. **Manual override**: ALL calculated fields can be manually overridden.
7. **APIs**: All free, no login required.
8. **Export**: Excel (.xlsx) matching A3 table format.
9. **All 12 columns** of A3 must be present.

---

## 3. SECTION A3 — COLUMN DEFINITIONS

Each row = one acquisition lot of a foreign stock/ETF.

| # | Column Name | How to Calculate |
|---|---|---|
| 1 | Sl. No. | Auto-increment |
| 2 | Country Name and Code | From yfinance, e.g. `2-UNITED STATES OF AMERICA` |
| 4 | Address of entity | From yfinance, e.g. `5775 Morehouse Drive, San Diego, CA` |
| 5 | Zip code | From yfinance, e.g. `92121` |
| 6 | Nature of entity | `Company` or `ETF` (from yfinance quoteType) |
| 7 | Date of acquiring the interest | User input, format `DD/MM/YYYY` |
| 8 | Initial value of the investment (₹) | `buy_price_foreign × quantity × TTBR(last_wd_prev_month_of_buy_date)` |
| 9 | Peak value of investment during the period (₹) | Max of `(daily_close × qty_held_that_day × TTBR_for_that_month)` across all trading days in the calendar year |
| 10 | Closing balance (₹) | `close_price_dec31 × remaining_qty × TTBR(last_wd_prev_month_of_dec31)` — 0 if fully sold |
| 11 | Total gross amount paid/credited (dividends ₹) | `Σ(div_per_share × qty_on_ex_date × TTBR(last_wd_prev_month_of_ex_date))` |
| 12 | Total gross proceeds from sale (₹) | `Σ(sell_price × sell_qty × TTBR(last_wd_prev_month_of_sell_date))` — 0 if not sold |

---

## 4. SBI TT RATE — DATA SOURCE & FORMAT

### Source
GitHub repo: `sahilgupta/sbi-fx-ratekeeper`
Raw CSV URL: `https://raw.githubusercontent.com/sahilgupta/sbi-fx-ratekeeper/main/csv_files/SBI_REFERENCE_RATES_USD.csv`
For GBP: `https://raw.githubusercontent.com/sahilgupta/sbi-fx-ratekeeper/main/csv_files/SBI_REFERENCE_RATES_GBP.csv`

### CSV Format (verified)
```
DATE,PDF FILE,TT BUY,TT SELL,BILL BUY,BILL SELL,FOREX TRAVEL CARD BUY,FOREX TRAVEL CARD SELL,CN BUY,CN SELL
2020-01-06 09:00,https://...,71.65,72.50,71.59,72.65,71.00,72.85,70.70,73.00
```

### Key Notes
- Column index 0 = DATE (format: `YYYY-MM-DD HH:MM`)
- Column index 2 = TT BUY (this is what we need)
- **TT BUY = 0.00 on weekends/holidays** — must skip these and walk backward
- Data available from Jan 2020 onwards
- Some working days may be missing — walk backward to find nearest available
- Per SBI, use rates for ₹10-20 lakh transaction range (these ARE the reference rates in CSV)

### "Last Working Day of Previous Month" Algorithm
```python
def get_last_working_day_prev_month(d: date) -> date:
    first_of_month = d.replace(day=1)
    last_of_prev = first_of_month - timedelta(days=1)
    # Walk backward to skip weekends
    while last_of_prev.weekday() >= 5:  # 5=Sat, 6=Sun
        last_of_prev -= timedelta(days=1)
    return last_of_prev

def get_sbi_rate(d: date, currency='USD') -> float:
    rate_date = get_last_working_day_prev_month(d)
    # Look up rate_date in CSV cache
    # If not found (holiday), walk backward day by day
    # If TT BUY is 0.00, also walk backward
    # Return the rate
```

---

## 5. STOCK DATA — YFINANCE

### Company Info
```python
import yfinance as yf
t = yf.Ticker("QCOM")
info = t.info
# info['longName']      → "QUALCOMM Incorporated"
# info['address1']      → "5775 Morehouse Drive"
# info['city']          → "San Diego"
# info['state']         → "CA"
# info['zip']           → "92121"
# info['country']       → "United States"
# info['quoteType']     → "EQUITY" or "ETF"
```

### Historical Prices (for peak value + closing balance)
```python
hist = t.history(start="2024-01-01", end="2025-01-01")
# hist.index = DatetimeIndex
# hist['Close'] = daily closing prices
```

### Dividends
```python
divs = t.dividends
# Series: DatetimeIndex → dividend_per_share (float)
# Filter for calendar year: divs['2024']
```

### Ticker Mapping
- US stocks: ticker as-is (e.g., `QCOM`)
- VWRA (Vanguard FTSE All-World Acc ETF): use `VWRA.L` (London Stock Exchange)
- User should be able to specify the Yahoo ticker if different

---

## 6. DATA MODEL (JSON)

```json
{
  "calendar_year": 2024,
  "stocks": [
    {
      "id": "stock_uuid",
      "ticker": "QCOM",
      "yahoo_ticker": "QCOM",
      "currency": "USD",
      "skip_dividends": false,
      "company_info": {
        "country_code": "2-UNITED STATES OF AMERICA",
        "name": "Qualcomm Incorporated",
        "address": "5775 Morehouse Drive, San Diego, CA",
        "zip": "92121",
        "nature": "Company"
      },
      "dividends": [
        {
          "id": "div_uuid",
          "ex_date": "2024-03-01",
          "amount": 0.75
        }
      ],
      "lots": [
        {
          "id": "lot_uuid",
          "buy_date": "2021-08-20",
          "quantity": 10.5,
          "buy_price": 143.50,
          "sells": [
            {
              "id": "sell_uuid",
              "sell_date": "2024-05-15",
              "quantity": 5.0,
              "sell_price": 175.00
            }
          ]
        }
      ]
    }
  ],
  "overrides": {
    "lot_uuid": {
      "initial_value": null,
      "peak_value": null,
      "closing_balance": null,
      "total_dividends": null,
      "sale_proceeds": null
    }
  },
  "sbi_rate_overrides": {
    "2024-07-31_USD": 83.95
  }
}
```

---

## 7. PROJECT STRUCTURE

```
itr_fa/
├── app.py                          # Flask entry point + all routes
├── config.py                       # Constants, data paths
├── requirements.txt                # Dependencies
├── README.md                       # Setup & usage instructions
├── APPLICATION_PLAN.md             # This file
│
├── core/
│   ├── __init__.py
│   ├── sbi_rates.py                # SBI TTBR fetch, cache, lookup
│   ├── stock_data.py               # yfinance wrapper
│   ├── calculator.py               # A3 row calculations
│   └── excel_export.py             # openpyxl Excel generation
│
├── data/                           # Runtime data (gitignored)
│   ├── sbi_rates_cache.json        # Cached SBI rates
│   └── portfolios/                 # Saved user portfolios
│       └── portfolio_CY2024.json
│
├── static/
│   ├── css/
│   │   └── style.css               # Dark-mode premium UI
│   └── js/
│       └── app.js                  # Frontend SPA logic
│
└── templates/
    └── index.html                  # Main Jinja2 template
```

---

## 8. BACKEND API ROUTES

| Method | Route | Purpose |
|---|---|---|
| GET | `/` | Serve main HTML page |
| POST | `/api/lookup-stock` | Fetch company info by ticker via yfinance |
| GET | `/api/sbi-rate?date=YYYY-MM-DD&currency=USD` | Get SBI TTBR for last working day of prev month |
| GET | `/api/stock-price?ticker=QCOM&date=YYYY-MM-DD` | Get close price on a date |
| POST | `/api/calculate` | Calculate all A3 rows from portfolio JSON |
| POST | `/api/export-excel` | Generate and download Excel file |
| POST | `/api/save` | Save portfolio JSON to disk |
| GET | `/api/load?year=2024` | Load saved portfolio JSON |
| GET | `/api/list-saves` | List all saved portfolio files |
| POST | `/api/fetch-sbi-rates` | Bulk download & cache SBI rates CSV |
| GET | `/api/monthly-rates?year=2024&currency=USD` | Fetch all 12 monthly rates for a specific calendar year |
| POST | `/api/save-manual-rate` | Save manual user override for a specific month's rate |
| GET | `/api/dividends?ticker=QCOM&year=2024` | Auto-fetch dividends for UI pre-population |

---

## 9. FRONTEND UI DESIGN

### Layout
- **Header**: App title, year selector dropdown, action buttons (Save, Load, Import Previous Year, Export Excel)
- **Add Stock Section**: Ticker input + Lookup button
- **Stock Cards** (one per stock): Collapsible, containing:
  - Company info fields (editable)
  - Lots table (buy date, qty, price, add/remove)
  - Per-lot sells table (sell date, qty, price, add/remove)
  - **Dividends section**: Table of auto-fetched dividends, fully editable by user
  - Skip dividends toggle
- **Results Section**: Full A3 table preview
  - Each calculated cell has a pencil icon → click to override manually
  - Override values shown in amber; calculated in default color
- **Per-Stock Dividend Summary**: Displays an aggregated total of dividends calculated for each stock
- **SBI Rates Panel**: Collapsible section showing all 12 monthly rates for any selected year, allowing full manual edits

### Styling
- Dark mode: bg `#0f1117`, cards `#1a1d28`, accent `#6366f1` (indigo)
- Font: Inter from Google Fonts
- Glassmorphism cards, smooth transitions, micro-animations
- Responsive layout

---

## 10. PEAK VALUE CALCULATION — DETAILED ALGORITHM

Peak value is the maximum INR value of the investment at any point during the calendar year.

```python
def calculate_peak_value(lot, sells_in_cy, daily_prices, sbi_rates, cy_year):
    peak = 0
    initial_qty = lot['quantity']
    # Sort sells by date
    sorted_sells = sorted(sells_in_cy, key=lambda s: s['sell_date'])
    sell_idx = 0
    qty_remaining = initial_qty

    for date, row in daily_prices.iterrows():
        trading_date = date.date()
        # Apply any sells that happened on or before this date
        while sell_idx < len(sorted_sells):
            sd = parse_date(sorted_sells[sell_idx]['sell_date'])
            if sd <= trading_date:
                qty_remaining -= sorted_sells[sell_idx]['quantity']
                sell_idx += 1
            else:
                break

        if qty_remaining <= 0:
            break

        close_price = row['Close']
        # SBI rate for this month = rate on last working day of previous month
        ttbr = get_sbi_rate_for_date(trading_date, sbi_rates)
        value_inr = close_price * qty_remaining * ttbr

        if value_inr > peak:
            peak = value_inr

    return round(peak)
```

**Important**: Since TTBR only changes monthly (it's the last working day of the *previous* month), optimize by caching monthly TTBR values.

---

## 11. DIVIDEND CALCULATION — DETAILED ALGORITHM

```python
def calculate_dividends(lot, all_lots_for_ticker, sells, dividends_series, sbi_rates, cy_year):
    total_div_inr = 0
    # Filter dividends for the calendar year
    cy_divs = dividends_series[f'{cy_year}']

    for ex_date, div_per_share in cy_divs.items():
        ex_dt = ex_date.date()
        # Calculate qty held on ex_date for THIS LOT
        # (must check if lot was bought before ex_date)
        if parse_date(lot['buy_date']) > ex_dt:
            continue  # lot didn't exist yet
        qty = lot['quantity']
        for sell in lot.get('sells', []):
            if parse_date(sell['sell_date']) <= ex_dt:
                qty -= sell['quantity']
        if qty <= 0:
            continue
        ttbr = get_sbi_rate_for_date(ex_dt, sbi_rates)
        total_div_inr += div_per_share * qty * ttbr

    return round(total_div_inr)
```

---

## 12. COUNTRY CODE MAPPING

```python
COUNTRY_CODES = {
    "United States": "2-UNITED STATES OF AMERICA",
    "United Kingdom": "3-UNITED KINGDOM",
    "Ireland": "4-IRELAND",
    "Germany": "5-GERMANY",
    "Japan": "6-JAPAN",
    "Canada": "7-CANADA",
    "Singapore": "8-SINGAPORE",
    # Add more as needed — user can override
}
```

---

## 13. IMPLEMENTATION ORDER & STATUS

### Phase 1: Project Setup ✅
- [x] Create `requirements.txt`
- [x] Create `config.py`
- [x] Create `core/__init__.py`
- [x] Create `README.md`

### Phase 2: SBI Rates Module ✅
- [x] `core/sbi_rates.py` — download CSV, parse, cache, lookup
- [x] Handle TT BUY = 0.00 (walk backward)
- [x] Handle missing dates (walk backward)
- [x] Manual override support
- [x] Support USD and GBP currencies

### Phase 3: Stock Data Module ✅
- [x] `core/stock_data.py` — yfinance wrapper
- [x] Company info fetcher
- [x] Historical price fetcher
- [x] Dividend fetcher
- [x] Ticker mapping (VWRA → VWRA.L)

### Phase 4: Calculator Module ✅
- [x] `core/calculator.py` — all 12 columns
- [x] Initial value calculation
- [x] Peak value calculation (day-by-day)
- [x] Closing balance calculation
- [x] Dividend calculation
- [x] Sale proceeds calculation
- [x] FIFO sell handling
- [x] Override support

### Phase 5: Excel Export ✅
- [x] `core/excel_export.py` — openpyxl
- [x] Formatted A3 table with proper headers
- [x] Indian comma style for INR values
- [x] Column widths, borders, styling

### Phase 6: Flask Backend ✅
- [x] `app.py` — all API routes
- [x] Stock lookup endpoint
- [x] SBI rate endpoint
- [x] Calculate endpoint
- [x] Save/load endpoints
- [x] Export Excel endpoint

### Phase 7: Frontend ✅
- [x] `templates/index.html`
- [x] `static/css/style.css` — premium dark mode
- [x] `static/js/app.js` — SPA logic
- [x] Year selector
- [x] Stock entry + auto-lookup
- [x] Transaction tables (buy/sell)
- [x] Dividend section
- [x] A3 results table with override
- [x] **Monthly SBI Rates Manager**
- [x] Save/load/import buttons
- [x] Excel export button
- [x] **Official Currency Symbols (₹, $, £, etc.)**

### Phase 8: Testing ✅
- [x] Test QCOM lookup — verified auto-fetches company info correctly
- [x] Test SBI rate fetch — CSV download and caching works
- [x] Test full calculation with QCOM screenshot data
- [x] Test with VWRA.L
- [x] Test year import flow
- [x] Test Excel output format

---

## 14. DEPENDENCIES

```
flask>=3.0
yfinance>=0.2
openpyxl>=3.1
requests>=2.31
```

Install: `pip install -r requirements.txt`
Run: `python app.py`

---


The user provided an A3 screenshot with these rows (CY 2024 likely):

| Sl | Country | Entity | Address | Zip | Nature | Acquire Date | Initial ₹ | Peak ₹ | Closing ₹ | Dividends ₹ | Sale ₹ |
|---|---|---|---|---|---|---|---|---|---|---|---|


---

## 16. NOTES FOR CONTINUING AGENTS

1. **SBI CSV is large** (~1600 lines for USD). Download once and cache in `data/sbi_rates_cache.json`.
2. **yfinance may be slow** on first call. Cache company info in the portfolio JSON.
3. **Peak value is the most complex calculation** — must iterate daily prices × qty held × monthly TTBR.
4. **TT BUY = 0.00** in the CSV means it's a weekend/holiday — always skip and walk backward.
5. **Date format in CSV** is `YYYY-MM-DD HH:MM` — parse only the date part.
6. **The user wants ALL calculated fields to be overridable** — store overrides in the JSON data model.
7. **Import previous year**: Load last year's JSON, carry forward unsold lots, reset CY-specific fields.
8. **GBP rates** needed for VWRA.L — same CSV format, different file: `SBI_REFERENCE_RATES_GBP.csv`.
