# AirSense – Air Quality Analytics & Forecasting Platform

AirSense is an end-to-end air quality monitoring, comparative analysis, and time-series forecasting platform. It pulls ambient pollution data (PM2.5 and PM10) from multiple public and station APIs, normalizes and cleans the data using outlier trimming and weighted consensus, stores it in MySQL, and serves interactive visualizations, statistical comparisons, and predictive forecasts.

The project also includes a local LLM agent (powered by Ollama) that acts as an analytical copilot: it interprets natural language questions about air quality, verifies what tools are permitted through a capability critic, plans tool executions, and runs multi-city analyses automatically.

---

## Screenshots

### Landing & Workspace Overview
![AirSense Landing Page](screenshots/Screenshot%202025-12-17%20025015.png)

![AirSense Workspace & Live Monitoring](screenshots/Screenshot%202025-12-17%20030511.png)

### Multi-City Comparative Analytics
![City Comparison Overview](screenshots/Screenshot%202025-12-17%20025247.png)

![City Comparison Detailed Metrics](screenshots/Screenshot%202025-12-17%20025302.png)

### Time-Series Forecasting (SARIMAX Engine)
![Forecasting Interface](screenshots/Screenshot%202025-12-17%20025447.png)

### Automated PDF Export
![Generated Report Summary](screenshots/Screenshot%202025-12-17%20025501.png)

![Generated Report Visuals](screenshots/Screenshot%202025-12-17%20025522.png)

---

## What the System Does

### 1. Multi-Source Ingestion & Robust Aggregation
Most single air quality APIs suffer from sensor downtime, geographical gaps, or reporting delays. AirSense queries multiple upstream providers:
- **Open-Meteo Air Quality API** (hourly PM2.5 and PM10 based on coordinates)
- **OpenAQ** (global open monitoring stations)
- **IQAir** (web scraper and feed parser)
- **WAQI (World Air Quality Index)** (station feed and fallback parser)

Incoming readings pass through geocoding (Nominatim / Open-Meteo geocoding to resolve city names to coordinates), outlier removal (MAD / IQR trimming to discard erroneous sensor spikes), and weighted aggregation before being saved to MySQL with duplicate resolution (`ON DUPLICATE KEY UPDATE`).

### 2. Multi-City Comparison & Ranking
- Compares hourly pollutant distributions across multiple cities over custom time windows.
- Calculates core summary metrics: mean, median, min, max, standard deviation, and AQI severity bands (Good, Moderate, Unhealthy for Sensitive Groups, Unhealthy, Hazardous).
- Produces ranking tables (cleanest vs. most polluted locations) to quickly identify hotspots.

### 3. Machine Learning Forecasting
- **SARIMAX (Seasonal AutoRegressive Integrated Moving Average)**:
  - Default order `(1, 1, 1)` with daily seasonal periodicity `(1, 0, 1, 24)` to capture diurnal traffic and weather fluctuations.
  - Automatic time-series frequency alignment (`asfreq('H')`) and gap interpolation.
  - Backtesting validation reporting MAE (Mean Absolute Error) and RMSE (Root Mean Squared Error).
  - Serializes trained weights into `backend/models/<city>_sarimax.joblib` for fast inference.
- **Prophet Support**: Optional modular Prophet implementation for trend changepoint analysis.
- Multi-city forecast batching to forecast multiple target cities simultaneously.

### 4. Natural Language Analytical Agent (MCP Architecture)
Users can ask plain-English questions such as *"Compare air pollution in London and Paris over the last 5 days and forecast London for the next 48 hours"*.
- **Stage 1 (Capability Critic)**: Uses a local Ollama model (default `gemma3:4b`) to inspect user intent, verify tool safety, and filter out irrelevant requests (e.g. general chit-chat, poetry, blog generation).
- **Stage 2 (Planner)**: Produces an actionable JSON execution plan strictly mapped to backend tools (`scrape_city`, `compare_cities`, `forecast_city`, `forecast_multi`).
- **Stage 3 (Executor)**: Sequentially executes the pipeline, enforces user tier permissions, and returns structured data and execution logs back to the frontend.

### 5. Authentication & Tiered Access Control
- JWT-based authentication with bcrypt password hashing and refresh token rotation.
- Configurable subscription tiers (`free`, `pro`, `enterprise`):
  - **Free**: 7-day query window, maximum 3 comparison cities, 3-day forecast horizon.
  - **Pro**: 30-day query window, 10 comparison cities, 14-day forecast horizon, PDF export.
  - **Enterprise**: 90-day query window, unlimited cities, 30-day forecast horizon, full LLM agent access.

