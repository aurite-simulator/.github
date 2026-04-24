# aurite-simulator

A framework for building and running business process simulations driven by a Redis-based simulated clock, with AI agents that monitor and analyze those simulations in real time.

---

## What This Is

Business simulations are hard to test with real time. This organization provides a **framework** that lets you define workers — Python functions that execute at scheduled points in simulated time — and run them at any speed you choose. A simulation that covers one week of business activity might complete in minutes.

Alongside each simulation, **AI agents** (built with the Anthropic SDK) run independently, reading the simulation's output and producing reports, alerts, or analysis. The model and its agents are decoupled: the simulation doesn't know about the agents, and the agents just read logs and databases.

---

## Repository Structure

```
framework/                   ← core simulation engine (clone this first)
├── models/
│   ├── test_model/          ← simple 3-worker test simulation with cron
│   └── bizdev-model/        ← B2B sales pipeline simulation (5 workers, AI notes)
├── agents/
│   ├── test_agent/          ← monitors test_model worker logs for stalls
│   └── simp_monitor/        ← generates executive reports from bizdev-model databases
└── utilities/               ← shared code (AI note generation) used by models
```

Each of `models/`, `agents/`, and `utilities/` is a separate git repository cloned into the framework at the path shown above. The framework never changes between models — all customization lives in the cloned repos.

### `framework`

The simulation engine. Manages a Redis-backed simulated clock, a ready gate that holds the clock until all workers have checked in, and a Lua-scripted ACK protocol that ensures every worker executes exactly once per tick before the clock advances. Provides `run.sh` for launching everything in one terminal and `setup.sh` for installing all dependencies (framework + model + utilities + agents) into a shared virtualenv.

### Models

A model is a git repository with a `model_config.py` (worker list, simulation time window, task assignments) and a `tasks/` directory of worker functions. Each task function takes the current simulated time and returns the number of simulated hours until that worker should fire again. Workers read and write external state — SQLite databases, CSV files, Redis keys — freely.

### Agents

Agents are standalone AI programs that run outside the simulation loop. They use the Anthropic SDK to call Claude, read the simulation's output artifacts (logs, databases), and produce reports or surface anomalies. They have no control over the simulation.

### `utilities`

Shared Python code importable by any model as `from utilities.module import ...`. Currently contains `notes_generator.py`, which calls Claude Haiku to generate realistic CRM notes for the bizdev simulation.

---

## Getting Started

### Prerequisites

- Python 3.12+
- Redis running locally — if not installed:
  - **macOS**: `brew install redis && brew services start redis`
  - **Ubuntu/Debian**: `sudo apt install redis-server && sudo systemctl enable --now redis-server`
  - **Other**: see the [Redis installation docs](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/)
