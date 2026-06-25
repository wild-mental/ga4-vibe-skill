![AI Skills for Everyone](author/wildmental-bjpark.png)

# ga4-vibe
> Skill for Cursor, Claude, Codex agents

**Language / 언어:** [한국어](README.md) · [English](README.en.md)

**A Skill that reproduces — with the same procedure in any project — the 4-deliverable pipeline that connects your app's existing event-logging and analytics-dashboard system to Google Analytics 4 (+BigQuery). Invoked with `/ga4-vibe`. Supports Cursor, Claude Code, and Codex.**

The core principle is **no guessing** — scan the code first to pin everything to facts, ask when facts are missing, and derive docs and mapping tables from the code. GA4 event names, parameters, and transport paths match **1:1 with what the actual code sends**, so it is accurate from the first pass.

---

## What this skill solves

### 1. Produces 4 tracks in one procedure

| Code | Track | Deliverable | One line |
|---|---|---|---|
| **T-APP** | Actual app integration | Client tag + server transport code + env + tests | Real events flow into GA4 |
| **T1** | Dummy traffic generator | Deterministic (seed-based) synthetic user-funnel script (dry-run by default) | Validate the analytics environment with data |
| **T2** | Setup guide | Educational GA4 integration & dashboard doc (cites official URLs) | Just follow along and it connects |
| **T3** | Warehouse analysis | GA4→BigQuery linking guide + EDA/statistics SQL | Find patterns in the loaded data |

Flow: **Phase A scan/gate → B lock decisions → C implement (independent tracks in parallel) → D aztks evaluate & remediate → E verify & deliver.**

### 2. Phase A — automatic context scan + sufficiency gate (no guessing)

Before implementing, it directly scans the app's **event catalog, collection paths, transport hooks, dashboard, env schema, root layout, and toolchain** via grep/glob and pins them to variables. It must pass the sufficiency gate before starting; if anything is missing it **stops** and either ① requests materials or ② poses multiple-choice implementation-decision questions (`AskUserQuestion`, recommended option as #1).

### 3. Safe locked defaults

- **Aggregation-path separation (prevents double counting)** — client tag = automatic `page_view`, server Measurement Protocol = post-authentication behavioral events. The same event is never sent over both paths.
- **env gating / no-op** — active only when both the measurement ID and the server secret are set; if unset, a full no-op (build- and runtime-safe); if only one side is set, throw (makes misconfiguration visible).
- **Secret separation** — the measurement ID (`G-XXXX`) is public; the MP `api_secret` and service-account key are server-only secrets (never exposed in logs, URLs, tests, or docs).
- **Non-blocking** — every GA4 send never breaks the host flow (ingest ack / login).

### 4. Accuracy guardrails as up-front work instructions (get it right the first time)

Common failures that break accuracy (doc mapping table mismatching the code, double counting, secret exposure, missing irreversible console settings, etc.) are nailed down as **rules G1–G10**. In particular, GA4 console's **irreversible settings** (no BigQuery link backfill, data retention, custom dimensions, traffic filters) are highlighted as a callout at the very front of the guide, to be handled **before traffic begins**.

---

## Why a skill is needed

| Common attempt | Expectation | Reality |
|---|---|---|
| Start GA4 integration without scanning | Quick connection | Built on the wrong path/event name → rework |
| Write the mapping table from memory/guesswork | Doc completed | Code says `login_succeeded` but the doc says `login` → DebugView verification breaks |
| Send the same event from both client + server | Capture everything | GA4 numbers inflated (double counting) |
| Let traffic flow first, configure the console later | Configure leisurely | No BigQuery backfill, retention expired → past data permanently lost |
| Auto-run live sends | Verify right away | Real property polluted, credentials leaked |

This skill pins to facts via code scanning, derives the mapping from the code, front-loads the irreversible settings, and keeps dry-run as the default to head off the above pitfalls in advance.

---

## Quick start

### Prerequisites

