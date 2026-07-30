# Claude Code: Token Optimization & Context Management

[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

A practical guide to keeping Claude Code sessions cheap: what actually occupies the context window, which layers cut input, output, and cross-session cost, and how to prove each one pays for itself before you keep it.

> *If I were paying for every token a coding agent burns, what would I install first — and how would I know it worked?*

The tools split into four layers, each with its own token pool:

- **Built-in practices** — LSP navigation, dirty-context subagents, disciplined `/compact` and `/clear`, and pruning MCP/plugin surface. No installs, biggest wins.
- **Input-side tooling** — RTK (compressed shell output) plus one code-graph or cross-session memory layer (codebase-memory-mcp / claude-mem).
- **Output-side plugins** — Caveman (terse prose) and Ponytail (minimal code), acting on disjoint parts of the reply.
- **Docs on demand** — Context7, to stop the model hallucinating stale APIs.

Then installation for Linux/macOS and Windows, and a measurement layer (`/context`, `/cost`, `rtk gain`, Session Report) so you enable one thing at a time and keep only what earns its place.

> **Trust & scope note.** Except the official Anthropic plugins (the LSP connectors and Session Report), everything below is community-built (RTK, codebase-memory-mcp, claude-mem, Caveman, Ponytail, Context7, Claude HUD) — not Anthropic products. Review any `curl | sh` install script before running it. claude-mem sends observation content to a cloud LLM provider — check the company data policy before enabling it on proprietary code.

---

## Table of Contents

- [Part I — Built-in practices](#part-i--built-in-practices)
  - [1. LSP plugin — the first thing to add](#1-lsp-plugin--the-first-thing-to-add)
  - [2. Subagents as "dirty context" isolators](#2-subagents-as-dirty-context-isolators)
  - [3. /compact, /clear, and short tasks](#3-compact-clear-and-short-tasks)
  - [4. Disable unused MCP servers & plugins, keep `CLAUDE.md` lean](#4-disable-unused-mcp-servers--plugins-keep-claudemd-lean)
- [Part II — Input-side tooling](#part-ii--input-side-tooling)
  - [5. RTK + codebase-memory-mcp vs RTK + claude-mem](#5-rtk--codebase-memory-mcp-vs-rtk--claude-mem)
  - [6. What to choose](#6-what-to-choose)
- [Part III — Output-side plugins](#part-iii--output-side-plugins)
  - [7. Caveman — terse replies](#7-caveman--terse-replies)
  - [8. Ponytail — least code that works](#8-ponytail--least-code-that-works)
- [Part IV — Docs on demand](#part-iv--docs-on-demand)
  - [9. Context7](#9-context7)
- [Part V — Installation](#part-v--installation)
  - [Linux/macOS (bash)](#linuxmacos-bash)
  - [Windows 10+](#windows-10)
- [Part VI — Measuring savings](#part-vi--measuring-savings)
- [Summary](#summary)
- [Related](#related)
- [License](#license)

## Part I — Built-in practices

### 1. LSP plugin — the first thing to add

Claude Code has built-in LSP support via official code-intelligence plugins (plugins reference). Instead of chains like `grep → read → grep → read`, the agent runs structural queries:

```
find definition UserService.createUser
find references UserRepository.save
list symbols in user-service.ts
find implementations AuthProvider
```

**Why it matters:** unnecessary text never enters the context at all — better than any compression. After edits, LSP also returns type errors automatically, so the agent rarely needs a full build/compile.

**Install — two parts, both required.** The plugin is only a connector: it teaches Claude Code how to talk to a language server, but doesn't ship one. The binary is the actual server — without it the plugin has nothing to launch:

```bash
npm install -g typescript-language-server pyright   # 1. the language servers themselves
/plugin install typescript-lsp@claude-plugins-official   # 2. TS/JS connector (inside a session; terminal form: claude plugin install …)
/plugin install pyright-lsp@claude-plugins-official
```

Other languages: check the Discover tab in `/plugin` (typescript-lsp, pyright-lsp).

**Official LSP is usually enough if you have:** a typed language (TypeScript, Python, Rust, Go, Java), a small/medium monorepo, most cost coming from definition/reference lookups, and no need for complex architecture-wide graph queries.

### 2. Subagents as "dirty context" isolators

For tasks that require reading many files, use a dedicated exploration subagent (subagents docs):

```
Use a subagent to investigate authentication flow.
Return only: relevant files, relevant symbols, call chain,
likely change points, unresolved questions.
Do not return complete file contents.
```

The subagent gets its own context window; only the short summary returns to the main conversation — dozens of file reads never pollute the main context.

⚠️ **Don't use subagents for every small operation:** each one has its own startup context, tools, and instructions. They save *main* context, but not necessarily *total* tokens.

### 3. /compact, /clear, and short tasks

RTK shrinks new tool outputs but doesn't remove already-accumulated junk. Compact with explicit instructions:

```
/compact Preserve: current goal, changed files, confirmed architecture,
failing tests, unresolved issues.
Drop: successful command output, superseded plans,
exploratory dead ends, repeated file contents.
```

- Compaction instructions can also live in `CLAUDE.md`.
- For unrelated tasks use `/clear` — stale context is paid for and reprocessed with every message.
- After several failed approaches, a **fresh session with a good short brief** beats compressing old junk again (see managing costs).

### 4. Disable unused MCP servers & plugins, keep `CLAUDE.md` lean

Every MCP server can add tool names, descriptions, schemas, instructions, tool-selection mistakes, and extra tool calls. Tool definitions are deferred by default in Claude Code, but disabling unused MCP servers still pays off — and **prefer a CLI where a good one exists**: `gh`, `aws` are often cheaper in context than the equivalent MCP (managing costs).

**Plugins are the same tax in a different wrapper.** An enabled plugin can ship skills (each adds a line to the skills listing in every session), custom agents, hooks (some inject extra text into every single prompt), and bundled MCP servers. A dozen enabled plugins easily cost thousands of tokens before you type anything — keep enabled only what the current work needs.

**How to see what actually occupies the context** — run `/context` inside a session; it prints a per-category breakdown of the default (pre-conversation) load:

```
> /context
⛁ System prompt: 3.3k tokens (1.7%)
⛁ System tools: 14.2k tokens (7.1%)
⛁ MCP tools: 48.7k tokens (24.4%)   ← usually the biggest line
⛁ Custom agents: 2.6k tokens (1.3%)  ← plugin/user agents
⛁ Memory files: 1.9k tokens (1.0%)   ← CLAUDE.md + rules
⛁ Messages: 12.4k tokens (6.2%)
⛁ Free space: 117k (58.3%)
```

(Illustrative numbers from a session with ~10 MCP servers and several plugins connected; yours will differ — run it in your own session.) Then act on it:

```
/mcp        # list MCP servers, toggle/disable per-session
/plugin     # Installed tab — disable plugins you don't need right now
/context    # re-run to confirm the drop
```

Same class of problem: `CLAUDE.md`, hooks, and rules text ride along with **every** message (the "Memory files" line above). Keep `CLAUDE.md` short; move rarely-needed guidance into skills or docs loaded on demand.

**Browser MCP — pick one, toggle per task.** For frontend work add exactly one browser server, not both. A browser `snapshot` can return tens of thousands of tokens in a single tool result, and on a backend task an idle browser MCP is pure tool-surface tax.

| Server | Best for |
| --- | --- |
| **Playwright** | functional testing, forms, navigation, screenshots, responsive states, regression checks, accessibility trees |
| **Chrome DevTools MCP** | network, console errors, performance, source maps, runtime profiling, layout/rendering analysis |

Route by task: frontend feature → **Playwright**; performance/network investigation → **Chrome DevTools**; backend or refactor → **both off**. Toggle per session in `/mcp`.

## Part II — Input-side tooling

### 5. RTK + codebase-memory-mcp vs RTK + claude-mem

These combos solve different problems:

- **RTK + codebase-memory-mcp** = fewer tokens on shell output and code exploration
- **RTK + claude-mem** = less re-explaining past work between sessions

| Layer | RTK + codebase-memory-mcp | RTK + claude-mem |
| --- | --- | --- |
| Shell, tests, Git, logs | RTK | RTK |
| Finding functions & dependencies | Very strong | Barely addresses |
| Reducing Grep → Read → Read | Very strong | Limited |
| Memory of past decisions | Limited | Very strong |
| Resuming yesterday's task | Rebuild from Git/files | Auto-loads history |
| Short one-off tasks | Usually cheaper | May add overhead |
| Large monorepo | Better | Not sufficient alone |
| Codex + Claude Code | Both well supported | Supported, deeper Claude Code integration |
| Extra LLM calls | No | Yes (observations & summaries) |

**Why codebase-memory-mcp wins on tokens**

It builds a persistent knowledge graph of the repo (functions, classes, calls, dependencies, routes, cross-service links). Instead of dozens of greps and full-file reads: one structural query like *"who calls ProcessOrder?"*. No LLM inside; indexing runs locally, no API key (repo).

It perfectly complements RTK: RTK shrinks the output of commands that already ran; **codebase-memory-mcp** prevents unnecessary grep/read commands from running at all.

**Benchmark:**

- **[study]** 31 real-world repos (arXiv:2603.27277): **~10× fewer tokens**, **2.1× fewer tool calls** than file exploration. Trade-off: **83% answer quality vs 92%** for the file-exploration agent — graph-first is cheaper, but the agent sometimes still needs to check source code.

**Git worktrees: every worktree is its own project**

The index is keyed by repository path. A new worktree is a new path, so it starts as a separate project with an **empty index** — the agent runs a structural query, gets nothing back, falls back to grep/read, and stops calling the MCP. If your workflow creates a worktree per ticket and Claude "never uses" codebase-memory-mcp, this is why. Expected behavior, not a config bug: worktrees of one repo don't share an index.

What makes it work anyway:

- `auto_index true` (already set in the install below) — each new worktree indexes automatically on the first MCP connection. Indexing is fast, so a worktree per ticket is an acceptable cost.
- Check `auto_index_limit`: a repo above the file limit is **silently skipped** — the same "agent never uses it" symptom with a different cause.
- No manual re-indexing per session: the index is SQLite in `~/.cache/codebase-memory-mcp` (override via `CBM_CACHE_DIR`), persists across restarts, and `auto_watch` (default on) picks up changes via git. A manual "Index this project" is only needed when `auto_index` is off, the repo exceeds `auto_index_limit`, or the index looks stale.
- Diagnostics: the `list_projects` MCP tool shows which paths are actually indexed.

⚠️ Worktrees nested *inside* the main repo and excluded only via `.git/info/exclude` are still walked by the indexer — up to OOM on large trees ([#489](https://github.com/DeusData/codebase-memory-mcp/issues/489)). Exclude them in a committed `.gitignore` or `.cbmignore` instead.

**One code graph is enough — don't stack navigation MCPs.** codebase-memory-mcp, [Serena](https://github.com/oraios/serena), [CodeGraphContext](https://github.com/CodeGraphContext/CodeGraphContext), and [code-review-graph](https://github.com/tirth8205/code-review-graph) all answer the same "who calls / depends on / is affected by X" from a code graph. Running two at once is a net loss: overlapping tools with near-identical descriptions degrade the model's tool selection — it picks the wrong one or calls both — and every extra server piles on tool-surface tax (§4) for a capability you already have.

| Tool | Graph source | Extra infra | Niche it adds |
| --- | --- | --- | --- |
| **codebase-memory-mcp** (this stack) | local persistent index (SQLite) | none — no DB, no API key, no LLM | general navigation: callers, deps, impact; auto-index + git watch |
| **Serena** | live LSP (language servers) | the language servers | duplicates the official LSP plugins already here (§1); adds semantic *editing* |
| **CodeGraphContext** | graph database | a graph DB backend (FalkorDB/Neo4j, usually Docker) + 20+ tools | Cypher queries, dead-code, complexity metrics |
| **code-review-graph** | Tree-sitter AST | none (pip) | narrower — computes the minimal file set to read *at review time* |

**Why codebase-memory-mcp is the default pick here:** it's the only one giving repo-wide navigation with zero extra infrastructure — no graph database to run, no API key, no LLM calls. Serena overlaps the LSP plugins this doc already installs; CodeGraphContext buys Cypher/dead-code power at the cost of a DB backend and a 20-tool surface; code-review-graph is review-scoped, not general navigation. A pure diff-review workflow → code-review-graph can *replace* codebase-memory-mcp, but never run both.

**Why claude-mem may NOT reduce tokens**

Claude-mem stores tool usage, generates AI observations/summaries, and injects them into future sessions (repo). Useful for continuity, but it doesn't replace structural navigation of current code. Defaults are heavy (configuration docs): 50 observations, 10 recent sessions, 5 expanded observations; even the progressive-disclosure index costs on the order of 1k+ tokens per session start (grows with observation count — measure yours).

**Short-task example:** fix one CSS bug → claude-mem loads unrelated backend history, creates new observations, generates a summary → **more tokens, not fewer**. On long projects the effect flips: no re-reading old discussions, no repeated investigations.

⚠️ **Extra costs:** claude-mem uses Claude Agent SDK / Gemini / OpenRouter for compression — extra LLM operations codebase-memory-mcp doesn't have. Observation content is sent to the configured AI provider, so semantic compression isn't fully local when using a cloud model.

### 6. What to choose

**RTK + codebase-memory-mcp if:** the main cost is Read/Grep/Glob; large repo or monorepo; agent keeps re-exploring call chains; you use both Claude Code and Codex; the goal is total token reduction.

**RTK + claude-mem if:** you resume tasks days later; past architectural decisions matter; the agent repeats past investigations; you open/close sessions often; the repo is small and navigation isn't the main cost.

**Best practical setup — all three**

They barely conflict, so combine them with a clear split of responsibilities:

| Tool | Owns |
| --- | --- |
| RTK | tests, build, Git, logs, shell |
| codebase-memory-mcp | current code structure, callers, dependencies, impact analysis |
| claude-mem (more conservatively than defaults) | why a decision was made, what was tried, known gotchas |

`rtk gain` on this machine shows 55.9% average across commands (small sample); vendor claims 60–90% on common dev commands (repo). Savings depend entirely on how noisy your commands are — clean `git status` saves almost nothing.

**RTK filters can lie — scope the hook to proven commands**

A filter can fabricate a clean summary over real errors, and a wrong summary sends the agent into retry loops that cost more than the filter saved. Documented cases: the generic lint filter printed "Lint: No issues found" over 4 real biome errors (local incident); `rtk grep` truncates lines to 80 chars and drops the matching part ([#1161](https://github.com/rtk-ai/rtk/issues/1161)); Playwright output collapses from 100K+ tokens to ~9 ([#1125](https://github.com/rtk-ai/rtk/issues/1125), [#690](https://github.com/rtk-ai/rtk/issues/690)); `rtk ls` returns a literal `"(empty)"` for any input ([#1418](https://github.com/rtk-ai/rtk/issues/1418)); piped/redirected output gets silently corrupted ([#1282](https://github.com/rtk-ai/rtk/issues/1282)); on Windows the Claude Code hook intermittently returns output that doesn't match the real command ([#2176](https://github.com/rtk-ai/rtk/issues/2176)).

The proven core is what RTK was built for: `git status`/`diff`/`log`, test runners, builds. RTK has no allowlist mode — the only scoping mechanism is `[hooks].exclude_commands` in its config, so the install blocks below seed it with the known-broken filters right after `rtk init`. Working rules on top of that:

- **Exit code beats summary.** Non-zero exit with clean-looking filtered output = the filter lied; re-run raw via `rtk proxy <cmd>` before concluding anything.
- **Dry-run new tools** with `rtk hook check "<command>"`; no dedicated filter → add to `exclude_commands`.
- Config wiring has broken before ([#1335](https://github.com/rtk-ai/rtk/issues/1335), [#243](https://github.com/rtk-ai/rtk/issues/243)) — after editing the config, verify an excluded command actually passes through unchanged.

## Part III — Output-side plugins

### 7. Caveman — terse replies

Caveman makes the agent answer tersely: drops filler, articles, and pleasantries while keeping all technical substance; code blocks stay untouched. Cuts **output** tokens only — input/cache usage is unchanged. Disable for documentation, user-facing texts, or nuanced explanations (`stop caveman`).

```
/plugin marketplace add JuliusBrussee/caveman
/plugin install caveman@caveman
```

**Style only, nothing else.** Caveman rewrites the agent's prose — explanations, openers, repetition. The model, its reasoning, code, commands, and exact error text stay untouched.

**Intensity levels** (switch with `/caveman <level>`):

| Level | What it does | When to use |
| --- | --- | --- |
| `lite` | Cuts filler and hedging, keeps normal sentences | Best default for daily work |
| `full` | Drops articles and connectives, allows short fragments | Maximum reasonable savings, still readable |
| `ultra` | Minimal phrases, each fact stated once | Trivial fixes in a well-known project |
| `wenyan-lite` | Compressed reply in Classical Chinese | Almost never needed |
| `wenyan-full` | Even shorter Classical Chinese | Only for maximum compression |
| `wenyan-ultra` | Extreme Classical Chinese compression | Unusable for normal development |

`/caveman wenyan` is a shortcut for `wenyan-full`.

The same answer at the first three levels:

```
normal: Your component re-renders because you create a new object
        reference on every render. Wrap the object in useMemo.
lite:   Your component re-renders because you create a new object
        reference each render. Wrap it in useMemo.
full:   New object ref each render. Wrap object in useMemo.
ultra:  New ref/render. useMemo it.
```

### 8. Ponytail — least code that works

Ponytail biases the agent toward the minimal working change ("the best code is the code you never wrote"): fewer generated lines → fewer output tokens and less code to review. Disable when exhaustiveness is the point (test suites, migrations, security-sensitive code).

```
/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```

**Code only, not prose.** Ponytail changes what gets built — smaller diffs, stdlib over dependencies, YAGNI. Reply style is Caveman's pool. Deliberate shortcuts get a `ponytail:` comment naming the ceiling; `/ponytail-debt` collects them into a ledger so "later" doesn't mean "never".

**Intensity levels** (switch with `/ponytail <level>`; no argument reports the current one; off: `/ponytail off` or "stop ponytail"):

| Level | What it does | When to use |
| --- | --- | --- |
| `lite` | Builds what's asked, names the lazier alternative in one line | You want to make the call yourself |
| `full` | The ladder enforced: stdlib/native first, shortest working diff | Best default for daily work |
| `ultra` | YAGNI extremist: deletion before addition, challenges the requirement itself | Codebase drowning in over-engineering |

The same task ("Add a cache for these API responses") at each level:

```
lite:  Done, cache added. FYI: functools.lru_cache covers this in one
       line if you'd rather not own a cache class.
full:  @lru_cache(maxsize=1000) on the fetch function. Skipped custom
       cache class, add when lru_cache measurably falls short.
ultra: No cache until a profiler says so. When it does: @lru_cache. A
       hand-rolled TTL cache class is a bug farm with a hit rate.
```

Two settings worth knowing:

- **Default level for new sessions:** `PONYTAIL_DEFAULT_MODE` env var (`lite`/`full`/`ultra`/`off`) or `defaultMode` in `~/.config/ponytail/config.json` (`%APPDATA%\ponytail\config.json` on Windows). Nothing set = `full`.
- **Subagent injection — a token cost worth scoping.** While active, the ruleset is injected into every subagent spawned via the Agent tool. `PONYTAIL_SUBAGENT_MATCHER` limits that: a regex tested against the subagent's `agent_type` (unanchored, case-insensitive; plugin agents look like `plugin:name`). Read-only search agents don't build code — paying the ruleset there is waste. Unset or invalid regex = inject into all.

**Benchmark:**

- **[vendor]** the plugin's own medians (5 everyday tasks × 3 models, see `/ponytail-gain`): 80–94% less code, 47–77% cheaper, 3–6× faster.

**Caveman + Ponytail together**

They stack cleanly — different token pools: Caveman compresses the agent's prose and never touches code blocks; Ponytail shrinks the generated code and doesn't touch prose style. No known conflicts, but savings are **not additive**: Caveman's ~65% applies to prose around code Ponytail already made shorter. Nuance: both bias toward brevity, so on tasks where detail is the deliverable (design docs, onboarding guides, exhaustive test suites) double-brevity starts dropping useful content.

- **Ponytail only** — code-heavy feature work where you want full explanations but minimal diffs.
- **Caveman only** — investigation, review, Q&A sessions with little generated code.
- **Both** — long agentic coding sessions where cost matters most.
- **Neither** — documentation, user-facing texts, security-sensitive code.

**Combined [measured]:** with both enabled in one session, generated code shrank −66…−73% (Ponytail A/B above) while prose replies kept the ~65% Caveman cut. No interference observed — they act on disjoint parts of the output.

## Part IV — Docs on demand

### 9. Context7

Context7 fetches current, version-specific library documentation on demand instead of the model guessing from stale training data. It saves tokens **indirectly**: fewer hallucinated APIs → fewer failed attempts and retry loops. Each docs fetch itself costs tokens, so it pays off on unfamiliar or fast-moving libraries, not on every question. Pre-installed only on Coder workspaces (check `/plugins`); on a local machine install it — see Installation below.

**Reach for it when the task touches** a new or updated library version, an external API, framework documentation, a migration guide, or an unfamiliar SDK. **Skip it for** CSS tweaks, local refactors, or business-logic changes on code you already know — a docs fetch there is pure overhead.

**Benchmark:** no meaningful direct number — the win is avoided retry loops, not compressed output.

## Part V — Installation

### Linux/macOS (bash)

```bash
# RTK (https://github.com/rtk-ai/rtk)
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
rtk init -g
rtk init --show

# Exclude known-broken filters from the RTK hook (see "RTK filters can lie").
# Fresh install has no [hooks] section; if `rtk config` already shows one,
# edit the file by hand instead of appending a duplicate section.
RTK_CFG="$HOME/.config/rtk/config.toml"   # macOS: "$HOME/Library/Application Support/rtk/config.toml"
mkdir -p "$(dirname "$RTK_CFG")"
cat >> "$RTK_CFG" <<'EOF'

[hooks]
exclude_commands = [
  "grep",          # 80-char truncation loses matches (rtk#1161)
  "ls",            # returns literal "(empty)" (rtk#1418)
  "curl",          # API/HTTP responses must arrive intact
  "playwright",    # E2E output crushed to ~9 tokens (rtk#1125, rtk#690)
  "pnpm --filter", # rtk misparses the --filter flag (rtk#259)
  "biome",         # lint filter printed "No issues found" over real errors
  "npx biome",
]
EOF
rtk config                          # verify the [hooks] section is loaded
rtk hook check "grep foo bar.txt"   # must print the command UNCHANGED

# Codebase Memory (https://github.com/DeusData/codebase-memory-mcp)
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash
codebase-memory-mcp config set auto_index true
codebase-memory-mcp config set auto_index_limit 50000

# LSP binaries
npm install -g typescript-language-server pyright

# Official LSP plugins — connectors only, they need the binaries above
# (terminal form; inside a session use /plugin install …)
claude plugin marketplace update claude-plugins-official
claude plugin install typescript-lsp@claude-plugins-official
claude plugin install pyright-lsp@claude-plugins-official

# Claude-Mem (https://github.com/thedotmack/claude-mem)
npx claude-mem install

# Verify
rtk --version
rtk gain                          # must work; if it fails you likely have
                                  # the wrong rtk ("Rust Type Kit" name collision)
codebase-memory-mcp --version
typescript-language-server --version
pyright --version
npx claude-mem status

# Configure claude-mem far more conservatively than defaults:
# short index of the most important decisions, filtered observation
# types/concepts, no full records. (The model is already Haiku by
# default — pinned here explicitly.) Run AFTER `npx claude-mem install`.
SETTINGS="$HOME/.claude-mem/settings.json"
[ -f "$SETTINGS" ] || echo '{}' > "$SETTINGS"
TMP_FILE="$(mktemp)"
jq '. + {
  "CLAUDE_MEM_MODEL": "claude-haiku-4-5-20251001",
  "CLAUDE_MEM_CONTEXT_OBSERVATIONS": "10",
  "CLAUDE_MEM_CONTEXT_SESSION_COUNT": "3",
  "CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES": "bugfix,decision,discovery",
  "CLAUDE_MEM_CONTEXT_OBSERVATION_CONCEPTS": "why-it-exists,problem-solution,gotcha,trade-off",
  "CLAUDE_MEM_CONTEXT_FULL_COUNT": "0",
  "CLAUDE_MEM_CONTEXT_FULL_FIELD": "narrative",
  "CLAUDE_MEM_CONTEXT_SHOW_READ_TOKENS": "false",
  "CLAUDE_MEM_CONTEXT_SHOW_WORK_TOKENS": "false",
  "CLAUDE_MEM_CONTEXT_SHOW_SAVINGS_AMOUNT": "false",
  "CLAUDE_MEM_CONTEXT_SHOW_SAVINGS_PERCENT": "false",
  "CLAUDE_MEM_CONTEXT_SHOW_TERMINAL_OUTPUT": "false",
  "CLAUDE_MEM_CONTEXT_SHOW_LAST_SUMMARY": "false",
  "CLAUDE_MEM_CONTEXT_SHOW_LAST_MESSAGE": "false",
  "CLAUDE_MEM_FOLDER_CLAUDEMD_ENABLED": "false"
}' "$SETTINGS" > "$TMP_FILE" && mv "$TMP_FILE" "$SETTINGS"
npx claude-mem restart
npx claude-mem status

# Context7 (https://github.com/upstash/context7)
npx ctx7 setup --claude   # guided setup: auth + install
# or manually (API key optional — free at context7.com/dashboard, higher rate limits):
claude mcp add --transport http context7 https://mcp.context7.com/mcp --header "CONTEXT7_API_KEY: <key>"

# Caveman
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman

# Ponytail
claude plugin marketplace add DietrichGebert/ponytail
claude plugin install ponytail@ponytail

# Claude HUD (https://github.com/jarrodwatts/claude-hud) — live context/cost/git in the status line.
# Install LAST: /claude-hud:setup (run below) overwrites the status line, so it must come after
# Caveman — whose plugin sets its own — for Claude HUD's to win. Needs Claude Code v1.0.80+, Node 18+.
claude plugin marketplace add jarrodwatts/claude-hud
claude plugin install claude-hud

# Session Report (official; per-session token/cache/subagent/skill HTML report — see Part VI).
# Marketplace claude-plugins-official already added in the LSP step above.
# Known cosmetic bug: a missing manifest can list its skill twice under different namespaces.
claude plugin install session-report@claude-plugins-official

# Launch Claude Code and execute the following:
/reload-plugins
/doctor
/mcp
/plugin
/caveman lite
/claude-hud:setup   # guided status-line preset; run after /reload-plugins so it overrides
```

### Windows 10+

**Prerequisites:** PowerShell 7+ (`pwsh`), Node.js + npm on `PATH`, Claude Code CLI (`claude`). No `jq` needed — the claude-mem config is edited with native PowerShell JSON. **Do NOT use the projects' `install.sh` scripts on Windows.**

> ⚠️ **Why not `bash … install.sh`:** on Windows `bash` usually resolves to **WSL** (or Git Bash). Under WSL the installer detects `os: linux`, downloads a **Linux** binary, and drops it inside the WSL filesystem (`/home/<user>/.local/bin/`) — invisible to and unrunnable by Windows-native Claude Code. Both projects publish real Windows `.zip` binaries (`codebase-memory-mcp-windows-amd64.zip`, `rtk-x86_64-pc-windows-msvc.zip`), so download those directly with PowerShell — no bash, no WSL, no Git Bash. Windows-on-ARM: swap `amd64`/`x86_64` for the `arm64`/`aarch64` assets.

```powershell
# ── STEP 0: PATH + a helper that installs a GitHub Windows .zip binary ──
# Both binaries live in ~/.local/bin (= %USERPROFILE%\.local\bin). PowerShell
# won't see them until this dir is on PATH — session + persisted for the user.
$bin = "$env:USERPROFILE\.local\bin"
New-Item -ItemType Directory -Force -Path $bin | Out-Null
$env:PATH = "$bin;$env:PATH"
$userPath = [Environment]::GetEnvironmentVariable("PATH", "User")
if ($userPath -notlike "*$bin*") {
  [Environment]::SetEnvironmentVariable("PATH", "$bin;$userPath", "User")
}

function Install-WinZipBinary($url, $exeGlob, $finalName) {
  $zip = Join-Path $env:TEMP "wz-$finalName.zip"
  $dir = Join-Path $env:TEMP "wz-$finalName"
  Invoke-WebRequest -Uri $url -OutFile $zip
  Remove-Item $dir -Recurse -Force -ErrorAction SilentlyContinue
  Expand-Archive -Path $zip -DestinationPath $dir -Force
  $exe = Get-ChildItem $dir -Recurse -Filter $exeGlob | Select-Object -First 1
  if (-not $exe) { throw "no $exeGlob inside $url" }
  Copy-Item $exe.FullName -Destination (Join-Path $bin $finalName) -Force
}

# ── RTK (https://github.com/rtk-ai/rtk) ───────────────────────────────
Install-WinZipBinary "https://github.com/rtk-ai/rtk/releases/latest/download/rtk-x86_64-pc-windows-msvc.zip" "rtk*.exe" "rtk.exe"
rtk init -g
rtk init --show

# Exclude known-broken filters from the RTK hook (see "RTK filters can lie").
# Fresh install has no [hooks] section; if `rtk config` already shows one,
# edit %APPDATA%\rtk\config.toml by hand instead of appending a duplicate section.
$rtkCfg = "$env:APPDATA\rtk\config.toml"
New-Item -ItemType Directory -Force -Path (Split-Path $rtkCfg) | Out-Null
@'

[hooks]
exclude_commands = [
  "grep",          # 80-char truncation loses matches (rtk#1161)
  "ls",            # returns literal "(empty)" (rtk#1418)
  "curl",          # API/HTTP responses must arrive intact
  "playwright",    # E2E output crushed to ~9 tokens (rtk#1125, rtk#690)
  "pnpm --filter", # rtk misparses the --filter flag (rtk#259)
  "biome",         # lint filter printed "No issues found" over real errors
  "npx biome",
]
'@ | Add-Content -Path $rtkCfg
rtk config                          # verify the [hooks] section is loaded
rtk hook check "grep foo bar.txt"   # must print the command UNCHANGED

# ── Codebase Memory (https://github.com/DeusData/codebase-memory-mcp) ──
Install-WinZipBinary "https://github.com/DeusData/codebase-memory-mcp/releases/latest/download/codebase-memory-mcp-windows-amd64.zip" "codebase-memory-mcp*.exe" "codebase-memory-mcp.exe"
codebase-memory-mcp --version
codebase-memory-mcp config set auto_index true
codebase-memory-mcp config set auto_index_limit 50000

# ── LSP binaries (native npm) ─────────────────────────────────────────
npm install -g typescript-language-server pyright

# ── Official LSP plugins — connectors only, they need the binaries above ──
claude plugin marketplace update claude-plugins-official
claude plugin install typescript-lsp@claude-plugins-official
claude plugin install pyright-lsp@claude-plugins-official

# ── Claude-Mem (https://github.com/thedotmack/claude-mem) ─────────────
npx claude-mem install

# ── Verify ────────────────────────────────────────────────────────────
rtk --version
rtk gain                          # must work; if it fails you likely have
                                  # the wrong rtk ("Rust Type Kit" name collision)
codebase-memory-mcp --version
typescript-language-server --version
pyright --version
npx claude-mem status

# ── Configure claude-mem conservatively (native PowerShell, no jq) ────
# Short index of the most important decisions, filtered observation
# types/concepts, no full records. Model pinned to Haiku (already the default).
# Run AFTER `npx claude-mem install`.
$settings = "$env:USERPROFILE\.claude-mem\settings.json"
New-Item -ItemType Directory -Force -Path (Split-Path $settings) | Out-Null
if (-not (Test-Path $settings)) { '{}' | Set-Content -Path $settings -Encoding utf8 }

$json = Get-Content $settings -Raw | ConvertFrom-Json
$overrides = [ordered]@{
  CLAUDE_MEM_MODEL                        = "claude-haiku-4-5-20251001"
  CLAUDE_MEM_CONTEXT_OBSERVATIONS         = "10"
  CLAUDE_MEM_CONTEXT_SESSION_COUNT        = "3"
  CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES    = "bugfix,decision,discovery"
  CLAUDE_MEM_CONTEXT_OBSERVATION_CONCEPTS = "why-it-exists,problem-solution,gotcha,trade-off"
  CLAUDE_MEM_CONTEXT_FULL_COUNT           = "0"
  CLAUDE_MEM_CONTEXT_FULL_FIELD           = "narrative"
  CLAUDE_MEM_CONTEXT_SHOW_READ_TOKENS     = "false"
  CLAUDE_MEM_CONTEXT_SHOW_WORK_TOKENS     = "false"
  CLAUDE_MEM_CONTEXT_SHOW_SAVINGS_AMOUNT  = "false"
  CLAUDE_MEM_CONTEXT_SHOW_SAVINGS_PERCENT = "false"
  CLAUDE_MEM_CONTEXT_SHOW_TERMINAL_OUTPUT = "false"
  CLAUDE_MEM_CONTEXT_SHOW_LAST_SUMMARY    = "false"
  CLAUDE_MEM_CONTEXT_SHOW_LAST_MESSAGE    = "false"
  CLAUDE_MEM_FOLDER_CLAUDEMD_ENABLED      = "false"
}
foreach ($k in $overrides.Keys) {
  $json | Add-Member -NotePropertyName $k -NotePropertyValue $overrides[$k] -Force
}
$json | ConvertTo-Json -Depth 10 | Set-Content -Path $settings -Encoding utf8

# Verify
$Config = Get-Content `
    "$HOME\.claude-mem\settings.json" `
    -Raw | ConvertFrom-Json
$Config | Select-Object `
    CLAUDE_MEM_MODEL,
    CLAUDE_MEM_CONTEXT_OBSERVATIONS,
    CLAUDE_MEM_CONTEXT_SESSION_COUNT,
    CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES,
    CLAUDE_MEM_CONTEXT_OBSERVATION_CONCEPTS,
    CLAUDE_MEM_CONTEXT_FULL_COUNT,
    CLAUDE_MEM_CONTEXT_SHOW_LAST_SUMMARY,
    CLAUDE_MEM_CONTEXT_SHOW_LAST_MESSAGE

npx claude-mem restart
npx claude-mem status

# ── Context7 (https://github.com/upstash/context7) ────────────────────
# Pre-installed only on Coder workspaces (check /plugins there). Elsewhere:
npx ctx7 setup --claude   # guided setup: auth + install
# or manually (API key optional — free at context7.com/dashboard, higher rate limits):
claude mcp add --transport http context7 https://mcp.context7.com/mcp --header "CONTEXT7_API_KEY: <key>"

# ── Caveman ───────────────────────────────────────────────────────────
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman

# ── Ponytail ──────────────────────────────────────────────────────────
claude plugin marketplace add DietrichGebert/ponytail
claude plugin install ponytail@ponytail

# ── Claude HUD (https://github.com/jarrodwatts/claude-hud) ────────────
# Live context/cost/git in the status line. Install LAST: /claude-hud:setup (run below)
# overwrites the status line, so it must come after Caveman — whose plugin sets its own —
# for Claude HUD's to win. Needs Claude Code v1.0.80+, Node 18+.
claude plugin marketplace add jarrodwatts/claude-hud
claude plugin install claude-hud

# ── Session Report (official; per-session token/cache HTML report — see Part VI) ──
# Marketplace claude-plugins-official already added in the LSP step above.
# Known cosmetic bug: a missing manifest can list its skill twice under different namespaces.
claude plugin install session-report@claude-plugins-official

# Launch Claude Code and execute the following:
/reload-plugins
/doctor
/mcp
/plugin
/caveman lite
/claude-hud:setup   # guided status-line preset; run after /reload-plugins so it overrides
```

#### claude-mem settings.json — what the preset does

> **Don't hand-replace the whole file with this JSON.** `npx claude-mem install` writes `provider`, authentication, worker port, and other runtime keys into `~/.claude-mem/settings.json`. The commands above (`jq '. + {…}'` on Unix, `ConvertFrom-Json` → `Add-Member -Force` → `ConvertTo-Json` on Windows) **update only the selected fields and preserve everything else** — run them *after* the installer, never instead of it.

What each override buys, versus claude-mem defaults:

| Key | Value | Effect |
| --- | --- | --- |
| `CLAUDE_MEM_CONTEXT_OBSERVATIONS` | `10` (default 50) | Smaller session-start context. |
| `CLAUDE_MEM_CONTEXT_SESSION_COUNT` | `3` (default 10) | Less stale history carried in. |
| `CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES` | `bugfix,decision,discovery` | Drops routine `change`, `feature`, `refactor` entries. |
| `CLAUDE_MEM_CONTEXT_FULL_COUNT` | `0` | No expanded narratives injected. |
| `CLAUDE_MEM_CONTEXT_SHOW_LAST_SUMMARY` / `…_LAST_MESSAGE` | `false` | Previous session isn't duplicated into context. |
| `CLAUDE_MEM_CONTEXT_SHOW_READ_TOKENS` / `…_WORK_TOKENS` / `…_SAVINGS_AMOUNT` / `…_SAVINGS_PERCENT` / `…_TERMINAL_OUTPUT` | `false` | Token-stat metrics and the stats banner line don't occupy context. |
| `CLAUDE_MEM_FOLDER_CLAUDEMD_ENABLED` | `false` | claude-mem won't scatter extra `CLAUDE.md` files into project folders. |
| `CLAUDE_MEM_MODEL` | `claude-haiku-4-5-20251001` | Haiku generates the memory — cheaper (already the default; pinned explicitly). |

**Valid values (official):** observation *types* are `bugfix`, `feature`, `refactor`, `discovery`, `decision`, `change`; observation *concepts* include `gotcha`, `trade-off`, `why-it-exists`. `CLAUDE_MEM_CONTEXT_FULL_COUNT=0` is officially allowed.

#### Windows notes

- **No `bash`/`install.sh`.** On Windows `bash` typically means WSL, whose `install.sh` fetches a Linux binary into the WSL VM — useless to Windows Claude Code. `Install-WinZipBinary` downloads the real Windows `.zip` (`Invoke-WebRequest` + `Expand-Archive`), finds the `.exe`, and drops it in `~/.local/bin` under a canonical name.
- **`export PATH=…` → PowerShell.** The original bash `export PATH="$HOME/.local/bin:$PATH"` becomes STEP 0: `$env:PATH` for the current session plus `[Environment]::SetEnvironmentVariable(... "User")` to persist it. Open a new terminal after install so the persisted `PATH` is picked up.
- **`jq` → native JSON.** `Get-Content | ConvertFrom-Json` → `Add-Member -Force` → `ConvertTo-Json` replaces the `jq '. + {…}'` merge with no extra dependency. `-Depth 10` guards against deep-object truncation.
- **`$HOME`, `mktemp`, `[ -f … ]`** have no PowerShell equivalents inline — replaced by `$env:USERPROFILE`, `Set-Content` to a temp-free write, and `Test-Path`.
- Everything else (`npm`, `npx`, `claude …`) runs natively on Windows unchanged.
- **RTK hook fidelity on Windows:** the Claude Code hook intermittently returns output not faithful to the real command — duplicated lines, spurious markup ([#2176](https://github.com/rtk-ai/rtk/issues/2176)). The exit-code-beats-summary rule matters doubly here; if it bites often, drop the global hook (`rtk init -g`) and wrap only trusted commands manually (`rtk git diff`).


## Part VI — Measuring savings

Don't trust vendor numbers — measure on your own workflow:

```
rtk gain              # shell-output savings (add --history for per-command log)
npx claude-mem status # memory state; the session-start banner shows tokens read vs past work
```

Inside a session:

```
/context   # what occupies the context window right now (incl. per-MCP-server cost)
/cost      # token/cost usage of the current session
/usage     # current usage against your plan / rate limits
```

**Live status line — Claude HUD.** Instead of running `/context` on demand, [Claude HUD](https://github.com/jarrodwatts/claude-hud) keeps a context-fullness bar plus model, project path, git branch, and rate limits in the status line every turn — using Claude Code's native token data, not estimates (optional extra lines show tool/agent/todo activity). It's a monitoring aid, not a token saver. `/claude-hud:setup` runs a guided preset picker (Full/Essential/Minimal) and **overwrites the status line**, so install and set it up *after* Caveman, whose plugin sets its own — Claude HUD then wins. Requires Claude Code v1.0.80+ and Node 18+; install steps in Part V.

**Cross-session forensics — Session Report.** [session-report](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/session-report) (official) reads the last week of transcripts under `~/.claude/projects` and drops a self-contained HTML report in the working directory: input/output tokens, cache efficiency and miss causes, the most expensive prompts, subagent and skill usage, and per-session spend — sortable tables and bar charts, plus a skill that surfaces which projects or skills eat disproportionate tokens and whether one prompt skews the totals. This is the layer that actually answers *did it pay off*: whether RTK cut total spend, whether Caveman nets a saving, whether claude-mem isn't dragging in stale history, whether Context7 fires only when needed, and whether codebase-memory-mcp really reduced Read/Grep. Install, `/reload-plugins`, then invoke its `session-report` skill (ask for a session report). Known cosmetic bug: a missing manifest can list the skill twice under different namespaces — harmless to the report, but it clutters the skills list. Install steps in Part V.

**Rule:** enable one layer at a time, compare a week of `rtk gain` / `/cost` before and after, keep what pays for itself.

## Summary

| Tool | Owns | Expected effect |
| --- | --- | --- |
| RTK | shell, tests, Git, logs | 30–98% on noisy commands [measured] |
| codebase-memory-mcp | structural code navigation | ~10× on exploration, −9pp answer quality [study] |
| claude-mem (conservative config) | cross-session memory | saves re-investigation; ~5k/session index tax [measured] |
| TypeScript / Python LSP | definitions, references, type errors | replaces multi-KB grep+read chains [proxy] |
| Context7 | fresh library docs | indirect — fewer retry loops |
| Caveman (enable/disable on purpose) | terse agent replies | ~65% of output tokens [measured] |
| Ponytail (enable/disable on purpose) | minimal code output | ~70% less code on local A/B [measured] |
| Claude HUD (status-line plugin) | live context/cost/rate-limit visibility | no direct token savings — monitoring aid; overrides the status line |
| Session Report (official plugin) | forensic per-session token/cache/subagent/skill report | measurement — proves whether each layer nets out |
| Browser MCP (Playwright *or* Chrome DevTools, one at a time) | frontend testing / perf-network debugging | toggle per task — idle server is tool-surface tax, snapshots cost 10k+ tokens |

---

## Related

Three guides, one split — pick the layer you need:

- [Awesome AGENTS.md](https://github.com/khasky/awesome-agents-md) — the base, tool-agnostic ruleset every agent imports (one `AGENTS.md`). Token optimization sits on top of it.
- [Awesome Agent Skills](https://github.com/khasky/awesome-agent-skills) — portable `SKILL.md` skills every agent loads: code review, debugging, security and leak audits, code and text cleanup.
- **Claude Code Token Optimization** — *this repo:* the token-efficiency layer (RTK, LSP, Context7, `codebase-memory-mcp`, claude-mem, Caveman, Ponytail).
- [Agent MCP Integrations](https://github.com/khasky/agent-mcp-integrations) — MCP servers that connect agents to browsers, cloud, databases, infra, and domain APIs; the two browser servers referenced here are covered there in full.
- [Claude Code Security Audit](https://github.com/khasky/claude-code-security-audit) — the layered security-audit workflow (deep audit, continuous guardrails, scanners).

---

## License

Released under the [MIT license](LICENSE). The community tools this guide installs — RTK, codebase-memory-mcp, claude-mem, Caveman, Ponytail, Context7, Claude HUD — keep their own licenses; check each project before use.
