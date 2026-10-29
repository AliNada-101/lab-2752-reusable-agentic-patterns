# Run LAB-2752 with IBM Bob — Quick Runbook

Let Bob drive all four labs. Paste one prompt per lab; Bob builds, starts, and verifies.
Full lab detail lives in each folder's `README.md`. This page is the shortcut.

## Before you paste anything

All three `.env` files already exist and are filled. **Tell Bob to use them as-is.**

| File | What it holds |
|---|---|
| `01-fleet-backend/.env` | Orchestrate URL + API key + 5 agent IDs |
| `02-forecast-service/.env` | Backend URL, `USE_TTM_MODEL`, WML credentials |
| `04-carbon-dashboard/.env` | Port 4000, backend `:8085`, forecast `:5001` |

**Ground rules for every prompt below (paste once at the start of the session):**

```
Rules for this session: use the existing .env files as they are. Never overwrite them,
never print API keys or secrets, never commit them. Use python3.12 (not 3.13+).
After each step, verify with a curl or health check before moving on. Report failures
with the exact error and fix them rather than skipping.
```

## Ports

| Service | Port | Health check |
|---|---|---|
| Fleet backend | 8085 | `curl localhost:8085/api/trucks` |
| Forecast service | 5001 | `curl localhost:5001/api/forecast/fleet` |
| Carbon dashboard | 4000 | open `http://localhost:4000` |

Run each service in its own terminal (Bob can start them in the background).

---

## Lab 0 · Setup — Agent mode

```
Validate my setup: python3.12 and node 18+ are available, the three .env files have no
empty required values (report key names only), and .bob/custom_modes.yaml lists both the
TTM Forecasting and FleetOps Agent Builder modes. Install anything missing from
bob-modes-skills/.
```

## Lab 1 · Fleet backend — Agent mode

```
Start 01-fleet-backend on port 8085 using its existing venv (create it with python3.12
and pip install -r requirements.txt if missing). Then find the truck with incidentType
WEATHER_FAILURE, run POST /api/agents/execute for it, poll /api/agents/status until
COMPLETED, and confirm the truck is DIVERTED. Summarize each agent stage.
```

Expected: 10 trucks, incident truck found, `decision: EMERGENCY_REROUTE`, status `DIVERTED`.

## Lab 2 · Forecast service — 🔮 TTM Forecasting mode

Fast path (recommended): run the pre-built service. Build-from-scratch path: use the Bob prompt in `02-forecast-service/README.md`.

```
Run the forecast service from 02-forecast-service/checkpoint on port 5001. Create a
python3.12 venv, install requirements.txt, and export the existing 02-forecast-service/.env
into the shell (`set -a; source ../.env; set +a`), because the app only reads a .env in its
working directory. Don't copy or duplicate the file. Verify GET /api/forecast/temperature/{incident_truck_id} returns breach_predicted: true
and risk_level HIGH or CRITICAL, and that /api/forecast/fleet lists the incident truck
in trucks_at_risk.
```

> `.env` currently has `USE_TTM_MODEL=True` (real Granite TTM-R2 via WML). First startup is slower
> (model and torch install). If it fails or is too slow, tell Bob: *"Fall back to `USE_TTM_MODEL=False`."*

## Lab 3 · watsonx Orchestrate — 🚛 FleetOps Agent Builder mode

Ready-to-upload files are in `03-watsonx-orchestrate/assets/` (5 agents, 5 tools, 2 connections).

```
Deploy the 5 agents from 03-watsonx-orchestrate/assets to my active orchestrate env.
Order: import connections, then tools (with their requirements.txt), then agents.
Check `orchestrate env list` and the active env first. Then confirm all 5 agents are
listed, and that the agent IDs in 01-fleet-backend/.env match the deployed ones.
Restart the fleet backend and confirm the log says "WatsonX Orchestrate configuration validated".
```

Heads-up from experience (ADK 2.16.1 differs from the lab text):

- `orchestrate env add --name <n> --url <u>`, and pass `--api-key` to `env activate`.
- `orchestrate tools import -a <app-id>` replaces `--connection`.
- `decision_tool` needs a requirements file (`-r`).
- **Cloud Orchestrate cannot reach `localhost`.** Set the `fleetops_api` and `forecast_api` connection URLs to public tunnel URLs (for example ngrok or cloudflared on 8085 and 5001), not `http://localhost:...`.

## Lab 4 · Carbon dashboard — Agent mode

```
Start 04-carbon-dashboard (npm install, node server.js) on port 4000. Verify these in
order: /api/trucks proxies to 8085, the Forecast tab loads from 5001, and the Agent
Pipeline tab polls status every 3 seconds. Fix any wiring gaps in server.js or
public/*.js. Then trigger the live pipeline on the incident truck and report the final
decision, urgency, confidence, and the truck's status.
```

Note: the lab text says `cd FleetOps`. The folder is `04-carbon-dashboard`.

---

## One-shot version (all labs)

```
Run LAB-2752 end to end following README-BOB.md. Use existing .env files, never expose
secrets, python3.12 only. Start backend (8085), forecast service (5001, from
02-forecast-service/checkpoint), then deploy the Orchestrate assets, then start the
dashboard (4000). Verify each stage before the next, then run the live pipeline on the
WEATHER_FAILURE truck and show me the final decision.
```

## Success = all of these

- [ ] `localhost:8085/api/trucks` returns 10 trucks
- [ ] Forecast shows `breach_predicted: true` for the incident truck
- [ ] 5 agents active in Orchestrate and IDs match `01-fleet-backend/.env`
- [ ] Live pipeline completes with `EMERGENCY_REROUTE` and the truck becomes `DIVERTED`
- [ ] Dashboard Fleet, Truck Detail, Forecast, Agent Pipeline, and Notifications tabs show data

## If something breaks

| Symptom | Ask Bob |
|---|---|
| `pydantic-core` build error | *"I'm on Python 3.13. Recreate the venv with python3.12."* |
| Pipeline stuck on `RUNNING` | *"Check the backend log for IAM or token errors and test my Orchestrate API key."* |
| Agents fail to call tools | *"Check the connection URLs. They must be publicly reachable, not localhost."* |
| Forecast too slow or erroring | *"Switch USE_TTM_MODEL to False and restart."* |
| Dashboard tab empty | *"Debug the proxy for that tab in server.js."* |
