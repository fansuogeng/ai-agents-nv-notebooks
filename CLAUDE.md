# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Practice notebooks for the Nebius Academy × NVIDIA Agentic AI course. Modules 1–4 are standalone Jupyter
notebooks (designed for Colab, linked from the root `README.md`); module 6 is a deployable container image.
There is no build system, test suite or linter at the repo level.

## Modules 1–4: notebooks

- Each notebook installs its own deps in its first cell (`pip install -q ...`) and reads `NEBIUS_API_KEY`
  (plus `TAVILY_API_KEY` in modules 2 and 4) from the environment / `.env` via `python-dotenv`.
- LLM calls go to Nebius Token Factory, an OpenAI-compatible API. Module 1 uses
  `https://api.tokenfactory.us-central1.nebius.com/v1/`; modules 2–4 use `https://api.tokenfactory.nebius.com/v1/`.
- Stack: `openai` SDK (module 1), Tavily (module 2), `deepagents` (module 3), `langchain-nebius` / LLM-as-a-judge (module 4).
- When adding a notebook, also add its GitHub + "Open in Colab" links to the root `README.md`.
- Local setup: a uv-managed `.venv` at the repo root, registered as the Jupyter kernel "Python (ai-agents-nv)".
  Secrets live in the root `.env` (gitignored).

## Module 6: OpenShell Devlab image (`module-6/openshell-devlab/`)

Three trust layers, nested:

1. **Outer Devlab container** (`Dockerfile`, runs as root by design). `start-openshell.py` is the entrypoint:
   requires `JUPYTER_TOKEN` (refuses to start without it), seeds `/opt/research` into `/workspace/research`
   without overwriting user edits, starts a private `dind` daemon on `/run/openshell-docker.sock`, then the
   OpenShell gateway (`gateway.toml`, ports 17670/17671), registers it as `devlab`, then JupyterLab on 8888.
   API keys are stripped from the env of dockerd/gateway.
2. **Trusted wrapper** `research/openshell/run.py` (stdlib only). Subcommands: `setup` (import provider
   profiles from `providers/*.yaml`, create providers from `TOFA_API_KEY`/`TAVILY_API_KEY`), `build`
   (inner image `openshell-deep-research-agent:course-1` from `research/Dockerfile.openshell`), `run "<question>"
   [--delete]`, `inspect`, `demo` (denial checks), `delete`. It renders `policy.yaml`, substituting
   `${TOKEN_FACTORY_REGION}` (default `us-central1`).
3. **Sandboxed agent** (`research/main.py`, `agents.py`, `models.py`, `tools.py`, `prompts.py`): a `deepagents`
   lead agent + subagents (competitor-scout, competitor-researcher, fact-checker). Outputs go to
   `runs/<run-id>/` via `FilesystemBackend(virtual_mode=True)`. Deps are locked in `research/uv.lock`.

Invariants to keep in sync when changing any of these:
- `policy.yaml` is a network allowlist: only Nebius `POST /v1/chat/completions` and Tavily `POST /search`.
  The Token Factory host in `models.py` (`api.tokenfactory.{TOKEN_FACTORY_REGION}.nebius.com`) must match the
  policy host; adding any new external call requires a policy rule.
- Credentials reach the sandbox only as OpenShell providers, never as plain sandbox env or files. The
  agent reads `TOFA_API_KEY` (not `NEBIUS_API_KEY`). Non-secret config passed through: `LEAD_MODEL`,
  `WORKER_MODEL`, `SEARCH_EXCLUDE_DOMAINS`, `TOKEN_FACTORY_REGION`.
- OpenShell CLI and gateway are pinned to v0.1.1 (with sha256 checks) in `Dockerfile` and in `gateway.toml`;
  base images are pinned by digest.

### Build / deploy

```bash
cd module-6/openshell-devlab
./build-and-push.sh build                                   # local, linux/amd64 only
IMAGE=cr.<region>.nebius.cloud/<registry-id>/openshell-devlab TAG=<new-tag> ./build-and-push.sh push
```

Full Nebius flow (see `module-6/README.md`): create VM in UI → `./prepare-vm.sh` locally (creates
`devlab-build-sa`, copies code, writes `demo.env`) → `./demo.sh` on the VM (calls `registry-build-push.sh`)
→ create the Devlab in the UI → `./cleanup.sh` (not idempotent; run before a rerun).

- Never reuse a pushed `TAG`.
- The image needs nested Docker + Landlock, so it won't run on Docker Desktop for macOS. Test on a Linux VM:
  `docker run --privileged -p 8888:8888 -e JUPYTER_TOKEN=... <image>`.
- Debug a deployed Devlab with `nebius ai devlab logs <devlab-id> --tail 50`; healthy startup logs
  `OpenShell runtime is ready`.
