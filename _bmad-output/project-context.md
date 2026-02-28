---
project_name: 'cc-for-non-coders-dev-container'
user_name: 'Чапаев'
date: '2026-02-28'
sections_completed: ['technology_stack', 'language_rules', 'framework_rules', 'testing_rules', 'code_quality', 'workflow_rules', 'critical_rules']
status: 'complete'
rule_count: 47
optimized_for_llm: true
---

# Project Context for AI Agents

_This file contains critical rules and patterns that AI agents must follow when implementing code in this project. Focus on unobvious details that agents might otherwise miss._

---

## Technology Stack & Versions

### Container Base
- **OS:** Ubuntu 22.04
- **User:** `coder` (non-root, sudo NOPASSWD)
- **Entrypoint:** `dumb-init` → `entrypoint.sh`
- **Resource limits:** 2 CPU, 4 GB RAM

### Runtime
- **Node.js:** 22 (NodeSource) — requires `NODE_PATH="/usr/lib/node_modules"` for global modules
- **Python:** 3 (system, Ubuntu 22.04)
- **code-server:** 4.109.2
- **Claude Code CLI:** latest (global npm)

### API Configuration (Critical)
- **Endpoint:** `https://api.z.ai/api/anthropic` (Z.AI proxy, NOT direct Anthropic API)
- **Models:** GLM-5 (mapped to Opus & Sonnet), GLM-4.5-Air (mapped to Haiku)
- **Timeout:** 3,000,000 ms
- **Dual API keys:** primary + backup, switchable via `switch-api-key.sh`

### Key Dependencies (npm — global)
- `docx`, `pptxgenjs` — document generation
- `parcel`, `@parcel/config-default`, `html-inline` — HTML bundling

### Key Dependencies (pip)
- `anthropic>=0.39.0`, `mcp>=1.1.0` — AI/MCP SDK
- `playwright` + Chromium — browser automation
- `pandas`, `numpy`, `matplotlib` — data & charts
- `pypdf`, `pdfplumber`, `reportlab`, `pdf2image` — PDF
- `python-docx`, `python-pptx`, `openpyxl` — office formats
- `pillow`, `cairosvg`, `imageio` — graphics
- `defusedxml`, `PyYAML`, `lxml` — parsing

### System Tools
- LibreOffice (Writer, Calc, Impress) — format conversion
- FFmpeg — video/GIF
- Pandoc — document conversion
- Tesseract OCR + Russian language pack
- Poppler-utils, qpdf — PDF utilities

## Critical Implementation Rules

### Language-Specific Rules

**Python:**
- Infrastructure code (auth-gateway.py) uses **stdlib only** — no external frameworks
- Always use `#!/usr/bin/env python3` shebang
- Use `defusedxml` for XML parsing (installed), never raw `xml.etree`
- Encoding: UTF-8 everywhere (`LANG=C.UTF-8`, `LC_ALL=C.UTF-8`)
- Style: functional, minimal class abstractions (except HTTP handler)

**Bash:**
- Always start with `set -euo pipefail` (strict mode)
- Use `#!/usr/bin/env bash` shebang
- Heredocs for config generation (`.env`, `.bashrc` blocks)
- Background services via `&`, main process via `exec`

**JavaScript/Node.js:**
- Node.js 22 — both ES modules and CommonJS supported
- Global npm packages require `NODE_PATH="/usr/lib/node_modules"` — without this, `require()` fails
- Scripts in skills use `node` directly (no npm scripts, no package.json)
- No TypeScript — plain JavaScript only

**Bilingual Convention:**
- Code, configs, variable names, CLI output → **English**
- User-facing content, README, course materials, comments for users → **Russian**
- No linters/formatters configured (no ESLint, Prettier, Black) — maintain consistency with existing style
- No root-level package.json — this is NOT an npm project

### Architecture & Framework Rules

