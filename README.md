# ⌂ Personal Health Data Stack

*An agent-driven personal analytics setup — the umbrella that ties the ingester repos together.*

This repo is the **map**, not the code. The actual pipelines live in focused sibling repos (linked below). The whole system runs on a home computer (macOS here) with an AI/LLM agent (OpenClaw) as the orchestrator: it listens in chat, parses voice notes, writes to PostgreSQL, and produces reports.

## The big picture

```
                ┌─────────────────────────────────────────────┐
                │            Messaging (Telegram)             │
                │  voice notes / text check-ins / requests    │
                └───────────────┬─────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │  AI/LLM agent (OpenClaw)│
                    │  • parses voice → Whisper │
                    │  • understands intent     │
                    │  • runs scheduled jobs    │
                    │  • formats digests        │
                    └───────┬───────────┬───────┘
                            │           │
              ┌─────────────▼──┐    ┌───▼─────────────────┐
              │  Ingesters     │    │  PostgreSQL (dunkle_db)│
              │  (scheduled    │    │  • mood_log           │
              │   / on-demand) │    │  • food_log_entry     │
              └───────┬────────┘    │  • health_records     │
                      │             │  • airthings_samples  │
              ┌───────▼────────┐    │  • purpleair_samples  │
              │  Device/cloud  │    │  • daily_summary      │
              │  APIs + LAN    │    └─────────▲─────────────┘
              │  sensors       │              │
              └────────────────┘              │
                                   ┌──────────┴──────────┐
                                   │  Analysis / reports │
                                   │  correlation, fatigue│
                                   │  CUSUM, forecasting  │
                                   └─────────────────────┘
```

**How it works in practice:**
1. **Capture** — the user sends a voice note ("mood 7, had a rough meeting") or a food log ("lunch: chicken sandwich") by Telegram. The agent transcribes it (Whisper) and inserts it.
2. **Store** — sensors (Airthings, PurpleAir) push on cron to PostgreSQL; the Apple Watch exports via a local receiver.
3. **Analyze** — nightly and weekly jobs compute food kJ estimation, mood modeling, fatigue scores, and correlations.
4. **Report** — the agent renders tables as images (Telegram-friendly) and posts digests to chat/Discord.

## Repos in the stack

| Repo | What it does |
|------|--------------|
| [mood-log](https://github.com/dpwilcock/mood-log) | Mood check-in parsing (LLM processors), mood vs. workload modeling |
| [food-logging](https://github.com/dpwilcock/food-logging) | Free-text food logging with number-word normalization, nightly kJ estimation |
| [apple-health-data-monitor](https://github.com/dpwilcock/apple-health-data-monitor) | Apple Watch / Health data → Postgres (local receiver), fatigue & correlation analysis |
| [airthings-sync](https://github.com/dpwilcock/airthings-sync) | Airthings indoor-air sensors → Postgres, resumable backfill |
| [purpleair-sync](https://github.com/dpwilcock/purpleair-sync) | PurpleAir outdoor-air sensor → Postgres, live LAN + cloud history |
| [municipal-council-watchdog](https://github.com/dpwilcock/municipal-council-watchdog) | Civic watchdog (the non-health sibling — same stack, community focus) |

## Shared conventions

- **PostgreSQL** is the single store: one database, several schemas.
- **Schema name** is configurable via `PGSCHEMA` env var (default `"DW Health"` — the personal-health schema).
- **No secrets in repos** — all credentials come from environment variables (see each repo's README).
- **Idempotent writes** — inserts use `ON CONFLICT DO NOTHING` / upserts, so re-runs are safe.
- **Env-var config** everywhere: `PGDATABASE`, `PG_DSN`, `LOG_FILE`, `HEALTH_HOST`/`HEALTH_PORT`, etc.

## Adding your own data

Use the same shape:
1. Pick or write an ingester repo (they're small; fork one).
2. Point it at your Postgres via `PG_DSN` / `PGDATABASE` / `PGSCHEMA`.
3. Schedule it (cron works; the agent can also trigger on chat requests).
4. Add reporting: the agent's job is turning rows into answers.

## License

MIT, per-repo. This umbrella is just documentation.