- Cursor, Claude Code, or Codex
- An app codebase that **already has event logging and an analytics dashboard** to be integrated
- (Optional) the [`aztks-agent`](https://github.com/wild-mental/aztks-skill) subagent for the Phase D evaluation loop — if absent, it is replaced by an equivalent self AZTKS check

### Installing the skill

Install the skill by choosing either **personal** or **project** scope. Both scopes fetch only `SKILL.md` via `curl`. **Do not clone this entire repository into your working repo.**

| Tool | Personal path | Project path |
|------|-----------|---------------|
| Cursor | `~/.cursor/skills/ga4-vibe/` | `.cursor/skills/ga4-vibe/` |
| Claude Code | `~/.claude/skills/ga4-vibe/` | `.claude/skills/ga4-vibe/` |
| Codex | `~/.agents/skills/ga4-vibe/` | `.agents/skills/ga4-vibe/` |

#### Personal skill (recommended — leaves the Git repo unchanged)

```bash
# Cursor
mkdir -p ~/.cursor/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md \
  -o ~/.cursor/skills/ga4-vibe/SKILL.md

# Claude Code
mkdir -p ~/.claude/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md \
  -o ~/.claude/skills/ga4-vibe/SKILL.md

# Codex
mkdir -p ~/.agents/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md \
  -o ~/.agents/skills/ga4-vibe/SKILL.md
```

#### Project skill (install into the project path)

```bash
# Cursor
mkdir -p .cursor/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md \
  -o .cursor/skills/ga4-vibe/SKILL.md

# Claude Code
mkdir -p .claude/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md \
  -o .claude/skills/ga4-vibe/SKILL.md

# Codex
mkdir -p .agents/skills/ga4-vibe
curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md \
  -o .agents/skills/ga4-vibe/SKILL.md
```

#### After installing

- **Cursor**: **Reload Window** once
- **Claude Code**: skill edits reflect live during a session; a new top-level `.claude/skills/` added after the session starts may require a restart
- **Codex**: if the skill does not appear, restart Codex

### How to use

Invoke it from the app project you want to integrate, and the Agent starts with the Phase A scan.

| Tool | Manual invocation |
|------|-----------|
| Cursor | `/ga4-vibe` |
| Claude Code | `/ga4-vibe` |
| Codex | `/skills` or `$ga4-vibe` |

**Example requests this applies to:**

- "Connect our log/dashboard system to GA4"
- "Make a script that sends dummy events (a synthetic funnel) to GA"
- "Write a GA4 setup & dashboard guide an operator can follow"
- "Connect GA4→BigQuery and make EDA/retention SQL"

> **Phase D evaluation loop:** After each track is produced, get a GO/NO-GO from `aztks-agent` (MODE: EVALUATE) and remediate & re-evaluate. If `aztks-agent` ([aztks-skill](https://github.com/wild-mental/aztks-skill)) is installed, it uses that; otherwise it performs an equivalent self AZTKS 5-dimension check.

---

## Skill structure

```
.cursor/skills/ga4-vibe/SKILL.md   # for Cursor
.claude/skills/ga4-vibe/SKILL.md   # for Claude Code
.agents/skills/ga4-vibe/SKILL.md   # for Codex
```

| Section (SKILL.md) | Contents |
|------|------|
| Phase A | Automatic context scan (10 variables) + sufficiency gate + material requests / multiple-choice decision questions |
| Phase B | Locked defaults (path separation, env no-op, secret separation, non-blocking, minimal dependencies, schema invariance) |
| Phase C | 4-track implementation (T-APP critical path; independent tracks parallel with disjoint write scope) |
| Accuracy guardrails | G1–G10 + §4.1 irreversible console settings (handle before traffic) |
| Phase D | aztks-agent EVALUATE → NO-GO remediation & re-evaluation (self-check if not installed) |
| Phase E | Exit gates (typecheck/test/lint/build exit 0), artifact existence, draft PR |
| Representative deliverable explanations | Friendly what/why/how guidance |
| Anti-Pattern Block | Blocks skipping the scan, guessed mapping, dual sends, secret exposure, live auto-run |

---

## Recommended for

- Teams that want to layer an app with its own logging/dashboard onto a **standard analytics platform (GA4/BigQuery)**
- Those who have experienced **DebugView verification breaking** because GA4 event names/parameters diverged from the code
- Those who want to **validate & demo the analytics environment with synthetic data** before real traffic
- Those who want to avoid **irreversible-setting pitfalls** like no BigQuery backfill

---

## References

- GA4 Measurement Protocol, BigQuery Export, data retention, custom dimensions, data filters — for official docs, follow the per-step URL citations in SKILL.md
- Companion evaluation: [aztks-skill](https://github.com/wild-mental/aztks-skill) (`aztks-agent` MODE: EVALUATE)
- Cursor Agent Skills: [Creating Skills](https://docs.cursor.com)
- Claude Code Skills: [Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- Codex Agent Skills: [Agent Skills](https://developers.openai.com/codex/skills/)

---

## For LLMs

<!-- install-only; operational rules live in SKILL.md — load skill after install -->

```
skill=ga4-vibe
repo=https://github.com/wild-mental/ga4-vibe-skill
install.method=curl-only  # never clone this repo into user's project

flow:
  1. ask scope: "개인 스킬(~/, 모든 프로젝트) vs 프로젝트 스킬(./, 현재 repo만) 중 어디에 설치할까요?"
  2. ask tools: "Cursor / Claude Code / Codex 중 어떤 도구에 설치할까요?" (복수 선택 가능)
  3. run matching curl commands below; do not install both scopes unless user asks
  4. post_install steps; then load SKILL.md — do not infer rules from README

scope.user.paths:
  cursor=~/.cursor/skills/ga4-vibe/SKILL.md
  claude=~/.claude/skills/ga4-vibe/SKILL.md
  codex=~/.agents/skills/ga4-vibe/SKILL.md

scope.project.paths:
  cursor=.cursor/skills/ga4-vibe/SKILL.md
  claude=.claude/skills/ga4-vibe/SKILL.md
  codex=.agents/skills/ga4-vibe/SKILL.md

install.user.cursor=mkdir -p ~/.cursor/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md -o ~/.cursor/skills/ga4-vibe/SKILL.md
install.user.claude=mkdir -p ~/.claude/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md -o ~/.claude/skills/ga4-vibe/SKILL.md
install.user.codex=mkdir -p ~/.agents/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md -o ~/.agents/skills/ga4-vibe/SKILL.md

install.project.cursor=mkdir -p .cursor/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.cursor/skills/ga4-vibe/SKILL.md -o .cursor/skills/ga4-vibe/SKILL.md
install.project.claude=mkdir -p .claude/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.claude/skills/ga4-vibe/SKILL.md -o .claude/skills/ga4-vibe/SKILL.md
install.project.codex=mkdir -p .agents/skills/ga4-vibe && curl -fsSL https://raw.githubusercontent.com/wild-mental/ga4-vibe-skill/main/.agents/skills/ga4-vibe/SKILL.md -o .agents/skills/ga4-vibe/SKILL.md

invoke.cursor=/ga4-vibe
invoke.claude=/ga4-vibe
invoke.codex=/skills|$ga4-vibe

post_install.cursor=Reload Window
post_install.claude=live reload; restart if new top-level .claude/skills/ after session start
post_install.codex=restart if skill not detected

optional_companion=aztks-agent (from wild-mental/aztks-skill) for the Phase D EVALUATE loop; degrades to self-review if absent

contract:
  deliverables=[T-APP app integration code, T1 deterministic dummy traffic generator, T2 GA4 setup guide, T3 GA4->BigQuery analysis kit]
  phases=[A scan+sufficiency-gate (no guessing; request materials or AskUserQuestion), B lock decisions, C implement (T-APP critical path; independent tracks parallel/disjoint write scope), D aztks-agent EVALUATE loop, E exit gates + delivery]
  no_guessing=scan code first, pin to facts, ask when insufficient; derive docs/tables from the mapper code (not memory)
  path_split=client gtag=auto page_view; server Measurement Protocol=authenticated behavioral events; one event = exactly one path (no dual count)
  env_gating=server send active only when measurement-id AND server secret both set; both unset=full no-op; one-side-only=throw
  secrets=measurement id (G-XXXX, NEXT_PUBLIC_*) is public; MP api_secret + service-account key are server-only secrets (never logged/printed/tested/documented)
  non_blocking=all GA4 sends never break the host flow (ingest ack / login); failures = one error log line
  accuracy_guardrails=G1..G10 (mapping table derived from mapper code; exact event names/params/transport; no dual count; env getter unit-tested; measure base gate first; secret non-exposure; UI steps verified by official URLs; dry-run default; irreversible console settings before traffic)
  irreversible_before_traffic=[BigQuery link (no backfill), data retention (raise to 14mo), custom dimensions (no retro), data filters (internal/dev traffic)]
  exit_gate=typecheck && test && lint && build all exit 0; artifact existence check; draft PR (no main merge / no prod auto-deploy by default)
```

---

## License

[MIT License](LICENSE)
