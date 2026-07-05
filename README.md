# Political Alpha Engine

**Macro trading signals derived from the language of politicians and central bankers — not what they say, but how they say it.**

![Python](https://img.shields.io/badge/backend-Python%203.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/API-FastAPI-009688?logo=fastapi&logoColor=white)
![Next.js](https://img.shields.io/badge/frontend-Next.js%2014-000000?logo=next.js&logoColor=white)
![ROCm](https://img.shields.io/badge/inference-AMD%20ROCm-ED1C24?logo=amd&logoColor=white)
![Vercel](https://img.shields.io/badge/deploy-Vercel-000000?logo=vercel&logoColor=white)
![Status](https://img.shields.io/badge/status-research%20prototype-yellow)

> **Signal-only.** This system generates and backtests research signals. It does not place trades. See [Honest limitations](#honest-limitations).

---

## Table of contents

- [Why this exists](#why-this-exists)
- [Architecture](#architecture)
- [Stance taxonomy](#stance-taxonomy-the-actual-ip-of-this-project)
- [AMD AI kit usage](#amd-ai-kit-usage-recommended-split)
- [Quick start](#quick-start)
- [Repo layout](#repo-layout)
- [Deployment](#deployment)
- [Roadmap](#roadmap)
- [Honest limitations](#honest-limitations)

---

## Why this exists

Generic news-sentiment trading tools score positive/negative and call it a day. That's not
how macro markets actually react to political and central-bank language — a Fed statement
that's *hawkish but hedged* ("may consider tightening") moves markets far less than one
that's *hawkish and certain* ("will raise rates"). Tariff rhetoric from a podium moves
different sectors than the same rhetoric buried in a committee report.

This project builds a **domain-specific stance model** — not sentiment analysis — fine-tuned
and served locally on an AMD AI Developer Kit, feeding a backtested signal layer and a
Vercel-hosted research dashboard.

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐     ┌────────────┐
│  Ingestion  │ --> │  Stance Engine   │ --> │  Signal Mapper  │ --> │ Dashboard  │
│  (Python)   │     │  (AMD box/ROCm)  │     │  + Backtester   │     │ (Next.js/  │
│             │     │                  │     │  (Python)       │     │  Vercel)   │
└─────────────┘     └──────────────────┘     └─────────────────┘     └────────────┘
     news APIs         classifier + LLM         historical corr         signal feed
     Fed calendar       fallback                  lookup table          + stance gauge
     congress.gov                                 vectorbt backtest     + backtest chart
```

**Key constraint that shapes everything:** Vercel has no GPU/NPU. The AMD kit runs the NLP
inference layer on your own machine, exposed as a FastAPI service (tunneled, not
publicly opened). Vercel hosts only the Next.js dashboard, which reads precomputed
signals — recommended via a small hosted Postgres so the UI stays live even when the
AMD box is offline (see [Deployment](#deployment)).

## Stance taxonomy (the actual IP of this project)

Every statement is scored on 4 independent axes instead of a single sentiment score:

| Axis | Range | What it captures |
|---|---|---|
| `hawkish_dovish` | -1.0 to 1.0 | Central bank tightening vs. easing language |
| `protectionism` | 0.0 to 1.0 | Tariff / trade-war rhetoric intensity |
| `regulatory_threat` | 0.0 to 1.0 | Sector-specific regulatory crackdown language |
| `certainty` | 0.0 to 1.0 | Hedged ("may consider") vs. committed ("will") framing |

`certainty` multiplies the other three — a hawkish statement said with low conviction
produces a weaker signal than the same statement said with certainty. This is the detail
generic sentiment pipelines miss entirely.

## AMD AI kit usage (recommended split)

1. **Primary — fine-tuned classifier.** Fine-tune a small encoder (DistilBERT/FinBERT) on
   labeled political/Fed statements against the 4 axes above. Export to ONNX, serve via
   ONNX Runtime with the ROCm execution provider. Fast and cheap enough to score every
   incoming headline — this is what gets backtested.
2. **Fallback — local LLM via ROCm.** Statements the classifier scores low-confidence on
   get routed to a locally-hosted quantized model (Llama 3 8B / Mistral 7B via
   `llama.cpp` built with `LLAMA_HIPBLAS=on`) for a structured, explained score.
3. Every LLM-scored case gets logged against the classifier's original guess — this
   becomes the next fine-tuning batch, so accuracy compounds over time.

Implementation of both paths behind one interface: [`backend/app/stance_model.py`](backend/app/stance_model.py).

## Quick start

```bash
# 1. Clone and set up the backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. Smoke-test the pipeline end-to-end on bundled sample data
python -m backtest.sample_run

# 3. Run the API (once you've exported a fine-tuned classifier — see stance_model.py)
uvicorn app.main:app --reload

# 4. In a separate terminal, run the dashboard (works on mock data out of the box)
cd ../frontend
npm install
npm run dev
```

Open `http://localhost:3000` — the dashboard renders against mock signals until
`BACKEND_URL` is set, so you can iterate on UI/UX before the ML pipeline is trained.

## Repo layout

```
backend/            FastAPI service — runs on your AMD machine
  app/
    main.py          API entrypoint (/analyze, /signals, /backtest)
    stance_model.py  Classifier + LLM fallback (the AMD-hosted piece)
    schemas.py       Pydantic models
  ingestion/
    sources.py       Tracked speakers/feeds config
    news_fetcher.py  Pulls news + Fed calendar + congress.gov statements
  signals/
    mapper.py        Stance scores -> asset-class directional signals
  backtest/
    engine.py        Event-study backtest of the stance -> price-move hypothesis
    sample_run.py    End-to-end example run against bundled sample data
  data/
    sample_events.json   Labeled example events to bootstrap fine-tuning
  requirements.txt
  Dockerfile         ROCm base image + llama.cpp HIP build

frontend/           Next.js app — deploys to Vercel
  app/page.tsx        Dashboard (signal feed + stance gauge)
  app/api/signals/route.ts   Reads from DB / proxies to backend / mock fallback
  components/         StanceGauge, SignalFeed, EventCard
```

## Deployment

- **Backend** — run on the AMD box (bare metal or `backend/Dockerfile`). Expose via
  Cloudflare Tunnel or Tailscale Funnel; never open the port directly to the internet.
- **Recommended pattern** — backend writes computed signals to a small hosted Postgres
  (Supabase/Neon free tier); Next.js just reads from that DB. Keeps the dashboard live
  even when the AMD machine is off.
- **Frontend** — `vercel deploy` from `frontend/`. Set `BACKEND_URL` (or `DATABASE_URL`
  if using the DB pattern) in Vercel's environment variables.

## Roadmap

- [ ] Label 500–1,000 real statements across the 4 axes (the actual bottleneck — see below)
- [ ] Fine-tune and export the DistilBERT/FinBERT classifier to ONNX
- [ ] Wire live ingestion (NewsAPI + Fed calendar transcript scraper)
- [ ] Replace prior-based weights in `signals/mapper.py` with fitted coefficients from backtest results
- [ ] Add a backtest-results chart to the dashboard (currently API-only via `/backtest`)

## Honest limitations

- Historical labeled data for "political statement → market move" is thin and noisy;
  `backtest/engine.py` is a hypothesis-testing tool, not proof of alpha.
- This is a **research signal generator**, not an execution system, by design — mixing
  unproven NLP signals with live capital is how you lose both money and credibility.
- Stance-taxonomy weights in `signals/mapper.py` are documented priors from Fed-speak
  research, not fitted parameters — v1, meant to be replaced once real backtests exist.
