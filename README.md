# LLM-in-the-Loop Adaptive HVAC Control

> Bridging Natural Language Understanding and Closed-Loop PID Control via Agent-Based Architecture

**CMU 12-770 Course Project | Spring 2026**

**Authors:** Zhengjie Cui, Zihao Hu, Junchi Fan, Bo Zhang

**Project Blog:** https://github.com/AlexCui8591/12770-2026S-proejct/

---

## Overview

Traditional smart thermostats require manual configuration of temperatures, schedules, and operating modes—failing to account for dynamic factors such as real-time weather, electricity tariffs, and personal daily routines. This project develops a **Natural-Language-Driven Intelligent HVAC Control Agent** that accepts plain-English comfort directives (e.g., *"Make the bedroom cooler, but keep electricity costs low"*) and translates them into adaptive control actions for downstream PID controllers.

The core innovation is the **PID-as-Tool** paradigm: rather than generating a one-shot setpoint, the LLM agent treats the PID controller as a bidirectional callable tool—reading telemetry and writing parameter updates at runtime—transforming the agent from a translator into an online closed-loop supervisor.

## Architecture

The system is organized into four layers:

```
┌─────────────────────────────────────────────────┐
│  Layer 1: User Input Layer                      │
│  Natural-language thermal comfort directives    │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│  Layer 2: Semantic Decision Layer (LLM Agent)   │
│  Qwen2.5-7B via Ollama                          │
│  • Intent parsing → structured schema           │
│  • Tool invocation (7 tools)                    │
│  • Supervisory reasoning (every 5 min)          │
│  • Dual-strategy inference (tool-call + fallback│
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│  Layer 3: Adaptive Control Layer                │
│  • Self-adaptive PID (Kp, Ki, Kd online tuning) │
│  • Cost function: J = ω_comfort·∫e²dt           │
│    + ω_energy·∫P dt + ω_response·∫(du/dt)²dt   │
│  • Nelder-Mead / Bayesian Optimization          │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│  Layer 4: Data Support Layer                    │
│  SQLite database with 7 tool interfaces:        │
│  weather, tariff, schedule, room state,          │
│  user habits, solar irradiance, PID telemetry   │
└─────────────────────────────────────────────────┘
```

## Repository Structure

```
12770-Project/
├── pyproject.toml            # Project metadata and dependencies (uv)
├── README.md
├── docs/                     # GitHub Pages blog / journal
│   └── index.md
├── agent/                    # LLM agent core
│   ├── agent.py              # Main agent loop
│   ├── tools.py              # Tool definitions (7 tools)
│   ├── schema.py             # Output schema validation
│   └── dual_strategy.py      # Tool-call + prompt-injection fallback
├── controller/               # PID control layer
│   ├── pid.py                # Baseline fixed-parameter PID
│   ├── adaptive_pid.py       # Self-adaptive PID with cost function
│   ├── optimizer.py          # Nelder-Mead / BO gain search
│   ├── mapper.py             # Home Assistant action mapping
│   └── safety.py             # Safety constraints enforcement
├── data/                     # Data support layer
│   ├── database.py           # SQLite interface
│   ├── weather.py            # Outdoor weather query
│   ├── tariff.py             # Electricity tariff lookup
│   ├── schedule.py           # User schedule query
│   ├── solar.py              # Solar irradiance forecast
│   └── telemetry.py          # PID telemetry tool
├── simulation/               # Simulation environment
│   ├── thermal_model.py      # Single-zone thermal dynamics
│   ├── scenarios.py          # S1–S6 scenario definitions
│   └── runner.py             # Experiment runner
├── experiments/              # Experiment configs and results
│   ├── configs/              # YAML configs for C0–C3 conditions
│   └── results/              # Raw logs and analysis outputs
└── tests/                    # Unit and integration tests
    ├── test_agent.py
    ├── test_pid.py
    └── test_thermal_model.py
```

## Environment Setup

### Prerequisites

- **Python 3.11+**
- **[uv](https://docs.astral.sh/uv/)** — fast Python package manager
- **[Ollama](https://ollama.com/)** — local LLM inference runtime

### 1. Clone the repository

```bash
git clone https://github.com/Boz322/12770-Project.git
cd 12770-Project
```

### 2. Initialize the project with uv

The project uses `uv` for dependency management. If you don't have `uv` installed:

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Then initialize and sync:

```bash
# Initialize (if pyproject.toml doesn't exist yet)
uv init

# Install all dependencies
uv sync
```

### 3. Configure `pyproject.toml`

The `pyproject.toml` should contain:

```toml
[project]
name = "llm-hvac-control"
version = "0.1.0"
description = "LLM-in-the-Loop Adaptive HVAC Control System"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
authors = [
    { name = "Zhengjie Cui" },
    { name = "Zihao Hu" },
    { name = "Junchi Fan" },
    { name = "Bo Zhang" },
]

dependencies = [
    "ollama>=0.4.0",
    "numpy>=1.26.0",
    "scipy>=1.12.0",
    "pandas>=2.2.0",
    "matplotlib>=3.8.0",
    "pyyaml>=6.0",
    "requests>=2.31.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "ruff>=0.4.0",
]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### 4. Install and pull the LLM model

```bash
# Install Ollama (see https://ollama.com/download)
# Then pull the Qwen2.5-7B model:
ollama pull qwen2.5:7b
```

### 5. Verify installation

```bash
# Activate the virtual environment
source .venv/bin/activate   # Linux/macOS
# or: .venv\Scripts\activate  # Windows

# Run tests
uv run pytest

# Verify Ollama is running
ollama list
```

## Usage

### Run the agent with a natural-language command

```bash
uv run python -m agent.agent --query "Make the bedroom cooler, but keep electricity costs low"
```

### Run a simulation scenario

```bash
# Run scenario S1 (steady-state tracking) with condition C0 (fixed PID baseline)
uv run python -m simulation.runner --scenario S1 --condition C0

# Run all scenarios with all conditions (full experiment)
uv run python -m simulation.runner --all --runs 5
```

### Run ablation studies

```bash
uv run python -m simulation.runner --ablation A1  # Cost-function ablation
uv run python -m simulation.runner --ablation A2  # Supervision frequency
uv run python -m simulation.runner --ablation A3  # Proactive vs. reactive
```

## Experimental Design

| Condition | Description | PID Params | LLM Role |
|-----------|-------------|------------|----------|
| C0 | Fixed PID Baseline | Static (manual) | None |
| C1 | LLM Setpoint-Only | Static | One-shot setpoint |
| C2 | LLM-in-the-Loop (Reactive) | LLM-supervised | Online supervisor |
| C3 | LLM-in-the-Loop (Proactive) | LLM-supervised | Proactive + reactive |

Six test scenarios (S1–S6) covering steady-state tracking, weather disturbance, cost optimization, occupancy adaptation, multi-disturbance stress, and LLM failure resilience. Each (condition, scenario) pair is run 5 times with different random seeds.

See the [project report](docs/) for full experimental design details.

## License

MIT

## Acknowledgments

CMU 12-770: Control Systems for Intelligent Buildings, Spring 2026.
