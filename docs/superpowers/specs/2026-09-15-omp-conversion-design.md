# ios-simulator-skill → OMP Conversion Design

Date: 2026-09-15
Version target: 2.0.0 (breaking — runtime target changes from Claude Code to Oh My Pi)

## Goal

Convert the ios-simulator-skill repository in place from a Claude Code plugin to an
OMP-native agent skill. All 29 scripts remain byte-identical (they are plain Python,
invoked via `bash`); the conversion is documentation + orchestration guidance only.

## Decisions (user-approved)

1. **Convert repo in place** — Claude Code plugin files removed.
2. **Docs + orchestration layer only** — no script changes, no new glue modules.
3. **OMP-only variant** — no Claude Code compatibility retained in skill docs.

## File changes

| File | Change |
|---|---|
| `ios-simulator-skill/skills/ios-simulator-skill/SKILL.md` | Rewritten for OMP: agentskills-spec frontmatter (`name`, `description` starting "Use when…", no workflow summary), OMP Orchestration section (see below), OMP install path `~/.omp/agent/skills/` |
| `README.md` | Install section → `~/.omp/agent/skills/ios-simulator-skill/`; plugin-marketplace block removed; OMP-ability feature notes; evals section rewritten (Claude Code evals no longer apply — replaced with subagent retrieval testing) |
| `CLAUDE.md` | Install/loading references updated to OMP; retains its role as developer/architecture guide (OMP auto-loads it as repo context) |
| `DEV.md` | `~/.claude/skills/` → `~/.omp/agent/skills/`; "Claude Code skill" phrasing → OMP |
| `.claude-plugin/` (root) | Deleted (marketplace.json) |
| `ios-simulator-skill/.claude-plugin/` | Deleted (plugin.json) |
| `.github/workflows/release.yml` | Remove plugin.json existence check (line 28); zip contents unchanged (still the skill dir root — same unzip layout, new destination); step-summary install hint → `~/.omp/agent/skills/` |
| `site/index.html` | Claude Code branding/paths → OMP; `/plugin marketplace` block → manual install |
| `pyproject.toml`, SKILL.md frontmatter | Version 1.5.0 → 2.0.0 (keeps `validate-version.yml` green: it greps the number in SKILL.md) |
| `scripts/xcode/config.py` | **Unchanged** (scripts byte-identical per scope). Wart noted: config path is `.claude/skills/<name>/config.json`; still functional under OMP since the dir name is auto-detected from install location. Revisit if a v2.1 touches scripts. |

Untouched: all 29 scripts, `common/`, `xcode/` modules, `tests/`, `lint.yml`, `pages.yml`,
`validate-version.yml`, `.gitignore` (`/.claude` entry is generic ignore hygiene, kept).

## SKILL.md structure

1. **Frontmatter** — `name: ios-simulator-skill`; `description: Use when…` with concrete
   triggers (iOS simulator tasks, Xcode builds, semantic UI navigation, hang detection,
   accessibility audits) and keywords (simctl, idb, xcodebuild, hang, xcresult).
2. **Quick Start** — unchanged commands (health check, launch, map, tap, type).
3. **Navigation Strategy** — accessibility-tree-first, screenshots last. Now two screenshot
   consumers: `inspect_image` (OMP vision, preferred) and inline (fallback).
4. **29 Production Scripts** — reference kept as-is (it is the core value; matches SKILL.md
   conventions of a reference skill).
5. **OMP Orchestration (new)** — five subsections:
   - **Screenshots → `inspect_image`**: capture to file, then ask a targeted question.
     ~0 image tokens in conversation vs 800–6,300 inline. Sizing presets remain capture knobs.
   - **Stateful sessions → `eval` kernel**: import script modules once, keep UDID and
     hang-session IDs as kernel state across steps. Worked example: boot → launch → navigate.
   - **Streaming → `hub op:start`**: `log_monitor.py --follow` and `hang_watcher.py --watch`
     are long-running; run supervised via hub with ready pattern, follow logs with cursors.
     HangBuster session mode (`--start`/`--stop`) remains the recommended path (already
     OMP-shaped); hub applies only to legacy live-stream modes.
   - **Parallel prep → `task` subagents**: one wave for independent boot + health check +
     install; never serialized padding.
   - **Multi-step flows → `todo`**: native phase tracking for UI test workflows.
6. **Native-tool substitution table** — `.xcresult` via `read` (directory + members),
   Core Data via `read db.sqlite:table`, session NDJSON via `grep`/`read`; `zcat | jq`
   recipes retained as shell alternative.
7. **Common Patterns / Typical Workflow / Configuration / Requirements / Troubleshooting** —
   kept, with "let Claude Code invoke them" tail line replaced by OMP discovery semantics
   (`skill://ios-simulator-skill`).

## Error handling

No runtime changes. Failure modes of this conversion are documentation-level: bad
frontmatter (skill won't surface), missing OMP guidance (agent falls back to inline
screenshots / blocking streams). Both are caught by the retrieval test below and by
`pytest`/`ruff`/`black` gates.

## Testing (writing-skills TDD)

- **RED (baseline)**: read-only subagent given a realistic simulator recon task with raw
  OMP tools only. Document: inline screenshot token bombs, blocking `log stream`,
  re-derived UDID, tool-policy violations. Running concurrently (agent `RedBaseline`).
- **GREEN**: same scenario shape re-run with converted SKILL.md available; agent should
  reach for `inspect_image`, keep state in `eval`, avoid blocking streams.
- Repo gates: `pytest tests/` (184 tests), `ruff check .`, `black --check .` — must pass
  unchanged (proves scripts untouched).

## Out of scope

- Any script CLI/env-var/session-storage change.
- MCP or `xd://` wrappers around scripts.
- Keeping Claude Code install docs.
- Renaming `config.json` storage path (scripts byte-identical).
