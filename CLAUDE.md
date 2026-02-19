# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight personal AI assistant framework (~4,000 lines of core agent code) published on PyPI as `nanobot-ai`. It connects to LLM providers via LiteLLM and routes messages through chat platforms (Telegram, Discord, WhatsApp, Slack, Feishu, DingTalk, Email, QQ, Mochat).

**Stack**: Python 3.11+, Pydantic v2, Typer/Rich CLI, LiteLLM, asyncio throughout. TypeScript for WhatsApp bridge (`bridge/`). Build system: hatchling via `pyproject.toml`.

## Commands

```bash
# Install (dev)
pip install -e .

# Tests
pytest                                    # all tests
pytest tests/test_tool_validation.py      # single file
pytest tests/test_foo.py::test_bar        # single test

# Lint & format (ruff: line-length=100, py311, rules E/F/I/N/W, E501 ignored)
ruff check .            # lint
ruff check --fix .      # lint + auto-fix
ruff format .           # format

# Run
nanobot agent           # interactive chat
nanobot agent -m "msg"  # single message
nanobot gateway         # start gateway with channels
nanobot status          # show status

# Line count verification
bash core_agent_lines.sh
```

## Architecture

Message bus pattern with clear separation:

```
Channels --> MessageBus (async queues) --> AgentLoop --> LLM Provider
                ^                              |
                |                              v
                +------ OutboundMessage <-- ToolRegistry
```

### Key packages (under `nanobot/`)

- **`agent/`** — Core agent loop (`loop.py` is the main file, ~21KB), context builder, memory (two-layer: `MEMORY.md` facts + `HISTORY.md` append-only log), skills loader, subagent spawning
- **`agent/tools/`** — Built-in tools with abstract `Tool` base class (`base.py`) and `ToolRegistry` (`registry.py`). Tools return strings, never raise exceptions to caller
- **`bus/`** — Async message queue decoupling channels from agent (`InboundMessage`/`OutboundMessage` events, `MessageBus` with async queues)
- **`channels/`** — Chat platform integrations. `BaseChannel` ABC with `start()`/`stop()`/`send()`/`is_allowed()`. Implementations are lazily imported only when enabled
- **`cli/`** — Typer CLI in a single `commands.py` file (~1000 lines)
- **`config/`** — Pydantic config schema (`schema.py`) and JSON loader (`loader.py`). Config lives at `~/.nanobot/config.json`. camelCase in JSON, snake_case in Python (Pydantic alias generator)
- **`providers/`** — LLM provider abstraction. `LLMProvider` ABC with `chat()` returning `LLMResponse`. `ProviderSpec` registry in `registry.py` is the single source of truth for all providers
- **`session/`** — JSONL-based conversation persistence with append-only history and consolidation offset tracking
- **`skills/`** — Bundled `SKILL.md` markdown files with YAML frontmatter, loaded progressively (summaries first, full on demand)

### Adding new components

- **New LLM provider**: (1) Add `ProviderSpec` to `PROVIDERS` in `providers/registry.py`, (2) Add field to `ProvidersConfig` in `config/schema.py`. Everything else (env vars, model prefixing, status display) is automatic.
- **New tool**: Extend `Tool` ABC in `agent/tools/`, register in `AgentLoop._register_default_tools()`
- **New channel**: Extend `BaseChannel` in `channels/`, add to `ChannelManager` lazy imports

### Key conventions

- The project tracks core line count (~3,761 lines) explicitly excluding `channels/`, `cli/`, and `providers/`
- Agent loop is iterative tool-use (max 20 iterations default) with `on_progress` callback for streaming
- `process_direct()` on AgentLoop is for CLI/cron usage without the message bus
- Subagents (via `SpawnTool`) get isolated tool registries (no message/spawn tools)
- Workspace bootstrap files: `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `HEARTBEAT.md` in `workspace/`
- Logging via `loguru`, disabled by default unless `--logs` flag
- Default gateway port: 18790
- Tests use `pytest` with `asyncio_mode="auto"` and `typer.testing.CliRunner`
