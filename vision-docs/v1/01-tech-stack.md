## Dev Setup
The dev-setup for this device is located in `Desktop/dev-setup/`.

## Technology Stack

### Core Application

* **Python** — Primary language for the programming engine, integrations, data processing, and business logic.
* **FastAPI** — Local API layer for exposing programming operations to the agent and other clients.
* **Pydantic** — Validation and structured data models for workouts and API requests.
* **SQLAlchemy** — Database models and persistence layer.
* **SQLite** — Initial source of truth for workout history, program versions, recommendations, and decisions. PostgreSQL may replace it later if remote hosting, concurrency, or multiple users become necessary.

### Agent Layer

* **Hermes Agent** — Conversational interface, task orchestration, scheduling, and tool usage.
* **MCP server** — Narrow interface between Hermes and the Python application. The agent should call purpose-built tools rather than receive unrestricted database, spreadsheet, or shell access.

Hermes should coordinate workflows and explain decisions. Deterministic Python code should perform calculations, validate data, and enforce coaching policies.

### User/Athlete Interface and Integrations

* **Google Sheets** — Primary interface for viewing workouts and logging completed sets.
* **Google Sheets API** — Reads completed workouts and publishes upcoming sessions.
* **Google service account or OAuth** — Provides access only to the required spreadsheet.

### Development and Operations

* **Git** — Source control for application code, coaching policies, schemas, and documentation.
* **Local macOS installation** — Hermes, the FastAPI service, SQLite, and scheduled jobs initially run under a dedicated standard macOS user on the spare MacBook.

## Architectural Principle

The coaching platform should remain independent of the agent framework.

Hermes is responsible for conversation, orchestration, scheduling, and presenting recommendations. The standalone Python application is responsible for storing data, calculating metrics, applying programming rules, and producing auditable decisions.

This separation allows Hermes to be replaced by OpenClaw or another agent framework later without rewriting the core coaching system.