# 📈 AI-Powered Stock Market Assistant
### Portfolio Tracking & Optimization Tool

A full-stack web application that lets a small group of friends each track their own stock portfolio, get real-time prices, run Modern Portfolio Theory (MPT) optimization, and ask an AI assistant questions about their holdings — all in plain English.

---

## ✨ Features

| Feature | Description |
|---|---|
| 🔐 Multi-user auth | Each user has a separate account and private portfolio |
| 📂 CSV import | Upload holdings via CSV (ticker, shares, cost basis) |
| 💰 Live prices | Alpha Vantage API fetches real prices, cached daily in Postgres |
| 📊 Performance chart | Portfolio value over time as a line chart |
| 🥧 Allocation pie | Current holdings breakdown by market value |
| 🔴 Drift alerts | Warns when a ticker drifts 10%+ from its optimal weight |
| 📐 MPT Optimization | Max-Sharpe and Min-Volatility allocations + efficient frontier |
| 🤖 AI Chat | Ask Gemini why a rebalancing is suggested, in plain language |

---

## 🗂 Project Structure

```
portfolio-app/
├── backend/                  # Python FastAPI backend
│   ├── app/
│   │   ├── core/
│   │   │   ├── database.py       # Postgres connection & session
│   │   │   ├── security.py       # JWT auth, password hashing
│   │   │   ├── deps.py           # Auth dependency injection
│   │   │   ├── csv_parser.py     # Holdings CSV validator
│   │   │   ├── alpha_vantage.py  # Alpha Vantage API client
│   │   │   ├── price_sync.py     # Price fetch & cache logic
│   │   │   ├── scheduler.py      # Daily price sync job
│   │   │   └── mpt.py            # MPT optimization engine
│   │   ├── models/
│   │   │   ├── models.py         # SQLAlchemy DB models
│   │   │   └── schemas.py        # Pydantic request/response schemas
│   │   ├── routers/
│   │   │   ├── auth.py           # /auth/signup, /auth/login
│   │   │   ├── holdings.py       # /holdings/ CRUD + CSV upload
│   │   │   ├── portfolio.py      # /portfolio/value, /history, /optimize
│   │   │   ├── alerts.py         # /alerts/ drift detection
│   │   │   └── ai.py             # /ai/chat Gemini integration
│   │   └── main.py               # App entrypoint, router wiring
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/                 # React frontend
│   ├── src/
│   │   ├── App.jsx               # Router, navbar, private routes
│   │   ├── Login.jsx             # Signup / login page
│   │   ├── Dashboard.jsx         # Holdings, charts, alerts
│   │   ├── Optimize.jsx          # MPT results & frontier chart
│   │   ├── Chat.jsx              # AI chat interface
│   │   └── api.js                # Axios client with JWT
│   └── Dockerfile
├── docker-compose.yml        # Orchestrates all 3 services
├── .env.example              # Environment variable template
└── sample_holdings.csv       # Example CSV for testing
```

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12, FastAPI |
| Database | PostgreSQL 16 |
| Auth | JWT (python-jose + passlib/bcrypt) |
| Market data | Alpha Vantage API |
| Optimization | NumPy, SciPy (mean-variance MPT) |
| AI layer | Google Gemini API |
| Frontend | React 18, Recharts |
| Containerization | Docker + Docker Compose |

---

## ⚙️ Prerequisites

Before running this project you need:

