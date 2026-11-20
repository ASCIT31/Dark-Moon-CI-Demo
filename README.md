<div align="center">

# DarkMoon, the open-source AI penetration testing platform

**50 specialist AI agents. Real exploits chained across web, cloud, Active Directory and Kubernetes. Proof for every finding. Self-hosted, and the model never sees your real IPs, hosts or credentials.**

[![GitHub stars](https://img.shields.io/github/stars/ASCIT31/Dark-Moon?style=social&label=Star)](https://github.com/ASCIT31/Dark-Moon)

[**⭐ Star DarkMoon**](https://github.com/ASCIT31/Dark-Moon) · [**▶️ Watch the 60s demo**](https://youtu.be/1bFRVuMkZzY?si=peKxwuxzbXBnb2zO) · [**Main repo**](https://github.com/ASCIT31/Dark-Moon)

</div>

[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-0A2472)](LICENSING.md)

> [!IMPORTANT]
> **License change.** This repository is licensed under the GNU AGPL v3.0 since 20 November 2026 (commits up to `4b606b3` had no licence file). Darkmoon Community is moving to the GNU AGPL v3.0 too (its commits up to `edfe9cb`, 27 October 2026, and earlier outside contributions stay GPL v3.0). Details: [LICENSING.md](LICENSING.md) · [Darkmoon LICENSING.md](https://github.com/ASCIT31/Dark-Moon/blob/master/LICENSING.md).

---

# Dark Moon CI — Automated Pentest Pipeline

> Autonomous black-box penetration testing triggered by GitHub Actions.  
> Powered by **Darkmoon AI** — no manual setup, no docker-compose.ci.yml, no complex configuration.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [How it Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Repository Secrets](#repository-secrets)
- [Self-Hosted Runner Setup](#self-hosted-runner-setup)
- [Usage](#usage)
- [install.sh — The Key to Simplicity](#installsh--the-key-to-simplicity)
- [Workflow Details](#workflow-details)
- [Artifacts](#artifacts)
- [Security Notes](#security-notes)

---

## Overview

This repository provides a **ready-to-use GitHub Actions CI pipeline** for autonomous penetration testing using the Darkmoon platform.

**One trigger. One prompt. One report.**

```
workflow_dispatch → TARGET: 192.168.1.1 → full pentest → report artifact
```

No need to:
- Write a `docker-compose.ci.yml`
- Manage environment variables manually
- Configure OpenCode or agents
- Handle Docker networking

Everything is handled by `install.sh`.

---

## Architecture

```
GitHub Actions
    │
    ▼
Self-Hosted Runner (your machine / VPS)
    │
    ├── curl install.sh          ← downloads & configures Darkmoon
    │       │
    │       ├── docker pull ascit/opencode-darkmoon-pro
    │       ├── docker pull ascit/darkmoon
    │       ├── writes .opencode.env (LLM provider config)
    │       ├── writes .darkmoon-license.env
    │       └── docker compose up -d
    │
    ├── ./darkmoon.sh run --agent pentest "TARGET: ..."
    │       │
    │       └── opencode exec → pentest agent → MCP tools → findings → report
    │
    └── docker cp reports/ → GitHub Artifact
```

---

## How it Works

### 1. `install.sh` — Single command setup

`install.sh` (hosted at `client.dark-moon.org`) handles the **entire Darkmoon stack setup**:

- Pulls Docker images from Docker Hub
- Generates `.opencode.env` from CLI args (no interactive form in CI)
- Generates `.darkmoon-license.env`
- Extracts policy file from image
- Starts `docker compose up -d`

**Usage in CI:**
```bash
curl -fsSL https://client.dark-moon.org/install.sh -o install.sh
chmod +x install.sh
./install.sh "$LICENSE_KEY" \
  --provider "$OPENROUTER_PROVIDER" \
  --model "$OPENCODE_MODEL" \
  --api-key "$OPENROUTER_API_KEY"
```

**Non-interactive CLI args:**

| Arg | Description | Example |
|-----|-------------|---------|
| `LICENSE_KEY` | Darkmoon license (positional) | `XXXXX-XXXXX-XXXXX-XXXXX` |
| `--provider` | LLM cloud provider | `anthropic` |
| `--model` | Model name | `claude-opus-4-6` |
| `--api-key` | Provider API key | `sk-ant-xxx` |
| `--local` | Enable local model mode | — |
| `--local-engine` | Local engine | `ollama` / `llama.cpp` / `custom` |
| `--local-url` | Local server base URL | `http://localhost:8001/v1` |
| `--local-model` | Local model name | `qwen2.5-coder-7b` |
| `--init` | Force provider reconfiguration | — |
| `--help` | Show usage | — |

**Idempotent:** If `.opencode.env` already exists and is valid, `install.sh` skips the configuration form entirely. Subsequent runs are faster.

---

### 2. `darkmoon.sh` — The run interface

`darkmoon.sh` is the CLI wrapper around `docker compose exec opencode`. It forwards all arguments directly to OpenCode inside the container.

```bash
./darkmoon.sh run --agent pentest "TARGET: 192.168.1.1"
```

This is equivalent to running inside the container:
```bash
opencode run --agent pentest "TARGET: 192.168.1.1"
```

**Modes:**
```bash
./darkmoon.sh                          # Interactive TUI
./darkmoon.sh run --agent pentest "…" # Headless run (CI)
./darkmoon.sh --log <session_id>       # Monitor live session
```

---

### 3. The pentest agent

The `pentest` agent is a multi-plane orchestrator:

- Performs environmental discovery and technology fingerprinting
- Dispatches specialized sub-agents (nodejs, php, graphql, headless-browser, etc.)
- Follows OWASP / NIST SP 800-115 / MITRE ATT&CK methodology
- Generates a structured Markdown report with:
  - **Management Summary** (non-technical, CEO-readable, max 1 page)
  - Executive Summary
  - Per-finding sections (CVSS, evidence, remediation)
  - Risk classification and remediation roadmap

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Docker + Docker Compose v2 | Running on the runner machine |
| x86_64 architecture | ARM / macOS not supported |
| Darkmoon license key | From `client.dark-moon.org` |
| Docker Hub credentials | `mehdiboutayeb` account |
| LLM API key | Anthropic, OpenAI, OpenRouter, or local model |
| Self-hosted GitHub runner | Registered on this repository |

---

## Repository Secrets

Configure these secrets in **Settings → Secrets and variables → Actions**:

| Secret | Description | Example |
|--------|-------------|---------|
| `DARKMOON_LICENSE_KEY` | Darkmoon license key | `XXXXX-XXXXX-XXXXX-XXXXX` |
| `DOCKERHUB_USERNAME` | Docker Hub username | `mehdiboutayeb` |
| `DOCKERHUB_TOKEN` | Docker Hub access token | `dckr_pat_xxx` |
| `OPENROUTER_PROVIDER` | LLM provider name | `anthropic` |
| `OPENCODE_MODEL` | Model name | `claude-opus-4-6` |
| `OPENROUTER_API_KEY` | Provider API key | `sk-ant-xxx` |

> **Security:** Never log secrets. The workflow uses `***` masking for all secret values. No secret appears in job logs.

---

## Self-Hosted Runner Setup

### Register the runner

```bash
# Download runner (if not already done)
mkdir ~/actions-runner && cd ~/actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Get registration token from GitHub:
# Settings → Actions → Runners → New self-hosted runner

# Configure for this repo
./config.sh \
  --url https://github.com/ASCIT31/Dark-Moon-CI-Demo \
  --token <REGISTRATION_TOKEN> \
  --name darkmoon-runner-demo \
  --labels "darkmoon-runner-demo,self-hosted,Linux,X64" \
  --unattended

# Start the runner
./run.sh
```

### Runner requirements

- Docker daemon running
- User in `docker` group (or run as root)
- Internet access (to pull images + call LLM API)
- Sufficient RAM: minimum **8GB** recommended (16GB for large models)

> **WSL note:** The runner must be started manually in a terminal (`./run.sh`) and kept running during CI jobs. It cannot run as a background daemon reliably in WSL.

---

## Usage

### Trigger a pentest

1. Go to **Actions** → **darkmoon** → **Run workflow**
2. Fill in the `Target` field:

```
TARGET: https://target.example.com
```

Or with full options:
```
TARGET: https://target.example.com NOISE=low SEVERITY=critical
```

3. Click **Run workflow**

### Monitor progress

While the job runs, you can monitor it live from your machine:

```bash
./darkmoon.sh --log <session_id>
```

The session ID is displayed in the job logs after initialization.

---

## install.sh — The Key to Simplicity

Before this approach, CI pipelines required:
- A `docker-compose.ci.yml` generated inline in the workflow
- Manual environment variable injection
- Complex Docker networking management
- Separate steps for pulling, starting, and configuring

**Now:**
```yaml
- name: Download & install Darkmoon
  run: |
    curl -fsSL https://client.dark-moon.org/install.sh -o install.sh
    chmod +x install.sh
    ./install.sh "${{ secrets.DARKMOON_LICENSE_KEY }}" \
      --provider "${{ secrets.OPENROUTER_PROVIDER }}" \
      --model "${{ secrets.OPENCODE_MODEL }}" \
      --api-key "${{ secrets.OPENROUTER_API_KEY }}"
```

That's it. One step. Everything else is automatic.

**What install.sh does internally:**
1. Checks Docker prerequisites
2. Reads/saves existing `.opencode.env` (idempotent)
3. Stops existing stack + purges images
4. Downloads `docker-compose.yml` and `darkmoon.sh` from `client.dark-moon.org`
5. Writes `.opencode.env` with LLM provider config (from CLI args or interactive form)
6. Writes `.darkmoon-license.env`
7. `docker compose pull`
8. Extracts policy file from image (anti-tamper guard)
9. `docker compose up -d`

---

## Workflow Details

### `darkmoon.yml` — Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      prompt:
        description: "Target (e.g. TARGET: 172.18.0.3)"
        default: "TARGET: 172.18.0.3"
```

Passes secrets and input to `template.yml`.

### `template.yml` — Execution

```
Login Docker Hub
    → Download & install Darkmoon (install.sh with CLI args)
        → docker pull images
        → docker compose up -d
    → Run pentest (darkmoon.sh run --agent pentest "TARGET: ...")
        → opencode pentest agent
        → sub-agents (nodejs, graphql, headless-browser, ...)
        → report written to /root/.local/share/opencode/reports/
    → Collect reports (docker cp from container)
    → Upload artifact
```

### Report collection

Reports are written **inside the container** to tmpfs (`/root/.local/share/opencode/reports/`). They must be extracted via `docker cp` before the container stops:

```yaml
- name: Collect reports from container
  if: always()
  run: |
    mkdir -p reports
    docker cp opencode:/root/.local/share/opencode/reports/. reports/ 2>/dev/null || true
```

---

## Artifacts

After each run, a **darkmoon-reports** artifact is uploaded containing:

- `pentest_report_<target>_<datetime>.md` — full Markdown report

**Report structure:**
```
## Management Summary          ← Non-technical, CEO-readable (max 1 page)
## Executive Summary           ← Technical overview
## Vulnerability Summary       ← Severity table
## Technology Stack            ← Detected technologies
## Detailed Findings           ← One section per vuln (CVSS, evidence, remediation)
## Cross-Plane Correlation     ← Attack chains
## Risk Classification         ← CVSS scores
## Remediation Roadmap         ← Prioritized action table
```

---

## Security Notes

- **No secrets in logs:** All secrets are masked by GitHub Actions (`***`)
- **No secrets in docker-compose:** Credentials are passed via `.opencode.env` (not in `docker-compose.yml`)
- **License isolation:** `.darkmoon-license.env` and `.opencode.env` are generated at runtime, never committed
- **Runner isolation:** Use a dedicated runner machine. Do not share runners between sensitive projects
- **Network:** The `opencode` container runs on a bridge network (`darkmoon-stack_default`). For local model inference, ensure the inference server is on the same Docker network

---

## License

This repository is licensed under the GNU AGPL v3.0 (AGPL-3.0-only), see [LICENSE](LICENSE) and [LICENSING.md](LICENSING.md). Commits up to `4b606b3` had no licence file. Darkmoon itself is AGPL-3.0, see its [LICENSING.md](https://github.com/ASCIT31/Dark-Moon/blob/master/LICENSING.md).

---

*Darkmoon — dark-moon.org | contact@asc-it.fr*