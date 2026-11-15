# AI Trading Terminal: Plan

Working title. A stock terminal for Indian markets with a machine learning model and an LLM chat assistant built in. This is a learn-and-build project: every piece is here either because it teaches something or because it makes the demo real.

Plan revision 2 (15 Nov 2026): updated after checking data sources, licences, and terms. Changes are listed in section 15.

## 1. Ground rules

- **Market:** Indian stocks, Nifty 50 only.
- **No real money.** Version 1 has no orders at all. Paper (fake-money) trading arrives in Week 3.
- **Runs on your own machine.** Market data is licensed for personal use, so the app is not hosted publicly. The project is shown through the GitHub repo (code only, no price data committed) and a demo video.
- **Honest results.** The goal is a model that is tested properly, not one that claims to beat the market. Whatever numbers come out go in the README as they are.
- **Time:** 2 hours a day. One week for version 1, then four weeks of deeper phases.
- **Division of work:** Claude supplies setup, UI, and plumbing code. Aditya writes the ML and LLM parts, with explanation first and review after.
- Nothing in this project is investment advice.

## 2. Version 1 scope (Week 1)

**In**

- Five years of daily prices for the Nifty 50, stored in PostgreSQL
- Terminal screen: watchlist, candlestick chart, volume
- One ML model, tested on data it never saw, with its prediction shown per stock
- Chat box that answers questions by querying the project's own data
- README with real results, short demo video, resume bullets

**Out (comes back in Weeks 2 to 5)**

- Live prices, paper trading, strategy simulator, alerts, company announcements

## 3. How the pieces fit

```
  Browser (Next.js)
  ├── Watchlist + chart        ──►  GET /api/stocks, /api/candles
  ├── Prediction badge         ──►  GET /api/predictions
  └── Chat box                 ──►  POST /api/chat
                                         │
  Django + DRF  ─────────────────────────┤
  ├── Market data API                    │
  ├── Prediction API  ◄── saved model file (scikit-learn)
  └── Chat API ──► LLM ──► calls tools ──► same database
                                         │
  PostgreSQL: stocks, candles, features, predictions
        ▲
  Daily job: download prices ► compute inputs ► run model
```

## 4. Tech choices

| Part | Choice | Why |
|---|---|---|
| Backend | Django + DRF | Already known, so no time lost |
| Frontend | Next.js + TypeScript | Already known |
| Charts | TradingView `lightweight-charts` v5 | Free (Apache 2.0), built for candlesticks. Keep its TradingView attribution logo switched on; the licence requires the credit |
| Database | PostgreSQL | Already known, handles this data size easily |
| Stock list | Official Nifty 50 CSV from niftyindices.com | Gives symbol, company name, and industry; downloaded on Day 1, not typed by hand |
| Price data (v1) | `yfinance`, NSE tickers (e.g. `RELIANCE.NS`) | Free, no sign-up, prices already corrected for splits |
| Price data fallback 1 | `jugaad-data` (reads NSE directly) | Free, actively maintained, caches downloads. Prices are not corrected for splits, so that step would be ours |
| Price data fallback 2 | Zerodha Kite Connect paid plan | ₹500 a month per app, includes history and live data |
| Live prices (Week 4) | Zerodha Kite Connect paid plan | Local use only; see section 12 |
| ML | pandas + scikit-learn | The standard starting tools |
| LLM | Any LLM API that supports tool calling | Use whichever key is already available |
| Daily job | Django management command (cron later) | Simplest thing that works |

## 5. Data rules

These came out of the research and apply from Day 1.

- **Download once, keep it.** Yahoo throttles the free library when it is hit repeatedly. Fetch all 50 stocks in one batched call, save the raw result to a local file, and load the database from that file. After that, only fetch the newest day.
- **Use split-corrected prices.** When a company splits its shares, the raw price drops overnight without any real move. Uncorrected data would show these as huge moves and poison the model. Check two or three known splits by eye on Day 1.
- **Not every stock has five years.** The Nifty 50 list changes a couple of times a year, and some members listed recently. Use whatever history exists and record the start date per stock.
- **Today's list is a biased sample.** Stocks are in the index today partly because they did well. That flatters any "would this have made money" test. It barely affects Week 1's question, but it is noted in the README and handled in Week 3.
- **No price data in git.** Add data folders to `.gitignore` on Day 1.

## 6. Data model

| Table | Columns | Notes |
|---|---|---|
| `stocks` | symbol, name, sector, history_start | 50 rows |
| `candles` | stock, date, open, high, low, close, volume | One row per stock per day; unique on (stock, date) |
| `features` | stock, date, plus one column per model input | Computed from `candles` |
| `predictions` | stock, date, probability, model_version | What the model said, saved so it can be checked later |

## 7. The ML part

**The question the model answers**

> "Will this stock move more than usual tomorrow?"

Precisely: is tomorrow's price change (up or down, ignoring direction) bigger than the stock's typical daily change over the last 20 days? Answer: yes or no.

Why this question and not "will it go up?": direction is close to a coin flip and teaches little. The size of moves has real patterns (calm days follow calm days, wild days follow wild days), so the model has something genuine to learn and the test results mean something.