### 6. Automated PDF Reporting
- Generates publication-ready PDF reports on the fly using ReportLab.
- Embeds statistical summary tables, data coverage matrices, and Recharts/matplotlib snapshots.
- Includes browser-native print stylesheets for clean in-browser PDF export.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| **Frontend** | React 19, Vite, Tailwind CSS, Recharts, Framer Motion, OGL / WebGL (LiquidEther) |
| **Backend** | Python 3.10+, FastAPI, Uvicorn, SQLAlchemy ORM, PyMySQL |
| **Database** | MySQL 8.x |
| **Machine Learning** | Statsmodels (SARIMAX), Scikit-Learn, Prophet, Pandas, NumPy, Joblib |
| **LLM / Agent** | Ollama (`gemma3:4b` or compatible local LLM), JSON tool calling |
| **PDF Generation** | ReportLab |

---

## Project Structure

```text
aqi-analysis/
├── backend/
│   ├── app/
│   │   ├── core/            # Config, security, JWT auth, logging middleware
│   │   ├── routers/         # API endpoints (auth, compare, forecast, agent, report, health)
│   │   ├── services/        # Business logic, scrapers, SARIMAX engine, LLM client
│   │   │   └── fetchers/    # Upstream data sources (Open-Meteo, OpenAQ, IQAir, WAQI)
│   │   ├── db.py            # Database session & engine initialization
│   │   ├── models.py        # SQLAlchemy models (User, RefreshToken, measurements)
│   │   ├── schemas.py       # Pydantic validation schemas
│   │   └── main.py          # FastAPI application entrypoint
│   ├── models/              # Serialized joblib models
│   ├── requirements.txt     # Python backend dependencies
│   └── .env.example         # Sample environment variables
├── frontend/
│   ├── src/
│   │   ├── components/      # UI components (charts, forecast, comparison, assistant)
│   │   ├── contexts/        # Auth & theme context providers
│   │   ├── pages/           # Landing, Workspace, Signin, Signup, Print reports
│   │   ├── api.js           # Axios client configured with JWT interceptors
│   │   └── main.jsx         # React application root
│   ├── package.json
│   └── vite.config.js
├── screenshots/             # Interface and report screenshots
├── .gitignore
└── README.md
```

---

## Getting Started

### Prerequisites
- **Python 3.10+**
- **Node.js 18+** and **npm**
- **MySQL 8.0+**
- **Ollama** (optional, required if using the LLM agent feature)

---

### 1. Database Setup

Create a MySQL database named `airq`:

```sql
CREATE DATABASE airq CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

### 2. Backend Setup

1. Navigate to the backend directory:
   ```bash
   cd backend
   ```

2. Create and activate a Python virtual environment:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```

3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

4. Configure environment variables:
   ```bash
   cp .env.example .env
   ```
   Edit `.env` to match your MySQL credentials and desired secret keys:
   ```env
   MYSQL_HOST=127.0.0.1
   MYSQL_PORT=3306
   MYSQL_DB=airq
   MYSQL_USER=root
   MYSQL_PASSWORD=your_mysql_password
   JWT_SECRET=replace-with-a-secure-random-key
   OLLAMA_BASE_URL=http://127.0.0.1:11434
   OLLAMA_MODEL=gemma3:4b
   ```

5. Start the FastAPI development server:
   ```bash
   uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
   ```
   Interactive API documentation will be available at `http://localhost:8000/docs`.

---

### 3. Frontend Setup

1. In a separate terminal, navigate to the frontend directory:
   ```bash
   cd frontend
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Start the Vite dev server:
   ```bash
   npm run dev
   ```
   Open `http://localhost:5173` in your browser.

---

### 4. (Optional) Ollama Agent Setup

If you want to use the Enterprise AI assistant:
1. Install [Ollama](https://ollama.ai/).
2. Pull the model configured in your `.env` (default is `gemma3:4b`):
   ```bash
   ollama pull gemma3:4b
   ```
3. Make sure Ollama is running (`ollama serve`).

---

## Primary API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/auth/register` | Register a new user account |
| `POST` | `/auth/login` | Authenticate user & return JWT token |
| `GET` | `/auth/me` | Fetch authenticated user profile & subscription tier |
| `POST` | `/scrape` | Fetch, clean, and persist live air quality data for a city |
| `GET` | `/compare` | Fetch comparative stats and time series for multiple cities |
| `GET` | `/forecast` | Generate SARIMAX time-series forecast for a single city |
| `POST` | `/forecast/multi` | Batch forecast across multiple cities |
| `POST` | `/agent/plan` | Run natural language query through the critic + planner agent |
| `POST` | `/agent/execute` | Execute an approved agent tool plan |
| `GET` | `/report/pdf` | Download auto-generated ReportLab PDF analysis |
| `GET` | `/health` | Service health status check |

---

## Author & Repository

- **Author**: Kaustubh Srivastava ([@kaustubh-28](https://github.com/kaustubh-28))
- **Repository**: [https://github.com/kaustubh-28/aqi-analysis](https://github.com/kaustubh-28/aqi-analysis)

---

## License

This project is open-source and available under the [MIT License](LICENSE).
