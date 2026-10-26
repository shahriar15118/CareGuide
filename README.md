# API Sentinel — Intelligent API Monitoring & Alert System

API Sentinel is a demonstration dashboard for ingesting API observations, detecting common anomalies, generating concise explanations, and storing alerts in PostgreSQL.

> **Demo-data note:** The dashboard's “Run sample check” action submits predefined synthetic observations to the backend. The API Health view summarizes the latest recorded alert per API; it is not an independent live probe of external services.

## Features

- Accepts one API observation or a batch of observations through `POST /monitor`.
- Detects:
  - HTTP client errors (`4xx`)
  - HTTP server errors (`5xx`)
  - Slow responses (configurable threshold; defaults to 3,000 ms)
  - Zero records returned
- Generates concise, actionable anomaly explanations using an OpenAI-compatible chat completion when an API key is configured.
- Uses a deterministic fallback explanation if the LLM key is absent or the LLM request fails.
- Persists anomaly records in PostgreSQL.
- Lists active or historical alerts and supports resolving an alert.
- Provides Overview, API Health, and Alert History views in a React dashboard.

## Technology stack

- Frontend: React, TypeScript, Vite, Lucide React
- Backend: Node.js, Express, TypeScript
- Database: PostgreSQL
- LLM integration: OpenAI Node SDK
- Local orchestration: Docker Compose

## Requirements

- Docker Desktop with Docker Compose
- Node.js and npm (only needed if running the frontend outside Docker)
- An OpenAI API key is optional. API usage and billing are separate from a ChatGPT subscription.

## Quick start (Docker Compose)

Run these commands in PowerShell from the repository root—the directory containing `compose.yaml` or `docker-compose.yml`.

1. **Configure environment variables.**

   Create a root `.env` file:

   ```powershell
   @'
   OPENAI_API_KEY=
   OPENAI_MODEL=gpt-4o-mini
   '@ | Set-Content -Encoding ascii .env
   ```

   Open `.env` in a text editor and set `OPENAI_API_KEY` to your own key if you want live LLM explanations. Keep the key private. Do not commit `.env` to Git.

   If no key is configured, the application should use the documented fallback explanation.

2. **Build and start the backend and PostgreSQL.**

   ```powershell
   docker compose up -d --build
   ```

3. **Start the frontend.**

   Open a second PowerShell terminal and change to the `frontend` directory:

   ```powershell
   cd frontend
   npm install
   Copy-Item .env.example .env
   npm run dev
   ```

   If `frontend/.env` already exists, check that it contains:

   ```dotenv
   VITE_API_BASE_URL=http://localhost:4000
   ```

4. **Open the dashboard.**

   Visit `http://localhost:5173`.

5. **Check service health.**

   In PowerShell:

   ```powershell
   Invoke-RestMethod http://localhost:4000/health
   ```

   Expected result includes a healthy status and the service name.