**Inputs (first set)**

- Price change over the last 1, 5, and 20 days
- How jumpy the stock has been over the last 5 and 20 days
- Today's volume compared with its 20-day average
- Today's high-to-low range compared with its usual range
- Day of the week

**Testing honestly**

- Split by time, never randomly: learn from roughly the first four years, test on the final year.
- Every input for a given day uses only information available at that day's close.
- Two "dumb guess" scores to beat first:
  1. Always answer the more common outcome
  2. "Tomorrow will be like today"
- Models, in order: logistic regression (simple and easy to understand), then random forest.
- Report: how often it is right, how often its "yes" calls are right, how many real "yes" days it catches, and all of that next to the dumb guesses.

**Done when:** the README has a results table with model vs. dumb guesses on the test year, whatever the numbers are.

## 8. The LLM part

**How it works:** the LLM never answers from its own memory about stock prices. It is given a small set of functions ("tools"), picks which to call, and writes its answer from what they return.

**Tools**

| Tool | What it does |
|---|---|
| `get_price_history(symbol, days)` | Recent candles for one stock |
| `screen_stocks(sector, min_change_pct, max_change_pct, period_days, min_volume_ratio)` | Finds stocks matching filters |
| `get_prediction(symbol)` | The model's latest call and its probability |
| `compare_stocks(symbols, period_days)` | Side-by-side returns and jumpiness |

**Rules given to the LLM**

- Use only numbers that came back from a tool.
- If the data does not cover the question, say so.
- State the date range the answer is based on.
- No buy or sell recommendations.

**Checking it:** a file of 10 questions with known correct answers (worked out by direct database query). Run all 10, record how many are right.

Sample questions:

1. Which IT stocks fell more than 3% this week?
2. Which stock had the highest volume compared to its average yesterday?
3. Compare HDFC Bank and ICICI Bank over the last 3 months.
4. What does the model say about Reliance for tomorrow?
5. What was TCS's price on a date outside our data? (Correct answer: "I don't have that.")

## 9. API endpoints

| Method | Path | Returns |
|---|---|---|
| GET | `/api/stocks` | Watchlist with latest close and daily change |
| GET | `/api/candles?symbol=&from=&to=` | Candles for the chart |
| GET | `/api/predictions?symbol=` | Latest model call |
| POST | `/api/chat` | LLM answer plus which tools it used |

## 10. Repo layout

```
trading-terminal/
├── PLAN.md
├── README.md
├── .gitignore                 # includes data/ and .env
├── docker-compose.yml         # PostgreSQL
├── data/                      # raw downloads, never committed
├── backend/
│   ├── config/                # Django settings
│   ├── market/                # stocks, candles, data download
│   ├── ml/                    # features, training, predictions
│   │   └── notebooks/         # exploration and results
│   └── assistant/             # LLM tools, chat endpoint, test questions
└── frontend/                  # Next.js app
```

## 11. Week 1, day by day

Each day is about 2 hours: a short lesson, then building.

### Day 1: Setup and data
- Lesson: how to read a candle; what volume tells you; what a split does to a price chart
- [ ] Repo, Django project, PostgreSQL running, `.gitignore` covering `data/`
- [ ] Load the official Nifty 50 list into `stocks`
- [ ] One batched download of five years of daily prices, saved to `data/` first, then loaded into `candles`
- [ ] Spot-check two or three known splits to confirm prices are corrected
- [ ] If the download is blocked or wrong, switch to fallback 1 before doing anything else
- **Done when:** a query shows each stock's row count and start date, with no gaps or duplicates

### Day 2: Terminal screen
- Lesson: how the terminal's parts talk to each other
- [ ] `/api/stocks` and `/api/candles`
- [ ] Watchlist on the left, candlestick chart with volume on the right (lightweight-charts v5; older tutorials use a different way of adding a series)
- [ ] TradingView attribution left visible
- **Done when:** clicking a stock in the watchlist redraws the chart

### Day 3: Model inputs
- Lesson: turning raw prices into something a model can learn from; why you set a dumb-guess score first
- [ ] Notebook: compute the inputs from section 7
- [ ] Compute the yes/no answer for every day
- [ ] Score both dumb guesses
- **Done when:** the `features` table is filled and the dumb-guess scores are written down

### Day 4: Train and test
- Lesson: learning vs. testing; how a model "cheats" by seeing the future
- [ ] Train logistic regression, then random forest
- [ ] Test on the final year, compare with dumb guesses
- [ ] Save the model; `/api/predictions`; badge on the terminal
- **Done when:** results table exists and the terminal shows a prediction per stock

### Day 5: Chat box, part 1
- Lesson: how an LLM uses tools instead of guessing
- [ ] `get_price_history` and `screen_stocks` tools
- [ ] `/api/chat` endpoint and chat panel in the UI
- **Done when:** "which IT stocks fell over 3% this week?" returns a correct answer