**Container Architecture (3 services behind reverse-proxy):**
- `auth-gateway.py` (:8080) — sole external entry point, HMAC-SHA256 cookie auth
- `code-server` (:8081) — internal only, no own auth
- `File Browser` (:9090) — internal only, noauth mode, baseurl=/files/
- Adding a new service: start in `entrypoint.sh` before `exec`, add proxy rule in `auth-gateway.py`

**Routing (auth-gateway.py):**
- `/ide/*` → strip prefix, proxy to :8081
- `/files/*` → keep prefix (baseurl-aware backend), proxy to :9090
- WebSocket relay for code-server terminal (raw TCP socket, bidirectional, threaded)
- `ThreadedHTTPServer` — each request in its own thread
- If `PASSWORD` env is empty — auth is skipped entirely (dev mode)

**Skills Framework (`skills/`):**
- Each skill = directory with mandatory `SKILL.md`
- Skills are copied to 3 locations at build: `~/.claude/skills/`, `course/.claude/skills/`, `.course-image/.claude/skills/`
- Complex skills contain `scripts/`, `references/`, `templates/` subdirectories
- SKILL.md = complete Claude Code instructions (prompts, steps, rules, triggers)

**Demo Projects (`course/sessions/`):**
- 5 sessions × N demos = 28 demos total
- Each demo: `README.md` (instructor guide) + data files (CSV/MD/JSON)
- Some demos have their own `CLAUDE.md` for AI context

**Volume Strategy:**
- Pristine image at `/home/coder/.course-image/` (baked into Docker image)
- Working data at `/home/coder/course/` (named volume `student-data`)
- First run: copy from pristine → volume (marker: `course/.initialized`)
- Reset: `docker compose down -v` destroys volume, next start re-copies

### Testing Rules

**No automated tests exist.** Testing is entirely manual:

1. `docker compose build` — must complete without errors
2. `docker compose up -d` → healthcheck at `/healthz` returns 200
3. Auth via `/login` with password from `.env`
4. Verify code-server at `/ide/`, File Browser at `/files/`
5. Run `claude` in terminal — must connect to API
6. Invoke a skill (e.g. `/pdf`) — must work

**Healthcheck (docker-compose.yml):**
- `curl -f http://localhost:8080/healthz` — interval 30s, timeout 5s, 3 retries

**Rules for agents:**
- Do NOT write tests unless explicitly asked
- Validate infrastructure changes by rebuilding the Docker image
- If adding a new service or changing routing — verify healthcheck still passes

### Code Quality & Style Rules

**Project Structure:**
- Root = infrastructure (Dockerfile, entrypoint.sh, auth-gateway.py, docker-compose.yml)
- `course/` = course materials (sessions, demos)
- `skills/` = Claude Code skills (19 total)
- `docs/` = generated project documentation
- Flat root structure — no nested `src/`, `lib/`, `config/` directories

**File Naming (kebab-case everywhere):**
- Infrastructure: `auth-gateway.py`, `code-server-settings.json`
- Skill directories: `slack-gif-creator/`, `web-artifacts-builder/`
- Demo directories: `financial-dashboard/`, `crm-cleanup/`
- Sessions: numeric prefix + kebab-case (`01-setup/`, `02-context-skills/`)

**Mandatory Files:**
- Each skill: `SKILL.md` + `LICENSE.txt`
- Each demo: `README.md` (instructor guide)

**Data Formats:**
- CSV: comma separator, UTF-8
- Markdown: GitHub Flavored Markdown
- JSON: 2-space indent
- YAML: 2-space indent

**Documentation Language:**
- Project README.md → Russian
- Code comments in infrastructure → English (docstrings in auth-gateway.py)
- `course/CLAUDE.md` → English (Claude Code instructions)
- Root `CLAUDE.md` → minimal (near-empty)

### Development Workflow Rules

**Git:**
- Default branch: `main`
- Workflow: fork → feature branch → PR to main
- Branch naming: `type/description` (e.g. `planning/docs-and-api-providers`)
- Commit messages: concise, Russian or English, optional prefix (`docs:`, `fix:`)
- No pre-commit hooks, no CI/CD pipeline

