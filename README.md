🚆 Transit Reliability Agent

A commute tool that tells you how much to trust your ETA — not just what it is.

Combines live arrivals across BART, LA Metro, and NYC MTA with real station equipment status and a tool-calling AI agent that reasons over the data to answer commute questions.




📍 Why this exists

Most transit apps — Citymapper, Transit, Google Maps, Moovit — give you one point-estimate ETA built from the same official real-time feed everyone else uses. None of them tell you how much to trust that number, and none of them factor in station-level equipment reliability, even though several agencies publish live elevator and escalator status.


This project treats a commute prediction as a range with a confidence level, not a single number — and layers in real equipment status where it's available, instead of assuming every station behaves the same.




✨ Features


🚇 Live arrivals across three metros — BART, LA Metro, and NYC MTA, all ingested through a single unified parser built on the GTFS-realtime standard
♿ Equipment-aware reliability scoring — BART and NYC MTA publish live elevator/escalator status; this is layered on top of the core pipeline as an optional per-agency plugin. LA Metro falls back to arrival-variance-only scoring where equipment data isn't available
📊 Reliability scoring — heuristic in v1 (equipment outage frequency + arrival variance); trained quantile regression model planned for v2 once enough historical data is collected
🤖 Tool-calling AI agent — powered by the Claude API, with direct access to live arrival, equipment, and reliability data as callable tools, scoped by agency and station — genuinely agentic, not a prompt-and-summarize wrapper



🏗️ Architecture

GTFS-RT feeds (BART, LA Metro, NYC MTA)
        │
  unified ingestion worker (one parser, per-agency config table)
        │
     PostgreSQL (agency_id on every core table)
        │
  equipment status plugin layer
      (BART → BSA API │ NYC → elevator/escalator API │ LA → fallback)
        │
  scoring service
      (equipment-aware where available, arrival-variance-only otherwise)
        │
  agent layer — Claude API + tools:
      get_arrivals(agency, station)
      get_elevator_status(agency, station)
      get_reliability_score(agency, segment)
        │
     frontend (React — city selector, arrivals view, agent chat)


🧰 Tech stack

LayerToolsBackendFastAPI · Uvicorn · SQLAlchemy/SQLModel · Alembic · Pydantic · APScheduler · gtfs-realtime-bindingsDatabasePostgreSQL · Docker ComposeAI / AgentAnthropic Claude API (tool-calling)FrontendReact (Vite) · TypeScript · TanStack Query · Tailwind CSS · shadcn/ui (optional)DeploymentRailway/Render/Fly.io (backend + DB) · Vercel (frontend) · GitHub Actions (optional CI)Testing / ConfigPytest · python-dotenv

Built as a PWA, not React Native — keeps deployment simple for a demo-focused build.


🗄️ Data model

agencies             id · name · code (BART / LAMETRO / MTA)
stations             id · agency_id · name · agency_station_code
segments             id · agency_id · name · origin_station_id · destination_station_id · notes
arrival_snapshots    id · agency_id · station_id · line · scheduled_time · estimated_time · captured_at
advisories           id · agency_id · station_id · type · status · description · started_at · resolved_at
reliability_scores   id · agency_id · segment_id · score · computed_at · method

segments are your personally tracked routes — this is what the reliability score is computed against.


🚀 Getting started

Prerequisites


Python 3.11+
Node.js 18+
Docker (for Postgres via docker-compose)
Agency API access:

BART GTFS-RT — no registration required
LA Metro GTFS-RT — no registration required
NYC MTA GTFS-RT + elevator/escalator API — register at api.mta.info



An Anthropic API key for the agent layer


Backend setup

bashgit clone <repo-url>
cd transit-reliability-agent

cp .env.example .env
# fill in agency API keys and ANTHROPIC_API_KEY in .env

docker-compose up -d          # starts Postgres
pip install -r requirements.txt

alembic upgrade head          # run database migrations

uvicorn app.main:app --reload

Frontend setup

bashcd frontend
npm install
npm run dev

Ingestion worker

bashpython -m app.workers.ingest


Polls each agency's GTFS-RT feed (and equipment APIs where available) on a schedule and writes snapshots to Postgres. Historical reliability data only accumulates while this runs — it's meant to run continuously in the background.




🧠 Design decisions


Unified GTFS-RT parser across all three agencies — BART, LA Metro, and NYC MTA all publish standard GTFS-realtime feeds, so one parser handles arrivals ingestion for all three instead of three one-off integrations. Agency-specific differences (auth, station ID mapping) live in a config table, not duplicated logic.
Equipment status as an optional plugin, not a core dependency — equipment data isn't standardized across agencies, so it's layered per agency rather than assumed universal. Scoring degrades gracefully to arrival-variance-only where it's missing.
Heuristic score over a trained model in v1 — a real confidence-band model needs weeks of historical variance data. The pipeline is built so that upgrade is a data problem later, not an architecture rewrite now.
Agent has direct tool access, not pre-fetched context — the model decides what's relevant to a given question at call time, scoped by agency and station. This is the actual difference between an agent and a wrapper.



🗺️ Roadmap


 Trained quantile regression model for confidence-band predictions
 Personal calibration — learn a user's specific missed-transfer patterns
 Cross-agency transfer scoring (e.g. LA Metro → connecting bus, NYC subway → connecting line)
 User accounts / multi-user support



📌 Status

Actively in development. Currently: BART, LA Metro, and NYC MTA arrivals live, with equipment-aware reliability scoring for BART and NYC MTA.