### Day 6: Chat box, part 2
- Lesson: how to check an LLM's answers and stop made-up ones
- [ ] `get_prediction` and `compare_stocks` tools
- [ ] The 10 test questions and a script that runs them
- **Done when:** the score out of 10 is recorded, and the out-of-range question gets "I don't have that"

### Day 7: Ship
- [ ] README: what it is, architecture diagram, results table, chat test score, known limits, what's next, how to run it locally
- [ ] 60 to 90 second demo video
- [ ] Repo public, with no price data in it
- [ ] Resume bullets with real numbers
- **Done when:** the project is on GitHub and on the resume
- No public hosting, so this is the lightest day and doubles as the buffer.

## 12. Weeks 2 to 5

| Week | Theme | Build | Learn |
|---|---|---|---|
| 2 | ML, properly | More inputs, a stronger model type, testing that repeatedly trains on the past and checks the following months | Why one model beats another; skill vs. luck; reading a model's mistakes |
| 3 | Does it make money on paper? | Simulator that replays stored daily history with the model's signals and trading costs; fake-money portfolio filled at end-of-day prices | How to judge a strategy; how test results mislead; the "today's list" bias from section 5 |
| 4 | Going live | Live prices from Kite Connect on the chart, on your machine only; alerts for unusual behaviour | Streaming data; spotting oddities without labelled examples |
| 5 | LLM, properly | Company announcements from NSE's official feeds, summarised per stock; search across them by meaning; bigger question set; final write-up | Feeding an LLM the right documents; measuring answer quality; watching a model after launch |

Each week ends with a working demo and an updated resume bullet.

**Week 3 and Week 4 constraints (from Zerodha's API terms)**

- Kite Connect live data cannot be shown to the public, and the terms prohibit using the API or its live data for virtual or mock trading apps. So paper trading (Week 3) runs only on stored daily history from the free source, and the Kite live feed (Week 4) is used only for the chart and alerts on your own machine. The two are kept separate.
- The terms also restrict building permanent copies of Kite data. Before paying in Week 4, read the current terms, and if local storage of live data is needed, ask Zerodha first. Week 4 can work without storing the live feed.
- Kite needs a fresh login every trading day. Week 4 includes a small login step for that.
- Cost: ₹500 for the month.

**Week 5 source**

- Primary: NSE's official RSS feeds for company announcements, results, and board meetings (free, public).
- Optional: news headlines with links from publisher RSS feeds. Store headline and link only, not article text.

## 13. Risks and fallbacks

| Risk | Fallback |
|---|---|
| Yahoo blocks or throttles the download | Batched one-time download and local file make this unlikely; otherwise `jugaad-data`, then Kite Connect |
| Split-corrected prices are wrong for a stock | Spot-check on Day 1; drop or fix that stock |
| A stock has under five years of history | Use what exists; note it |
| A day overruns | Day 7 is light and absorbs it |
| Model barely beats the dumb guess | Report it honestly; explaining why is a strong interview answer |
| LLM gives wrong numbers | Tighten the rules, add the failing question to the test file |
| Kite terms rule out something in Week 4 | Keep Week 4 to chart plus alerts with no storage, or use delayed daily data and build alerts on that |

## 14. Resume bullets (fill in after Day 7)

- Built an AI-assisted trading terminal for the Nifty 50 (Django, Next.js, PostgreSQL) with candlestick charting over ___ rows of market data
- Trained and evaluated a model predicting next-day large moves; reached ___ vs. a ___ baseline on a held-out year, with time-based testing to avoid leakage
- Built an LLM assistant with ___ tools over project data; ___/10 on a hand-checked question set, including refusing questions outside the data

## 15. What the research changed (15 Nov 2026)

| Checked | Finding | Change to plan |
|---|---|---|
| Free price data (`yfinance`) | Works for NSE tickers, but Yahoo rate-limits repeated use | Download once in a batch, save locally; added a second free fallback |
| `jugaad-data` | Maintained in 2026, reads NSE directly, has caching | Named as fallback 1 |
| Kite Connect pricing | Free plan has no market data; paid plan is ₹500 a month per app with history and live data | Kept as Week 4 feed and last-resort fallback |
| Kite Connect terms | No public display of live data; no mock-trading apps on the API or live data; daily login | App stays local; no public deploy; paper trading separated from Kite data; login step added |
| Nifty 50 list | Official CSV with symbol and industry; membership changes over time | Load from the CSV; handle short histories; note the bias |
| Chart library | Current major version is 5; attribution required | Use v5; keep the logo on |
| News for Week 5 | No dependable free per-company news API found; NSE publishes official announcement feeds | Week 5 built on NSE announcements, headlines optional |

Not researched because they are stable, well-established tools and methods: Django, Next.js, PostgreSQL, scikit-learn, time-based testing, and LLM tool calling.

Still to confirm by doing: that the Day 1 download works from your network, and Zerodha's position on storing Kite data locally (only matters in Week 4).

## 16. How we work

1. Plain-language explanation first, with a small example. Technical terms come after the idea is clear.
2. Aditya builds; Claude reviews and says what is wrong and what is good.
3. A few questions at the end of each day to check it stuck.
4. Tick the boxes in this file as days finish.