6. **Try a monitoring observation.**

   ```powershell
   $body = @{
       api_name = "DemoAPI"
       response_time_ms = 7000
       status_code = 500
       records_returned = 0
   } | ConvertTo-Json

   Invoke-RestMethod -Uri "http://localhost:4000/monitor" `
     -Method Post `
     -ContentType "application/json" `
     -Body $body
   ```

   This observation should trigger server-error, slow-response, and zero-record findings.

## API reference

### `GET /health`

Returns the backend health status.

### `POST /monitor`

Accepts either a single observation object or an array of observations.

Example request:

```json
{
  "api_name": "DemoAPI",
  "response_time_ms": 7000,
  "status_code": 500,
  "records_returned": 0
}
```

Observation fields:

| Field | Type | Constraints / meaning |
|---|---|---|
| `api_name` | string | Required; 1–120 characters |
| `response_time_ms` | integer | Required; 0–600,000 milliseconds |
| `status_code` | integer | Required; HTTP status range 100–599 |
| `records_returned` | integer | Required; 0–100,000,000 |
| `observed_at` | datetime string | Optional; ISO datetime accepted by validation |

A batch can contain 1–500 observations. The response reports the number processed, total anomalies, and per-observation results.

### `GET /alerts`

Returns alert records. Query parameters:

- `active=true` — return active alerts only.
- `active=false` — return resolved alerts only.
- `limit=200` — limit the number of results (the backend accepts 1–500).

Without `active`, the endpoint returns alerts across both states, subject to the limit.

### `PATCH /alerts/:id/resolve`

Marks the specified alert as resolved/inactive.

Example:

```powershell
Invoke-RestMethod -Uri "http://localhost:4000/alerts/1/resolve" -Method Patch
```

Replace `1` with an existing alert ID.

## Configuration

The backend configuration is supplied through environment variables in Docker Compose:

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | Backend listening port | `4000` |
| `DATABASE_URL` | PostgreSQL connection string | Set by Compose |
| `FRONTEND_ORIGIN` | Allowed frontend origin | `http://localhost:5173` |
| `OPENAI_API_KEY` | Optional LLM API key | Empty |
| `OPENAI_MODEL` | Chat completion model | `gpt-4o-mini` |
| `SLOW_RESPONSE_THRESHOLD_MS` | Slow-response threshold | `3000` |

Do not place secrets in frontend environment variables. Variables prefixed with `VITE_` are exposed to browser code.

## Useful commands

From the repository root:

```powershell
# Check running services
docker compose ps

# Follow backend logs
docker compose logs -f backend

# Rebuild the backend after backend code changes
docker compose up -d --build backend

# Stop services
docker compose down

# Stop services and delete the local database volume (DELETES STORED DATA)
docker compose down -v
```

The frontend dev server can be stopped with `Ctrl+C` in its terminal.

## Project structure

```text
.
├── backend/
│   └── src/
│       ├── server.ts       # HTTP routes, validation, persistence
│       └── monitor.ts      # Anomaly detection and LLM/fallback explanation
├── frontend/
│   └── src/
│       ├── App.tsx         # Dashboard and views
│       └── styles.css      # Dashboard styling
├── compose.yaml            # Docker Compose configuration (filename may vary)
├── .env                    # Local secrets; do not commit
└── README.md
```

Actual filenames may differ slightly depending on the repository's current structure.

## Troubleshooting

- **Frontend cannot reach backend:** confirm the backend is running and `VITE_API_BASE_URL` is `http://localhost:4000`.
- **Backend cannot connect to PostgreSQL:** run `docker compose ps` and inspect `docker compose logs postgres backend`.
- **LLM explanations fall back:** confirm the root `.env` contains a valid API key, then recreate the backend container with `docker compose up -d --build --force-recreate backend`. Check logs for LLM errors. Never share the key in screenshots or messages.
- **Port already in use:** stop the process using port `4000`, `5173`, or `5432`, or change the relevant port mapping and environment settings.
- **Dashboard has no alerts:** use “Run sample check” or send a test request to `POST /monitor`.

## Known scope and limitations

- This is an engineering demonstration, not a production-grade monitoring service.
- The sample check uses synthetic observations rather than polling configured third-party API URLs.
- The API Health view is derived from the latest alert record per API. APIs with no alert record may not appear in that summary.
- Alert resolution updates the stored alert state; it does not repair the upstream API.
- Configure and test authentication, authorization, rate limiting, secret management, retention policies, alert delivery, and production deployment before using this system with real services or sensitive data.
- Do not submit patient data or other personal/sensitive data to the LLM.

## Submission checklist

- [ ] Verify clean startup using the instructions above.
- [ ] Verify `GET /health`, `POST /monitor`, `GET /alerts`, and `PATCH /alerts/:id/resolve`.
- [ ] Confirm `.env` and API keys are excluded from Git.
- [ ] Include `AI_Prompts.docx` if AI assistance was used.
- [ ] Add the GitHub repository URL and demo video URL to the submission email.