- An [Anthropic API key](https://console.anthropic.com) — required for the bizdev simulation and both agents; not needed for the basic test simulation

---

### Example 1 — Test Simulation

A minimal simulation with three generic workers and a cron task. Use this to verify your setup and understand how the framework operates before running a full model.

**1. Clone all required repositories**

```bash
git clone https://github.com/aurite-simulator/framework framework
cd framework

git clone https://github.com/aurite-simulator/test_model  models/test_model
git clone https://github.com/aurite-simulator/test_agent  agents/test_agent
git clone https://github.com/aurite-simulator/utilities    utilities
```

**2. Install dependencies**

```bash
bash setup.sh
```

`setup.sh` creates a shared virtualenv and installs requirements for the framework, model, utilities, and any agents in one pass.

**3. Activate the virtualenv**

```bash
source venv/bin/activate
```

**4. Initialize Redis**

```bash
venv/bin/python3 setup_redis.py
```

**5. Run the simulation**

```bash
bash run.sh
```

This starts the controller and all workers in a single terminal. Press Ctrl+C to stop.

**6. Run the monitor agent** (in a separate terminal)

```bash
cd agents/test_agent
cp .env.example .env
# Edit .env — set ANTHROPIC_API_KEY and DATA_DIR (absolute path to framework's model_data/)
source ../../venv/bin/activate
python3 agent.py
```

The agent reads the simulation's worker log, identifies active and stalled workers, and prints a status report.

**7. (Optional) Schedule the monitor agent via the framework cron worker**

Instead of running the agent manually, you can have the framework's cron worker launch it automatically at a simulated-time interval. Open the model's crontab file:

```bash
models/test_model/crontab
```

Add a line specifying when (in simulated time) the agent should run. For example, to run it every 6 simulated hours:

```
0 */6 * * *   $MODEL_DIR/../../venv/bin/python3 $MODEL_DIR/../../agents/test_agent/agent.py
```

`$MODEL_DIR` is automatically set by the cron worker to the model's directory path, so this path resolves correctly regardless of where you cloned the framework. The agent runs as a fire-and-forget subprocess — it does not block the simulation clock.

---

### Example 2 — Business Development Simulation

A one-week simulation of a B2B sales pipeline. Five workers move leads from a raw prospect pool through scoring, assignment, and follow-up processing. Claude Haiku generates realistic CRM notes at each stage. A separate agent reads the three SQLite databases and produces an HTML executive dashboard.

**1. Clone all required repositories**

```bash
git clone https://github.com/aurite-simulator/framework framework
cd framework

git clone https://github.com/aurite-simulator/bizdev-model  models/bizdev-model
git clone https://github.com/aurite-simulator/simp_monitor   agents/simp_monitor
git clone https://github.com/aurite-simulator/utilities       utilities
```

**2. Install dependencies**

```bash
bash setup.sh
```

**3. Activate the virtualenv**

```bash
source venv/bin/activate
```

**4. Set your Anthropic API key**

```bash
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env
```

**5. Initialize databases and Redis**

```bash
venv/bin/python3 setup_redis.py
```

This seeds 1,000 prospect leads and 7 salespeople, clears the leads CSV, and prepares Redis for a fresh simulation run.

**6. Run the simulation**

```bash
bash run.sh
```

**7. Generate an executive report** (while the simulation is running, or after)

```bash
cd agents/simp_monitor
cp .env.example .env
# Edit .env — set ANTHROPIC_API_KEY and DATA_DIR (absolute path to framework's model_data/)
source ../../venv/bin/activate
python agent.py
```

The agent queries the three SQLite databases, calls Claude Haiku to write a narrative summary and recommendations, and saves a timestamped Markdown report and an HTML dashboard to `model_data/reports/`.

**8. (Optional) Schedule the report agent via the framework cron worker**

To have the agent generate reports automatically at a simulated-time interval, add it to the model's crontab file:

```bash
models/bizdev-model/crontab
```

For example, to generate a report every 12 simulated hours:

```
0 */12 * * *   $MODEL_DIR/../../venv/bin/python3 $MODEL_DIR/../../agents/simp_monitor/agent.py
```

`$MODEL_DIR` is automatically set by the cron worker to the model's directory path. The agent runs as a fire-and-forget subprocess and does not block the simulation clock.

**9. Inspect results directly**

```bash
# Pipeline status by stage
sqlite3 model_data/customer_database.db \
  "SELECT status, COUNT(*) FROM customers GROUP BY status"

# Top scored leads
sqlite3 model_data/customer_database.db \
  "SELECT name, company, score, salesperson, status FROM customers ORDER BY score DESC LIMIT 10"

# View AI-generated notes for a processed lead
sqlite3 model_data/customer_database.db \
  "SELECT name, notes FROM customers WHERE status='PROCESSED' LIMIT 1"
```

---

## How the Bizdev Pipeline Works

```
lead_database (1,000 seed leads)
        │
        ▼
  lead_generator ──► leads.csv
                          │
                          ▼
                    lead_finder ──► customer_database (NEW)
                                          │
                                          ▼
                                    score_lead ──► SCORED
                                                      │
                                                      ▼
                                                assign_lead ──► ASSIGNED
                                                                    │
                                                                    ▼
                                                             process_lead ──► PROCESSED
```

| Worker | Cadence | Role |
|---|---|---|
| `lead_generator` | every 5–8 sim hours | Pulls 2–10 new leads from the prospect pool, blanks 1–3 fields to simulate incomplete contact data |
| `lead_finder` | every 2–4 sim hours | Imports leads into the CRM with AI-generated opening notes via Claude |
| `score_lead` | dynamic | Scores each new customer 10–90; fires again in N hours where N = leads just scored |
| `assign_lead` | every 2–5 sim hours | Assigns the highest-scored unassigned lead to an available salesperson |
| `process_lead` | every 1–10 sim hours | Closes a lead, appends AI-generated follow-up notes via Claude, frees the salesperson |

---

## Adding a New Model

See `DEVELOPER_GUIDE.md` in the framework repository for the full task function contract, cron worker documentation, and step-by-step instructions for creating a new model repository and wiring it into the framework.