1. **Docker Desktop** — [docker.com/products/docker-desktop](https://docker.com/products/docker-desktop)
   - Windows users: requires WSL 2 (the installer will guide you)
   - Must be fully started (whale icon in taskbar must be **still**, not spinning) before running any commands

2. **VS Code** — [code.visualstudio.com](https://code.visualstudio.com)
   - Optional: install the **Docker extension** by Microsoft for a visual container view

3. **API Keys** (free tiers are enough):
   - **Alpha Vantage** — [alphavantage.co](https://www.alphavantage.co) → Get Free API Key (25 requests/day)
   - **Google Gemini** — [aistudio.google.com](https://aistudio.google.com) → click **Get API key** → **Create API key**. Needed only for the AI Chat feature.

---

## 🚀 Getting Started

### 1. Unzip and open the project
Unzip `portfolio-app-phase5.zip`, then in VS Code:
**File → Open Folder → select the `portfolio-app` folder**

### 2. Open the terminal
Press **Ctrl+`** in VS Code to open the terminal.

### 3. Navigate to the project folder
```bash
cd "C:\path\to\portfolio-app"
```

### 4. Create your .env file
**Windows:**
```
copy .env.example .env
```
**Mac/Linux:**
```
cp .env.example .env
```

### 5. Fill in your API keys
Open the `.env` file in VS Code and edit it:
```
JWT_SECRET_KEY=any-long-random-string-you-make-up
ALPHA_VANTAGE_API_KEY=your-alpha-vantage-key-here
GEMINI_API_KEY=your-gemini-key-here
```
Save the file (**Ctrl+S**).

### 6. Build and run
```
docker compose up --build
```
First run takes **2–3 minutes** — Docker is downloading and installing everything.
You'll see lots of logs scrolling. That's normal.

### 7. Open the app
When you see `Compiled successfully` and `Application startup complete` in the logs:

| URL | What it is |
|---|---|
| http://localhost:3000 | The app — open this in your browser |
| http://localhost:8000/docs | Backend API docs (Swagger UI) |

---

## 📋 How to use it

1. **Sign up** — create an account at localhost:3000
2. **Upload holdings** — go to Dashboard, upload `sample_holdings.csv` (or your own)
3. **Refresh prices** — click "Refresh prices" to fetch current market data from Alpha Vantage
4. **View your portfolio** — see cost basis, market value, gain/loss, charts
5. **Optimize** — go to the Optimize tab, click "Run optimization" to see MPT suggestions
6. **Ask AI** — go to AI Chat, ask anything: *"Why does max Sharpe suggest selling AAPL?"*

---

## 📁 Sample CSV format

The CSV you upload must have exactly these three columns:

```csv
ticker,shares,cost_basis
AAPL,10,150.25
MSFT,5,300.00
GOOGL,3,2750.50
```

| Column | Description |
|---|---|
| `ticker` | Stock symbol (e.g. AAPL, MSFT) |
| `shares` | Number of shares you own |
| `cost_basis` | Price per share at the time you bought it |

---

## ⚠️ Alpha Vantage rate limits

The free tier allows **25 requests per day** and **5 per minute**. The app handles this automatically:
- Prices are fetched once daily by a background job (runs at 9pm UTC / ~4pm ET after market close)
- Only up to 20 unique tickers are synced per run to stay under the daily cap
- All dashboard loads read from the local Postgres cache — never hits the API on page load

---

## 🤖 AI Chat notes

The AI Chat feature calls the Gemini API (`gemini-flash-latest`) rather than pinning an exact
model version, since Google periodically retires older model names (this project previously broke
when `gemini-1.5-flash` was shut down). If you ever see a `404 models/... is not found` error from
Gemini, it means Google has retired the model this app points to — check
[ai.google.dev/gemini-api/docs/changelog](https://ai.google.dev/gemini-api/docs/changelog) for the
current recommended model and update `GEMINI_API_URL` in `backend/app/routers/ai.py`.

---

## 🔧 Common commands

| Command | What it does |
|---|---|
| `docker compose up --build` | First run, or after code changes |
| `docker compose up` | Start without rebuilding (faster) |
| `docker compose down` | Stop and remove containers |
| `docker compose logs backend` | See backend error logs |
| `docker compose logs frontend` | See frontend error logs |
| `docker compose ps` | Check if containers are running |

---

## 🐳 Docker troubleshooting

**"docker is not recognized"**
→ Docker Desktop is not open. Open it from the Start menu and wait for the whale icon to stop animating, then try again.

**"failed to connect to Docker API / npipe error"**
→ Same fix — Docker Desktop must be fully running before you open VS Code or run any command.

**localhost:3000 not opening**
→ Run `docker compose ps` and check all three services show `running`. If any show `exited`, run `docker compose logs frontend` or `docker compose logs backend` and paste the output.

**"version is obsolete" warning**
→ Harmless, can be ignored. Already fixed in this version.

---

## 👥 Adding more users

Each friend just needs to:
1. Visit your hosted URL (or `localhost:3000` if running locally)
2. Click **Sign up** and create their own account
3. Upload their own CSV — portfolios are completely separate per user

---

## 🏗 Built phase by phase

| Phase | What was built |
|---|---|
| 0 | Project scaffold — FastAPI, Postgres, React, Docker, JWT auth |
| 1 | CSV import, manual add/remove holdings |
| 2 | Alpha Vantage price pipeline, daily cache job, portfolio value |
| 3 | MPT engine — max Sharpe, min volatility, efficient frontier |
| 4 | Gemini AI chat — explains optimization in plain language |
| 5 | Charts, drift alerts, Docker polish, this README |

---

## 📄 License

MIT — free to use, modify, and share.
