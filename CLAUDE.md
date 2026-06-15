# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

AI-Enhanced Cybersecurity Threat Detector — a single-machine Docker-based SOC pipeline that simulates a real-world Security Operations Center. Kali Linux and Ubuntu Server containers generate attacker/victim traffic, a BERT-based transformer detects anomalies, alerts flow to both Splunk Free (local) and Microsoft Sentinel (cloud), XSOAR automates response with VirusTotal and AbuseIPDB IOC enrichment, and Claude API generates CTI incident reports. A React dashboard visualizes everything.

## Architecture: Pipeline Layers

```
Simulation  →  Capture  →  Detection  →  Dual-SIEM  →  SOAR  →  Reporting  →  Frontend
Kali/Ubuntu    Wireshark    Preprocess     Splunk HEC    XSOAR    Claude API    React
containers     PCAP files   → Transformer  Azure Monitor  plays    CTI reports   dashboard
                            → Alert engine  + Defender    books
```

**Key integration points:**
- `forwarder/forwarder.py` fans out every alert to both Splunk HEC and Azure Monitor HTTP API simultaneously
- XSOAR is triggered by a Splunk webhook on High/Critical alerts, then queries Sentinel to pull correlated Defender (MDE) endpoint telemetry before enrichment
- Claude receives the full enriched context (anomaly score + Defender telemetry + Sentinel events + VirusTotal + AbuseIPDB) and returns a BLUF-format CTI report
- `datasets/` is gitignored (10+ GB); `model/weights/` is gitignored (large binaries)

## Directory Map

| Path | Purpose |
|------|---------|
| `attacker/` | Kali Linux container — Dockerfile + attack scripts |
| `victim/` | Ubuntu Server container — Dockerfile + MDE onboarding scripts |
| `capture/` | tshark/Wireshark automation for rolling PCAP export |
| `preprocessing/` | PCAP parser + feature extractor (IP, port, proto, timing, TCP flags) |
| `model/` | BERT-based transformer — training script, inference, weights (gitignored) |
| `api/` | FastAPI service exposing `/predict` — anomaly score + severity tier |
| `forwarder/` | Dual-SIEM forwarder — Splunk HEC + Azure Monitor HTTP API |
| `splunk/` | Splunk config (indexes.conf, inputs.conf) + SPL search files |
| `sentinel/` | KQL analytics rules, Sentinel workbook JSON, Azure/MDE setup guide |
| `xsoar/` | XSOAR playbook YAML exports |
| `claude/` | Claude API integration + prompt templates for CTI report generation |
| `frontend/` | React dashboard (Node.js/Express backend + Recharts visualizations) |
| `datasets/` | UNSW-NB15 (~2.5 GB) + CICIDS2017 (~8 GB) — gitignored, download locally |

## Commands

```bash
# Spin up entire local stack (Phases 1–4 services)
docker compose up -d

# Spin up specific service
docker compose up -d splunk

# Tear down
docker compose down

# Model training (run before containerizing inference)
cd model && python train.py

# Run FastAPI inference service locally (outside Docker)
cd api && uvicorn main:app --reload --port 8000

# Run dual-SIEM forwarder
cd forwarder && python forwarder.py

# Run preprocessing on a PCAP file
cd preprocessing && python pcap_parser.py --input ../captures/sample.pcap

# Frontend dev server
cd frontend && npm install && npm start

# Frontend production build
cd frontend && npm run build
```

## Environment Setup

Copy `.env.example` to `.env` and fill in all keys before running anything. The `.env` file is gitignored — never commit it.

**Phase-gated keys** (don't need all keys before Phase 1):
- Phase 2+: none (local only)
- Phase 3+: Azure, Sentinel workspace, MDE, Splunk HEC token
- Phase 4+: VirusTotal, AbuseIPDB, XSOAR
- Phase 5+: Anthropic API key

**Critical:** Activate Microsoft Sentinel and Defender for Endpoint trials only at the start of Phase 3 to maximize the 31-day free window.

## Detection Model

- Base: BERT-style transformer (Hugging Face `transformers`, PyTorch)
- Training data: UNSW-NB15 (primary) + CICIDS2017 (supplemental)
- Features: src/dst IP, port, protocol, packet size, inter-arrival timing, TCP flags, session metadata
- Output: anomaly score 0.0–1.0 → severity tier (Low/Medium/High/Critical) via configurable thresholds in `.env`
- Served via FastAPI `/predict` endpoint; alert engine fires when score exceeds threshold

## Microsoft Stack Notes

- Wireshark runs on the **host OS** (not in Docker) — requires Npcap on Windows, captures the Docker bridge interface
- MDE Linux agent is installed **inside the Ubuntu victim container** — generates EDR telemetry independent of network capture
- Sentinel receives alerts via two paths: Azure Monitor HTTP API (from forwarder) + native MDE connector (from Defender portal)
- Azure Resource Group: `ai-threat-detector-rg` — all Azure resources live here for easy cleanup

## Claude API Usage

- Model: `claude-sonnet-4-6`
- Invoked only on High/Critical severity cases to manage cost (~$10/month budget)
- Report format: BLUF summary → threat assessment → IOC table → MITRE ATT&CK tags → recommended actions
- Prompt templates live in `claude/prompt_templates/`
