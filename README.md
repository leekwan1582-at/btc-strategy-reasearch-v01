# OptiTrader Lab

> **A lightweight, AI-assisted web portal that turns strategy tuning into tracked, repeatable experiments.**

[](https://opensource.org/licenses/MIT)
[](https://react.dev/)
[](https://nodejs.org/)
[](https://www.mongodb.com/)
[](https://ai.google.dev/)
[](http://makeapullrequest.com)

`[SCREENSHOT: optitrader-lab-hero-dashboard]`

---

## Why OptiTrader Lab?

### The Problem
Solo quant traders and systematic developers typically rely on ad-hoc Python scripts, scattered Jupyter notebooks, and manual CLI parameter sweeps to tune trading strategies. Over time, this workflow becomes:

* **Tedious:** Manually editing indicator periods and re-running scripts for every hypothesis slows down research iteration.
* **Untraceable:** Without a centralized experiment ledger, it is easy to lose track of which exact dataset hash, fee assumption, or parameter grid produced a given Sharpe ratio.
* **Error-Prone:** Copy-pasting parameters across scripts introduces lookahead leaks, silent `NaN` propagation, and unrecorded out-of-sample peeking.

### Who It's For
**OptiTrader Lab** is purpose-built for **solo systematic traders and quant developers who backtest in Python** (`vectorbt`, `TA-Lib`, `pandas`) and want a structured, auditable web workbench that bridges natural-language strategy ideation with deterministic parameter optimization.

---

## Key Features

* **Natural-Language Assistant with Gemini Function Calling:** Describe a trading hypothesis or optimization grid in plain English. Powered by **Google Gemini (`gemini-1.5-pro`)** structured tool use, the assistant translates your prompt into a validated JSON strategy specification (`StrategySpec`) and launches optimization runs automatically—without ever letting the LLM write raw execution code.
* **Non-Blocking Background Worker Thread Execution:** Heavy Cartesian parameter grid evaluations and Monte Carlo drawdown simulations are offloaded to a dedicated Node.js `worker_threads` pool (`runBacktestWorker()`), streaming real-time progress updates (`QUEUED` → `RUNNING` → `COMPLETED`) to the UI without blocking the Express API.
* **Complete Historical Run Ledger & Metric Visualizers:** Every trial is hashed and stored in MongoDB alongside its dataset provenance (`data_id`) and point-in-time split boundaries (`60% Research / 20% Validation / 20% Holdout`). Compare runs side by side with interactive **Sharpe ratio**, **Max Drawdown**, **CAGR**, equity curves, and **parameter sensitivity heatmaps**.
* **Deterministic Compiler & Option A Guardrails:** Enforces TA-Lib indicator family whitelists (`trend`, `mean_reversion`, `momentum`, `volatility`), strict parameter bounds, and cross-cutting volume confirmation rules before any backtest executes.

`[SCREENSHOT: natural-language-gemini-assistant]`

`[SCREENSHOT: parameter-heatmap-and-run-ledger]`

---

## Tech Stack

| Layer | Technology | Role |
| :--- | :--- | :--- |
| **Frontend (`client/`)** | React 18, Vite, Tailwind CSS, Chart.js / Recharts | Interactive chat workbench, CSV data uploader, parameter grid builder, equity curves, and sensitivity heatmaps |
| **Backend (`server/`)** | Node.js, Express, `worker_threads`, JWT | REST API routing, authentication middleware (`authenticateToken`), and background worker pool orchestration |
| **Database (`server/models/`)** | MongoDB, Mongoose | Persistent storage for `Users`, versioned `Strategies`, and historical optimization `Runs` |
| **AI Integration (`server/services/`)** | Google Gemini API (`gemini-1.5-pro`) | Structured function calling (`geminiAssistant.js`) to convert natural language prompts into deterministic strategy specs and grid configs |

---

## Installation & Quick Start

### Prerequisites
* **Node.js** `v20.11+` and **npm** `v10+`
* **MongoDB** (Local instance or [MongoDB Atlas](https://www.mongodb.com/atlas) URI)
* **Google Gemini API Key** (`GEMINI_API_KEY` from [Google AI Studio](https://aistudio.google.com/))

### 1. Clone the Repository

```bash
git clone https://github.com/leekwan/optitrader-lab.git
cd optitrader-lab
```

### 2. Configure Environment Variables (`.env`)

Create a `.env` file in the `server/` directory:

```ini
# server/.env
PORT=5000
NODE_ENV=development
HOST=0.0.0.0

# MongoDB Connection String
MONGODB_URI=mongodb+srv://<username>:<password>@cluster0.mongodb.net/optitrader_lab?retryWrites=true&w=majority

# Google Gemini API Configuration
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-1.5-pro

# JWT Authentication Secret
JWT_SECRET=your_super_secret_jwt_key_here
```

### 3. Install Dependencies

Install packages for the root workspace, Express backend, and React frontend:

```bash
npm install
cd server && npm install
cd ../client && npm install
cd ..
```

### 4. Start the Development Servers

Launch both the Express API (`http://localhost:5000`) and Vite React client (`http://localhost:5173`) concurrently:

```bash
npm run dev
```

---

## Usage Example

Once the application is running at `http://localhost:5173`, you can execute a complete strategy optimization experiment in three steps:

### Step 1: Upload an OHLCV Dataset CSV
Navigate to the **1 Data Layer** tab and click **Upload CSV** to ingest your daily (`D1`) price history (columns: `date,open,high,low,close,volume[,benchmark]`). OptiTrader Lab automatically computes the dataset SHA-256 `data_id` and freezes the **60% Research / 20% Validation / 20% Holdout** split boundaries.

`[SCREENSHOT: step1-csv-upload-and-splits]`

### Step 2: Ask the AI Assistant to Optimize Moving Average Crossovers
Open the **AI Chatbot** tab and enter a natural-language optimization prompt:

```text
Create a trend-following BTC D1 strategy where EMA(fast) crosses above EMA(slow), confirmed by close > vwap(20), and run a parameter grid optimization for fast EMA [10, 15, 20, 25] and slow EMA [50, 100, 150, 200].
```

Click **Generate**. `geminiAssistant` translates your request into a structured function call, validates the indicator rules, and spawns `runBacktestWorker()` in the background.

### Step 3: Review the Ranked Trial Matrix & Heatmaps
As the worker thread completes the grid sweep, inspect the **Ranked Trial Matrix**, **Equity Curve**, and **Parameter Sensitivity Heatmap** in the workbench:

| Rank | Trial ID | Fast EMA | Slow EMA | VWAP Gate | Sharpe (Research) | CAGR | Max Drawdown | Trades |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **#1** | `run_01_ema20_100` | `20` | `100` | `20` | **1.68** | **+44.2%** | `-16.4%` | `38` |
| **#2** | `run_02_ema15_100` | `15` | `100` | `20` | **1.54** | **+41.0%** | `-18.1%` | `46` |
| **#3** | `run_03_ema25_150` | `25` | `150` | `20` | **1.39** | **+35.8%** | `-15.2%` | `29` |

From the top-ranked trial, click **Audit in Step 3 →** to run the 200-iteration Monte Carlo trade reshuffle and 2× cost sensitivity check before promoting the winner to **Step 4** for one-shot holdout verification.

`[SCREENSHOT: ranked-trial-matrix-and-equity-curve]`

---

## Documentation

For detailed walkthroughs, architecture diagrams, and API reference guides, see the project documentation:

* **User Guide:** [docs/user-guide.md](https://www.google.com/search?q=docs/user-guide.md) — Step-by-step instructions for dataset uploads, prompt patterns, robustness checks, FAQs, and troubleshooting.
* **Technical Documentation:** [docs/technical.md](https://www.google.com/search?q=docs/technical.md) — System architecture, Mongoose schemas, worker thread lifecycle (`runBacktestWorker`), Gemini function calling (`geminiAssistant`), JWT middleware (`authenticateToken`), and Jest/Vitest test suites.

---

## Project Structure

```text
btc-strategy-reasearch-v01/
├── client/                  # React + Vite + Tailwind CSS frontend UI & chart visualizers
├── server/                  # Express REST API, controllers, JWT middleware & worker threads
│   ├── models/              # Mongoose schemas (User, Strategy, Run)
│   ├── services/            # geminiAssistant.js (Gemini 3.5-flash-lite function calling)
│   └── workers/             # runBacktestWorker.js & backtestThread.js
├── docs/
│   ├── user-guide.md        # End-user handbook for solo quant traders
│   └── technical.md         # Developer architecture & API reference
└── package.json             # Concurrent dev scripts & test runners
```

---

## License

This project is licensed under the **MIT License** — see the `LICENSE` file for details.

## Author

**Lee Kwan**  
*Quantitative System Engineer & Full-Stack Developer*
