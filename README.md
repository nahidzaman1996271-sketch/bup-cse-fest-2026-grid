# GridWise — LLM-Assisted Energy Optimization API

GridWise is a FastAPI service that generates optimal 24-hour energy dispatch schedules for a solar + battery + grid setup. It combines natural-language operator directives with a linear programming solver to produce a cost-minimizing hourly plan.

**Live API:** https://bup-cse-fest-2026-grid.onrender.com

## How it works

1. **Directive interpretation** — Free-text operator notes (e.g. "reduce solar output by half between 2pm and 4pm") are parsed into structured directives. If `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` is set, the service can call an LLM for interpretation; otherwise it falls back to a robust rule-based parser.
2. **Guardrail validation** — Every interpreted directive is sanitized and bounds-checked (hours clamped to 0–23, factors clamped to 0–1, reserves clamped to battery capacity) before it ever reaches the solver.
3. **Optimization** — A linear program (via [PuLP](https://coin-or.github.io/pulp/) and the CBC solver) computes the hourly grid draw, solar usage, and battery charge/discharge schedule that minimizes total electricity cost while respecting all constraints.

## Tech stack

- **FastAPI** — API framework
- **Pydantic** — request/response validation
- **PuLP + CBC** — linear programming solver
- **Docker** — containerized deployment
- **Render** — hosting

## API Endpoints

### `GET /health`
Health check. Returns:
```json
{"status": "ok"}
```

### `POST /optimize-energy`
Computes the optimal 24-hour schedule.

**Request body:**
```json
{
  "scenario_id": "string",
  "operator_notes": ["string", "up to 3 notes"],
  "hours": [
    {
      "hour": 0,
      "demand_kwh": 0.0,
      "solar_kwh": 0.0,
      "tariff_bdt_per_kwh": 0.0
    }
    // ... exactly 24 entries, one per hour
  ],
  "battery": {
    "capacity_kwh": 0.0,
    "initial_energy_kwh": 0.0,
    "minimum_energy_kwh": 0.0,
    "max_charge_kwh_per_hour": 0.0,
    "max_discharge_kwh_per_hour": 0.0
  }
}
```

**Response body:**
```json
{
  "scenario_id": "string",
  "directive_interpretation": [ /* how each operator note was interpreted */ ],
  "hourly_plan": [ /* grid/solar/battery values for each of the 24 hours */ ],
  "total_grid_kwh": 0.0,
  "total_cost_bdt": 0.0,
  "peak_grid_kwh": 0.0,
  "plan_summary": "string"
}
```

Interactive docs (Swagger UI) are available at `/docs` once deployed.

## Project structure

```
bup/
├── Dockerfile
├── main.py
├── requirements.txt
└── vercel.json
```

## Running locally

```bash
cd bup
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Visit `http://localhost:8000/docs` to try it out.

## Running with Docker

```bash
cd bup
docker build -t gridwise .
docker run -p 8000:8000 gridwise
```

## Environment variables (optional)

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Enables LLM-based operator note interpretation (falls back to rule-based parsing if unset) |
| `OPENAI_API_KEY` | Alternative LLM provider for note interpretation |
| `LLM_MODEL` | Overrides the default Anthropic model used for interpretation |

## Deployment

This service is deployed on [Render](https://render.com) using the Docker environment, with **Root Directory** set to `bup` so Render builds from the `Dockerfile` in that folder.