**Build & Deploy:**
- `./run.sh` — one-liner: count files → `docker compose build` → `docker compose up -d` → print URL
- Dockerfile changes → full rebuild required (`docker compose build --no-cache`)
- entrypoint.sh / auth-gateway.py changes → container restart required
- course/ or skills/ changes → rebuild required (files are COPYed in Dockerfile)

**Extension Patterns:**
- New system package → add to `apt-get install` block in Dockerfile
- New Python package → add to `pip3 install` block in Dockerfile
- New npm package → add to `npm install -g` block in Dockerfile
- New skill → directory in `skills/` with `SKILL.md` + update README.md
- New demo → directory in `course/sessions/XX-topic/demo/` with `README.md` + data
- New VS Code extension → `code-server --install-extension` in Dockerfile

**Environment Variables:**
- Defined in `.env` (template: `.env.example`)
- Flow: `.env` → docker-compose.yml → entrypoint.sh → `~/.claude/.env`
- Sensitive data (API keys) — `.env` only, never hardcode

### Critical Don't-Miss Rules

**Claude Code Configuration Priority (CRITICAL):**
- `claude-settings.json` `"env"` section **overrides** OS environment variables (Docker ENV)
- `ANTHROPIC_BASE_URL` is hardcoded in `claude-settings.json` and **takes precedence** over docker-compose.yml `environment:` block
- `.claude/.env` file is **NOT read automatically** by Claude Code CLI — it exists only for `switch-api-key.sh` script which rewrites it, but Claude Code gets env from `claude-settings.json` → OS env
- Effective priority: `claude-settings.json "env"` > Docker ENV > `.env` file (not read by CLI)

**Anti-Patterns (NEVER do):**
- Do NOT use `curl` / `wget` inside container scripts — blocked in `claude-settings.json`
- Do NOT run `rm -rf /`, `sudo`, `chmod 777` — forbidden
- Do NOT hardcode API keys — `.env` → env vars only
- Do NOT switch to root in Dockerfile after `USER coder` — all subsequent ops run as `coder`
- Do NOT install packages at runtime (`pip install` / `npm install`) — Dockerfile build-time only
- Do NOT modify `/home/coder/.course-image/` — pristine copy, read-only by design

**Edge Cases:**
- Node.js 22 no longer auto-resolves global modules — always need `NODE_PATH="/usr/lib/node_modules"`
- `auth-gateway.py` with empty `PASSWORD` skips auth entirely — intentional dev mode, not a bug
- WebSocket relay uses raw sockets — when modifying proxy logic, don't forget the WebSocket branch
- File Browser uses `baseurl=/files/` — prefix is NOT stripped during proxying (unlike `/ide/`)
- `entrypoint.sh` rewrites `switch-api-key.sh` on every start — do not manually edit this file inside container
- `code-server --auth none` — safe because it's behind auth-gateway; NEVER expose :8081 externally

**Security:**
- Single external port: 8080 (auth-gateway)
- :8081 and :9090 — localhost only inside container
- HMAC-SHA256 cookie (secret derived from password)
- `claude-settings.json` — allow/deny lists for Claude Code commands
- `defusedxml` for XML parsing (XXE protection)

**Content Rules (from course/CLAUDE.md):**
- Apply humanizer guide — avoid AI cliches (inflated significance, promotional language, rule-of-three lists, em dash overuse)
- Specific details over vague claims
- Vary sentence structure naturally
- Tone: conversational but competent — not corporate, not overly casual
- No emojis in course materials unless explicitly requested

---

## Usage Guidelines

**For AI Agents:**
- Read this file before implementing any code
- Follow ALL rules exactly as documented
- When in doubt, prefer the more restrictive option
- Pay special attention to "Critical Don't-Miss Rules" section

**For Humans:**
- Keep this file lean and focused on agent needs
- Update when technology stack or patterns change
- Remove rules that become obvious over time

Last Updated: 2026-02-28
