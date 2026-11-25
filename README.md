# Claude Code: Token Optimization & Context Management

[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE) [![Emojery](https://api.emojery.app/badge/github/khasky/claude-code-token-optimization.svg)](https://emojery.app/react?t=github/khasky/claude-code-token-optimization)

A practical guide to keeping Claude Code sessions cheap: what actually occupies the context window, which layers cut input, output, and cross-session cost, and how to prove each one pays for itself before you keep it.

> *If I were paying for every token a coding agent burns, what would I install first — and how would I know it worked?*

The tools split into four layers, each with its own token pool:

- **Built-in practices** — effort level and model choice, LSP navigation, dirty-context subagents, disciplined `/compact`, `/rewind` and `/clear`, and pruning MCP/plugin surface. No installs, biggest wins.
- **Input-side tooling** — a code-graph navigation layer (codebase-memory-mcp) and ast-grep for structural search plus codemods the CLI performs instead of the model.
- **Output-side plugins** — Caveman (terse prose) and Ponytail (minimal code), acting on disjoint parts of the reply. Opt-in: on current models they shorten the reply and still cost more per short task.
- **Docs on demand** — Context7, to stop the model hallucinating stale APIs.

Then a short list of tools **not** worth installing, installation for Linux/macOS and Windows, and a measurement layer (`/context`, `/usage`, Session Report) so you enable one thing at a time and keep only what earns its place.

> **Trust & scope note.** Except the official Anthropic plugins (the LSP connectors and Session Report), everything below is community-built (codebase-memory-mcp, ast-grep, Caveman, Ponytail, Context7) — not Anthropic products. Review any `curl | sh` install script before running it.

---

## Table of Contents

- [Part I — Built-in practices](#part-i--built-in-practices)
  - [Effort level and model choice](#effort-level-and-model-choice)
  - [1. LSP plugin — the first thing to add](#1-lsp-plugin--the-first-thing-to-add)
  - [2. Subagents as "dirty context" isolators](#2-subagents-as-dirty-context-isolators)
  - [3. /compact, /clear, and short tasks](#3-compact-clear-and-short-tasks)
  - [4. Disable unused MCP servers & plugins, keep `CLAUDE.md` lean](#4-disable-unused-mcp-servers--plugins-keep-claudemd-lean)
  - [Protect the cached prefix](#protect-the-cached-prefix)
- [Part II — Input-side tooling](#part-ii--input-side-tooling)
  - [5. codebase-memory-mcp — code-graph navigation](#5-codebase-memory-mcp--code-graph-navigation)
  - [6. ast-grep — structural search and controlled codemods](#6-ast-grep--structural-search-and-controlled-codemods)
  - [context-mode — a candidate, not a default](#context-mode--a-candidate-not-a-default)
  - [7. What to choose](#7-what-to-choose)
- [Part III — Output-side plugins](#part-iii--output-side-plugins)
  - [8. Caveman — terse replies](#8-caveman--terse-replies)
  - [9. Ponytail — least code that works](#9-ponytail--least-code-that-works)
- [Part IV — Docs on demand](#part-iv--docs-on-demand)
  - [10. Context7](#10-context7)
- [Part V — Tools I don't recommend](#part-v--tools-i-dont-recommend)
  - [11. RTK — shell-output compression](#11-rtk--shell-output-compression)
  - [12. Claude HUD — a status line that re-runs every turn](#12-claude-hud--a-status-line-that-re-runs-every-turn)
- [Part VI — Installation](#part-vi--installation)
  - [Linux/macOS (bash)](#linuxmacos-bash)
  - [Windows 10+](#windows-10)
- [Part VII — Measuring savings](#part-vii--measuring-savings)
- [Summary](#summary)
- [Related](#related)
- [License](#license)

## Part I — Built-in practices

### Effort level and model choice

Two session settings move the bill more than any plugin, and both are built in.

**Effort.** Fable models, Opus 5.5, Sonnet 5.5 and Haiku 5.5 always think: thinking cannot be turned off, and thinking tokens are billed as output. The effort level is what scales it. The levels are `low`, `medium`, `high`, `xhigh` and `max`; Fable 5.2 and the 5.5 models default to `medium` ([model configuration](https://code.claude.com/docs/en/model-config#adjust-effort-level)).

| Level | Use it for |
| --- | --- |
| `low` | quick exchanges you review yourself: a rename, a first sketch |
| `medium` | day-to-day work with a clear scope |
| `high` | a bug in an existing codebase, anything where edge cases are likely |
| `xhigh`, `max` | hard problems the agent works through alone; `max` is prone to overthinking, so test it before adopting it |

Set it with `/effort <level>` in a session, `claude --effort <level>` for one launch, the `CLAUDE_CODE_EFFORT_LEVEL` environment variable, or `effort` in a skill's or subagent's frontmatter. With an API key or a Claude subscription, changing effort mid-session keeps the prompt cache on Fable 5.1 and later and on the 5.5 models, so dropping a level for a routine stretch costs nothing.

**Model.** Sonnet handles most coding tasks and costs less than Opus; keep Opus for architecture decisions and multi-step reasoning. A model switch mid-session is the expensive kind of change: each model has its own cache, so the next request rereads the whole conversation uncached. Pick the model at the start of a task. Simple subagents take `model: haiku` in their definition (§2).

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

A subagent's requests are billed like any others. Put read-only exploration on a smaller model with `model: haiku` in the subagent definition. Its cache lives five minutes even on a subscription, where the main conversation gets an hour (see Protect the cached prefix).

### 3. /compact, /clear, and short tasks

Context accumulates junk faster than any tool trims it. Compact with explicit instructions:

```
/compact Preserve: current goal, changed files, confirmed architecture,
failing tests, unresolved issues.
Drop: successful command output, superseded plans,
exploratory dead ends, repeated file contents.
```

- Compaction instructions can also live in `CLAUDE.md`.
- Compact at a natural break, while the cache is warm. The summary request rereads the conversation: from cache that is a fraction of the context's size, after a break longer than the cache lifetime it is the whole history at full input price. Resuming an old session and compacting it first is the most expensive order.
- A dead end you want to drop entirely → `/rewind` to the turn before it. Rewinding truncates back to a prefix that is already cached; compaction builds a new one.
- For unrelated tasks use `/clear` — it costs nothing, while stale context is reprocessed with every message.
- After several failed approaches, a **fresh session with a good short brief** beats compressing old junk again (see managing costs).
- A 1M-context model fills a long way before automatic compaction. `/autocompact <tokens>` (for example `/autocompact 500k`) sets the window yourself; `/autocompact auto` returns to the model's default.

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

### Protect the cached prefix

The lines `/context` prints are not just size — they are also position, and position is what decides whether you pay full price for them again on the next turn.

Prompt caching on the Messages API underneath is a **prefix match**. Claude Code orders every request so the parts that rarely change come first — system prompt and tool definitions, then project context (`CLAUDE.md`, auto memory, unscoped rules), then the conversation — and a cache entry is reusable only up to the first byte that differs. Cached tokens are read at roughly **one tenth** of the input price; writing the cache costs about **1.25×** normal input (5-minute TTL) or **2×** (1-hour TTL), so a prefix that survives two turns has already paid for itself and one that never survives is pure overhead.

Which mid-session actions cost a full reread is not intuitive ([prompt caching in Claude Code](https://code.claude.com/docs/en/prompt-caching)).

**These reread the conversation uncached on the next request:**

- **Switching models.** Each model has its own cache. `opusplan` switches on every plan-mode toggle, and a skill or command whose `model` differs from the session's is a switch for that turn.
- **Turning fast mode on or off.** The two speeds keep separate caches.
- **`/compact`.** The conversation layer is replaced and project context is rebuilt with it (§3 for when that is cheap).
- **Changing effort** on models older than Fable 5.1 and the 5.5 family, and on Amazon Bedrock, Google Cloud's Agent Platform or a gateway.
- **Adding or removing a tool definition while tool search is off or unavailable:** an MCP server connecting, a server disabled in `/mcp`, a deny rule on a bare tool name.
- **Upgrading Claude Code.** The first conversation after the restart builds its cache from the top.

**These keep the cache:**

- **Connecting or disabling an MCP server with tool search on,** the default on supported models. The tool list from the conversation's first request is kept for the whole conversation, and a server that connects later supplies its tools as deferred definitions.
- **Editing `CLAUDE.md`.** Claude Code reads the project-root and user-level files at session start and holds them in memory. A mid-session edit is not applied and not charged until `/clear`, `/compact` or a restart.
- **Changing effort** on Fable 5.1 and later, Opus 5.5, Sonnet 5.5 and Haiku 5.5, with an API key or a Claude subscription.
- **Invoking a skill, entering plan mode, switching permission mode or output style, `/recap`, `/rewind`, spawning a subagent.** Each appends to the conversation or leaves the request alone.
- **A plugin's skills, commands, agents and hooks.** Their content is appended after the conversation. A plugin's MCP servers follow the MCP rule above.

A long stable `CLAUDE.md` is served at cache-read rates after the first turn: its price is the size it adds to every request, never a reprice of the conversation.

**Cache lifetime.** A cached prefix expires after a period of inactivity, and the first message after the gap reprocesses the whole context. On a Claude subscription within plan usage the main conversation gets a one-hour TTL. With an API key, usage credits or a cloud provider it gets five minutes, and subagents get five minutes everywhere. The `promptCacheTtl` and `subagentPromptCacheTtl` settings (`5m` or `1h`, Claude Code 2.1.242 or later) choose it yourself: an hour pays off when you step away and come back, and costs the higher write rate on bursts that never idle past five minutes.

`/usage` is where you check this. Its `Prompt cache (main)` line gives the share of input read from cache, the miss count, and the likely cause of the last miss when Claude Code can name one (`likely cause: tool definitions changed`). Persistent misses on a session whose configuration you did not touch mean something in the prefix is varying — find it, because that one string is repricing your entire context on every message.

**One measurement note:** do not size prompts with `tiktoken` or any GPT tokenizer. It is a different tokenizer and it undercounts Claude by roughly 15–20% on prose, considerably more on code and non-English text. Use `/context` inside a session, or the API's `count_tokens` endpoint with the model you actually run.

## Part II — Input-side tooling

### 5. codebase-memory-mcp — code-graph navigation

codebase-memory-mcp owns finding code. It replaces `grep → read → grep → read` chains with structural queries and cuts the token cost of exploration.

**Why codebase-memory-mcp wins on tokens**

It builds a persistent knowledge graph of the repo (functions, classes, calls, dependencies, routes, cross-service links). Instead of dozens of greps and full-file reads: one structural query like *"who calls ProcessOrder?"*. No LLM inside; indexing runs locally, no API key (repo).

Its win is preventive: unnecessary grep/read commands never run, so their output never reaches the context.

**Benchmark:**

- **[study]** 31 real-world repos (arXiv:2603.27277): **~10× fewer tokens**, **2.1× fewer tool calls** than file exploration. Trade-off: **83% answer quality vs 92%** for the file-exploration agent — graph-first is cheaper, but the agent sometimes still needs to check source code.

**The installer adds more than a server.** `install.sh` ends by running `codebase-memory-mcp install`, which for Claude Code registers the MCP server (17 tools) plus a skill, three graph agents and hooks: `SessionStart`, `SubagentStart`, `PreToolUse` on `Grep`/`Glob`/`Bash`, and a coverage note after `Read`. The hooks are on by default and inject graph context into the session, the per-turn tax §4 warns about. `codebase-memory-mcp install --no-hooks` removes them and keeps the MCP registration, and the choice persists across updates. Start without hooks; turn them on only if `/usage` shows the agent still exploring with grep.

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

### 6. ast-grep — structural search and controlled codemods

[ast-grep](https://ast-grep.github.io/) matches the syntax tree instead of the text, and can rewrite what it matches. It is a plain CLI — no MCP server, no plugin — so the agent runs it through the shell and it adds **zero** permanent tool-surface tax (§4).

**Why it saves tokens — two different pools:**

- **Input.** `rg 'fetch\('` also hits comments, strings, docs and dead code, so the agent reads whole files to sort real calls from noise. `ast-grep -p 'fetch($$$ARGS)'` returns only real call expressions — the noise never enters the context.
- **Output, and this is the bigger one.** A rename across 40 files done by the model costs 40 file reads in and 40 edits out. Done by ast-grep it costs one command out; the CLI performs the edits and only `git diff --stat` comes back.

**Install per project**, so the version is pinned in the lockfile and the agent doesn't depend on a global `PATH`:

```bash
npm install --save-dev @ast-grep/cli
npx ast-grep --version
```

The command is `ast-grep`. Don't use the `sg` short alias on Linux — it collides with the system `setgroups` command.

**Pattern syntax.** Two metavariables cover almost everything; both must be uppercase:

| Pattern | Matches |
| --- | --- |
| `$NODE` | exactly one AST node (one argument, one expression, one name) |
| `$$$ARGS` | zero or more nodes — any argument list |

Repeating the same name means *the same code in both places*: `$VALUE && $VALUE()` matches `user.cb && user.cb()` but not `a && b()`.

**Search examples.** Identical in bash and PowerShell — single quotes stop both shells from expanding `$`:

```bash
npx ast-grep -p 'console.log($$$ARGS)' -l ts src        # -l tsx for TSX files
npx ast-grep -p '$OBJECT.save($$$ARGS)' -l ts src       # repository.save(user), db.save(entity, opts)
npx ast-grep -p 'requests.get($$$ARGS)' -l python src
```

**Rewrites — always search first, then `--interactive`:**

```bash
npx ast-grep -p '$PROP && $PROP()' -l ts src                                 # 1. see every match
npx ast-grep -p '$PROP && $PROP()' -r '$PROP?.()' --interactive -l ts src    # 2. accept/reject per hunk
git diff --stat && npm test                                                  # 3. verify
```

⚠️ **Never `ast-grep scan --update-all`** unless a bulk unconditional rewrite is exactly what you asked for — it applies every fix with no confirmation, and a bad pattern then costs far more tokens to unwind than the codemod saved.

**Give the rules to the agent as a skill, not as `CLAUDE.md` text.** Memory files ride along with every message (§4); a skill costs only its name and description until it's actually used.

```bash
mkdir -p .claude/skills/ast-grep
cat > .claude/skills/ast-grep/SKILL.md <<'EOF'
---
name: ast-grep
description: Syntax-aware structural search and controlled codemods (TypeScript, TSX, JavaScript, Python) via the project-local ast-grep CLI. Prefer LSP for symbol references and rg for literal text.
---

Run the project-local CLI: `npx ast-grep`.

1. Search before rewriting; scope to the narrowest relevant directory.
2. `$NODE` matches one AST node, `$$$ARGS` matches zero or more.
3. Review every match before changing code.
4. Rewrite only with `--interactive`. Never use `--update-all` unless the user explicitly asks for an unconditional bulk rewrite.
5. After a rewrite: inspect `git diff`, then run targeted tests.

Tool routing: literal text or an exact error string -> `rg`; definition, references, rename -> LSP; architecture and call chains -> codebase-memory-mcp; repeated syntax patterns and bulk codemods -> ast-grep.
EOF
```

```powershell
$SkillDir = ".claude\skills\ast-grep"
New-Item -ItemType Directory -Path $SkillDir -Force | Out-Null
@'
---
name: ast-grep
description: Syntax-aware structural search and controlled codemods (TypeScript, TSX, JavaScript, Python) via the project-local ast-grep CLI. Prefer LSP for symbol references and rg for literal text.
---

Run the project-local CLI: `npx ast-grep`.

1. Search before rewriting; scope to the narrowest relevant directory.
2. `$NODE` matches one AST node, `$$$ARGS` matches zero or more.
3. Review every match before changing code.
4. Rewrite only with `--interactive`. Never use `--update-all` unless the user explicitly asks for an unconditional bulk rewrite.
5. After a rewrite: inspect `git diff`, then run targeted tests.

Tool routing: literal text or an exact error string -> `rg`; definition, references, rename -> LSP; architecture and call chains -> codebase-memory-mcp; repeated syntax patterns and bulk codemods -> ast-grep.
'@ | Set-Content -Path "$SkillDir\SKILL.md" -Encoding utf8
```

Check it registered with `/skills`; invoke it manually with `/ast-grep`, or let the `description` trigger it automatically on structural-search requests.

**Permissions — don't allow the whole CLI.** `"Bash(npx ast-grep *)"` also auto-approves every rewrite command. Pin the allowlist to read-only npm scripts (`npm pkg set "scripts.ast:scan=ast-grep scan"`, `npm pkg set "scripts.ast:test=ast-grep test"`) and leave `npx ast-grep … -r` requiring approval, in `.claude/settings.json`:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm run ast:scan *)",
      "Bash(npm run ast:test *)",
      "PowerShell(npm run ast:scan *)",
      "PowerShell(npm run ast:test *)"
    ]
  }
}
```

**Optional — patterns worth keeping.** One-off searches need only `-p`. For rules that live in Git, `npx ast-grep new project` scaffolds `sgconfig.yml`, `rules/`, `rule-tests/`, `utils/`:

```yaml
# rules/no-console-log.yml
id: no-console-log
language: TypeScript
severity: warning
message: Avoid console.log in production code.
rule:
  pattern: console.log($$$ARGS)
```

Run one rule with `npx ast-grep scan --rule rules/no-console-log.yml src`, all of them with `npm run ast:scan`.

**Routing — ast-grep is not a replacement for the other three:**

| Task | Tool |
| --- | --- |
| Exact string, error message, TODO | `rg` |
| Definition, references, rename one symbol | LSP (§1) |
| Call chains, dependencies, impact | codebase-memory-mcp (§5) |
| The same syntactic construct in many places | `ast-grep` |
| Bulk codemod across files | `ast-grep --interactive` |

**Benchmark:** no vendor token number. The mechanism is the same as LSP's — matches replace file reads, and the CLI does the editing instead of the model [proxy].

### context-mode — a candidate, not a default

[context-mode](https://github.com/mksglu/context-mode) targets the pool nothing above touches: bulky tool output. Hooks redirect large reads, fetches and build or test runs into a local sandbox, the full output goes into a per-project SQLite index, and the agent gets back only what a script prints or a search matches.

It stays out of the stack above for three reasons:

- **The headline number is bytes, not tokens or dollars.** The README's 99.4% is a median over the author's own fixtures of 10 KB and larger. The one independent comparison that lists it ([ComputingForGeeks](https://computingforgeeks.com/reduce-claude-code-token-usage-tools/), April 2026) verified the install and ran no benchmark.
- **It changes what the agent sees,** the concern that rules out RTK (Part V §11). The raw output is kept and searchable where RTK rewrites it, so a doubt can be checked; whether the agent does check is the thing to watch.
- **Terms and trust.** The license is Elastic 2.0, source-available and not open source, and `ctx_execute` runs code on the host as your user.

Worth a trial when the expensive lines in `/usage` are logs, long test runs and fetched pages, not navigation. Run it under the Part VII rule: alone, for a week, against a baseline.

### 7. What to choose

**Add codebase-memory-mcp when:** the main cost is Read/Grep/Glob; large repo or monorepo; agent keeps re-exploring call chains; you use both Claude Code and Codex; the goal is total token reduction. On a small repo where navigation isn't the main cost, the built-in practices in Part I already cover most of the noise.

**Add ast-grep when:** the repo has repeated syntactic patterns (a deprecated API, a logger call, an old idiom) or a codemod is coming. It costs nothing while idle — a CLI, not a server — so on a JS/TS/Python repo there is no reason to skip it.

**Best practical setup — the whole input layer**

They barely conflict, so combine them with a clear split of responsibilities:

| Tool | Owns |
| --- | --- |
| codebase-memory-mcp | current code structure, callers, dependencies, impact analysis |
| ast-grep | repeated syntax patterns; bulk rewrites performed by the CLI, not by the model |

Both act before the model reads anything, so they don't compete: one answers *where is the code*, the other *rewrite this construct everywhere*. Shell output stays raw on purpose — see Part V.

## Part III — Output-side plugins

On Fable and the 5.5 models the first output lever is the effort level (Part I): thinking is billed as output and no style plugin touches it. The two plugins below shorten what is left, the visible prose and the generated code.

**Benchmark on current models.** The [silverback benchmark](https://github.com/khasky/silverback#benchmark) runs each plugin alone on seeded tasks (Claude Code 2.1.289, `--effort low`, 5 attempts per task). Held-out split, per task:

| | baseline | Caveman | Ponytail |
| --- | --: | --: | --: |
| `claude-opus-5-5`, words in the answer | 100 | 80 | 82 |
| `claude-opus-5-5`, source lines on open requests | 49 | 62 | 28 |
| `claude-opus-5-5`, cost | $0.095 | $0.131 | $0.116 |
| `claude-sonnet-5-5`, words in the answer | 77 | 50 | 75 |
| `claude-sonnet-5-5`, source lines on open requests | 47 | 47 | 28 |
| `claude-sonnet-5-5`, cost | $0.052 | $0.068 | $0.064 |

Both do what they claim: Caveman cuts words, Ponytail cuts lines. Both also cost more per task than no plugin. The tasks are short, so each one pays for the plugin's injected instructions and saves little output. Treat both as opt-in for long, output-heavy sessions, and keep either only if a week of `/usage` says it nets out (Part VII). A ruleset that already asks for terse replies and minimal diffs covers both pools in one injected text (silverback in the same benchmark on `claude-opus-5-5`: 57 words, 20 lines, $0.130); don't stack the plugins on top of it.

### 8. Caveman — terse replies

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

### 9. Ponytail — least code that works

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

### 10. Context7

Context7 fetches current, version-specific library documentation on demand instead of the model guessing from stale training data. It saves tokens **indirectly**: fewer hallucinated APIs → fewer failed attempts and retry loops. Each docs fetch itself costs tokens, so it pays off on unfamiliar or fast-moving libraries, not on every question. Pre-installed only on Coder workspaces (check `/plugins`); on a local machine install it — see Installation below.

**Reach for it when the task touches** a new or updated library version, an external API, framework documentation, a migration guide, or an unfamiliar SDK. **Skip it for** CSS tweaks, local refactors, or business-logic changes on code you already know — a docs fetch there is pure overhead.

**Benchmark:** no meaningful direct number — the win is avoided retry loops, not compressed output.

## Part V — Tools I don't recommend

Popular token-savers that cost more than they save, plus one monitoring aid whose bill arrives in CPU. The failure mode that matters is not "saves less than advertised" — it's a tool that changes what the agent *sees*, because then a defect is indistinguishable from the truth and the agent acts on fiction. The second failure mode is quieter: a cost no token counter reports, so nothing in `/usage` or Session Report ever flags it.

### 11. RTK — shell-output compression

[RTK](https://github.com/rtk-ai/rtk) wraps shell commands and rewrites their output into a compact summary, hooked globally into Claude Code. The idea is sound; the fidelity isn't. Filtered output arrives with no marker of what was dropped or altered, so the agent has nothing to distrust.

**Measured failures, one project, one week** (each re-verified afterwards with the built-in tools and `git diff`):

- **Fabricated a match and dropped a real one.** A repo search reported a hit in a file that does not contain the string anywhere, and omitted the real hit in `package.json`. The wrong conclusion ("nothing runs this config") became a task that turned out to be already done.
- **Rewrote non-grep stdout.** `node -e` printing `package.json` scripts returned 34 of 36: one entry silently missing, another's value altered. File unmodified, rerun printed 36/36. Anything crossing the hook is exposed, not just wrapped tools.
- **False zero.** `0 matches` on a file that had three, nearly deleting docs for live API.
- **Invented counters and mislabeled lines.** A single-file search reported "2 matches in 2 files"; one result had no file name and its text came from a neighboring line, with a `:0:` column plain `grep` never prints.
- **Truncated the match itself.** `grep -n "^export function bindingForShortsBar"` printed the line with the searched prefix cut off; other lines cut at ~80 chars with an ellipsis.

Three of the five arrived with a `rtk: Failed to resolve 'rg' via PATH, falling back to direct exec` banner. Two came with **no banner at all** — the wrapper intervenes silently.

**Same classes are open upstream** (all still open as of August 2026):

- [#2301](https://github.com/rtk-ai/rtk/issues/2301) — the Claude Code hook rewrites `rg` into BSD-grep mode; path-less searches "silently return 0 matches"
- [#2806](https://github.com/rtk-ai/rtk/issues/2806) — false negatives from `rtk rg`
- [#3220](https://github.com/rtk-ai/rtk/issues/3220) — "TypeScript: No errors found" printed on a non-zero exit, so a `tsc` that never ran reads as a clean typecheck
- [#2317](https://github.com/rtk-ai/rtk/issues/2317) — filters mask hard failures with benign summaries (a pytest collection error becomes "No tests collected")
- [#3370](https://github.com/rtk-ai/rtk/issues/3370) — `rtk read --max-lines N` delivers N/2 lines
- [#3339](https://github.com/rtk-ai/rtk/issues/3339) — inflated savings metric and "passthrough that doesn't pass through"

**Why config can't fix it.** RTK still has no allowlist mode — requested in [#2231](https://github.com/rtk-ai/rtk/issues/2231), open since June 2026. The only scoping knob is an `exclude_commands` blacklist, so every command class has to burn you once before you can exclude it — and the passthrough path itself has broken (#3339). The economics are asymmetric: 60% off a `git diff` is worth far less than one fabricated match sending the agent to edit the wrong file, plus the review time to catch it.

**Instead:** leave shell output raw. Claude Code already caps tool results, and real noise is cheaper to cut at the source — `git diff --stat`, `--quiet` / `--silent` test flags, `| head`, `| wc -l`, narrower paths. Zero fidelity risk. A `PreToolUse` hook that appends `| tail -n 100` to a test command automates the same idea ([example in managing costs](https://code.claude.com/docs/en/costs#offload-processing-to-hooks-and-skills)): it narrows the command in plain sight and leaves the output it returns untouched.

### 12. Claude HUD — a status line that re-runs every turn

[Claude HUD](https://github.com/jarrodwatts/claude-hud) renders context fullness, model, project path, git branch, and rate limits into the status line. It costs no tokens; the price is machine time, paid on every render, in every open session.

`/claude-hud:setup` writes a `statusLine` entry into `~/.claude/settings.json`: a shell pipeline that re-resolves the plugin directory, then execs the HUD. Claude Code re-runs that whole command on every status-line refresh — a new assistant message, a finished `/compact`, a permission-mode change, a vim-mode toggle. `refreshInterval`, the plugin's documented fix for timers going stale between messages, adds timer-driven renders on top of those.

**Measured** (Windows 10, 16 logical cores, claude-hud 0.6.0, the command exactly as `/claude-hud:setup` left it):

```text
shell wrapper (bash + stty + awk + ls + sort + tail)   ~110 ms   6 processes
node dist/index.js                                     ~300 ms   1 process
one render                                             ~410 ms   ~7 processes
```

At one render per second, six concurrent sessions spend 2.5 of 16 logical cores drawing a status line. None of it appears as a culprit in a process monitor: every process lives 400 ms and exits, so the symptom is 100% CPU, a 220-process tree, and no visible hog.

The shell wrapper is the Windows-heavy half — 110 ms for six spawns there, less on Linux and macOS — while the 300 ms of Node is not platform-specific. Current Claude Code sets `COLUMNS` natively, so the `stty`/`awk` width probe inside that wrapper is a legacy fallback for older versions: dead weight on every render.

**Instead:** run `/context` and `/usage` when you want the numbers — between calls they cost nothing. If you want the HUD anyway, keep one session open rather than six, leave `refreshInterval` unset, and remember `CLAUDE_HUD_DISABLE=1` suppresses it per launch without touching `settings.json`.

## Part VI — Installation

### Linux/macOS (bash)

```bash
# Codebase Memory (https://github.com/DeusData/codebase-memory-mcp)
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash
export PATH="$HOME/.local/bin:$PATH"
codebase-memory-mcp install -y --no-hooks   # keep the MCP server, drop the per-session hooks (Part II §5)
codebase-memory-mcp config set auto_index true
codebase-memory-mcp config set auto_index_limit 50000

# ast-grep (https://ast-grep.github.io) — per project, not machine-wide: run this
# inside each repo, then add the skill and the permissions from Part II §6.
cd /path/to/project
npm install --save-dev @ast-grep/cli
npm pkg set "scripts.ast:scan=ast-grep scan" "scripts.ast:test=ast-grep test"
npx ast-grep --version

# LSP binaries
npm install -g typescript-language-server pyright

# Official LSP plugins — connectors only, they need the binaries above
# (terminal form; inside a session use /plugin install …)
claude plugin marketplace update claude-plugins-official
claude plugin install typescript-lsp@claude-plugins-official
claude plugin install pyright-lsp@claude-plugins-official

# Verify
codebase-memory-mcp --version
typescript-language-server --version
pyright --version

# Context7 (https://github.com/upstash/context7)
npx ctx7 setup --claude   # guided setup: auth + install
# or manually (API key optional — free at context7.com/dashboard, higher rate limits):
claude mcp add --transport http context7 https://mcp.context7.com/mcp --header "CONTEXT7_API_KEY: <key>"

# Caveman — optional, measure before keeping (Part III)
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman

# Ponytail — optional, measure before keeping (Part III)
claude plugin marketplace add DietrichGebert/ponytail
claude plugin install ponytail@ponytail

# Session Report (official; per-session token/cache/subagent/skill HTML report — see Part VII).
# Marketplace claude-plugins-official already added in the LSP step above.
# Known cosmetic bug: a missing manifest can list its skill twice under different namespaces.
claude plugin install session-report@claude-plugins-official

# Launch Claude Code and execute the following:
/reload-plugins
/doctor
/mcp
/plugin
```

### Windows 10+

**Prerequisites:** PowerShell 7+ (`pwsh`), Node.js + npm on `PATH`, Claude Code CLI (`claude`). **Do NOT use the project's `install.sh` script on Windows.**

> ⚠️ **Why not `bash … install.sh`:** on Windows `bash` usually resolves to **WSL** (or Git Bash). Under WSL the installer detects `os: linux`, downloads a **Linux** binary, and drops it inside the WSL filesystem (`/home/<user>/.local/bin/`) — invisible to and unrunnable by Windows-native Claude Code. The project publishes a real Windows `.zip` binary (`codebase-memory-mcp-windows-amd64.zip`), so download that directly with PowerShell — no bash, no WSL, no Git Bash. Windows-on-ARM: swap `amd64` for the `arm64` asset.

```powershell
# ── STEP 0: PATH + a helper that installs a GitHub Windows .zip binary ──
# The binary lives in ~/.local/bin (= %USERPROFILE%\.local\bin). PowerShell
# won't see it until this dir is on PATH — session + persisted for the user.
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

# ── Codebase Memory (https://github.com/DeusData/codebase-memory-mcp) ──
Install-WinZipBinary "https://github.com/DeusData/codebase-memory-mcp/releases/latest/download/codebase-memory-mcp-windows-amd64.zip" "codebase-memory-mcp*.exe" "codebase-memory-mcp.exe"
codebase-memory-mcp --version
codebase-memory-mcp config set auto_index true
codebase-memory-mcp config set auto_index_limit 50000

# ── ast-grep (https://ast-grep.github.io) ─────────────────────────────
# Per project, not machine-wide: run this inside each repo, then add the
# skill and the permissions from Part II §6.
Set-Location C:\path\to\project
npm install --save-dev @ast-grep/cli
npm pkg set "scripts.ast:scan=ast-grep scan" "scripts.ast:test=ast-grep test"
npx ast-grep --version

# ── LSP binaries (native npm) ─────────────────────────────────────────
npm install -g typescript-language-server pyright

# ── Official LSP plugins — connectors only, they need the binaries above ──
claude plugin marketplace update claude-plugins-official
claude plugin install typescript-lsp@claude-plugins-official
claude plugin install pyright-lsp@claude-plugins-official

# ── Verify ────────────────────────────────────────────────────────────
codebase-memory-mcp --version
typescript-language-server --version
pyright --version

# ── Context7 (https://github.com/upstash/context7) ────────────────────
# Pre-installed only on Coder workspaces (check /plugins there). Elsewhere:
npx ctx7 setup --claude   # guided setup: auth + install
# or manually (API key optional — free at context7.com/dashboard, higher rate limits):
claude mcp add --transport http context7 https://mcp.context7.com/mcp --header "CONTEXT7_API_KEY: <key>"

# ── Caveman — optional, measure before keeping (Part III) ─────────────
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman

# ── Ponytail — optional, measure before keeping (Part III) ────────────
claude plugin marketplace add DietrichGebert/ponytail
claude plugin install ponytail@ponytail

# ── Session Report (official; per-session token/cache HTML report — see Part VII) ──
# Marketplace claude-plugins-official already added in the LSP step above.
# Known cosmetic bug: a missing manifest can list its skill twice under different namespaces.
claude plugin install session-report@claude-plugins-official

# Launch Claude Code and execute the following:
/reload-plugins
/doctor
/mcp
/plugin
```

#### Windows notes

- **No `bash`/`install.sh`.** On Windows `bash` typically means WSL, whose `install.sh` fetches a Linux binary into the WSL VM — useless to Windows Claude Code. `Install-WinZipBinary` downloads the real Windows `.zip` (`Invoke-WebRequest` + `Expand-Archive`), finds the `.exe`, and drops it in `~/.local/bin` under a canonical name.
- **`export PATH=…` → PowerShell.** The original bash `export PATH="$HOME/.local/bin:$PATH"` becomes STEP 0: `$env:PATH` for the current session plus `[Environment]::SetEnvironmentVariable(... "User")` to persist it. Open a new terminal after install so the persisted `PATH` is picked up.
- Everything else (`npm`, `npx`, `claude …`) runs natively on Windows unchanged.


## Part VII — Measuring savings

Don't trust vendor numbers — measure on your own workflow. Inside a session:

```
/context   # what occupies the context window right now (incl. per-MCP-server cost)
/usage     # session cost, tokens by model, prompt-cache line, plan limits (/cost is an alias)
/insights  # HTML report on how recent sessions went: friction points, misread requests
```

**`/usage` first.** Its Session block carries the cost estimate, tokens by model and the `Prompt cache (main)` line (Part I). On a Pro, Max, Ultra, Team or Enterprise plan it also attributes recent usage to skills, subagents, plugins and individual MCP servers, and lists a recommendation for anything behind 10% or more of it, such as `25%  Trim ~/code/api/CLAUDE.md · 49k tokens per request`. `f` applies the ones Claude Code can act on (2.1.305 or later); `d` and `w` switch between the last 24 hours and the last 7 days. That attribution is the per-plugin and per-server answer the layers above need.

A status-line HUD turns those on-demand calls into a command that re-runs every turn; the CPU arithmetic is in Part V §12.

**Cross-session forensics — Session Report.** [session-report](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/session-report) (official) reads the last week of transcripts under `~/.claude/projects` and drops a self-contained HTML report in the working directory: input/output tokens, cache efficiency and miss causes, the most expensive prompts, subagent and skill usage, and per-session spend — sortable tables and bar charts, plus a skill that surfaces which projects or skills eat disproportionate tokens and whether one prompt skews the totals. This is the layer that actually answers *did it pay off*: whether Caveman nets a saving, whether Context7 fires only when needed, and whether codebase-memory-mcp really reduced Read/Grep. Install, `/reload-plugins`, then invoke its `session-report` skill (ask for a session report). Known cosmetic bug: a missing manifest can list the skill twice under different namespaces — harmless to the report, but it clutters the skills list. Install steps in Part VI.

**Rule:** enable one layer at a time, compare a week of `/usage` and Session Report before and after, keep what pays for itself.

## Summary

| Tool | Owns | Expected effect |
| --- | --- | --- |
| Effort level and model choice (built in) | thinking tokens, per-token price | the first lever: thinking is billed as output and no plugin touches it |
| codebase-memory-mcp | structural code navigation | ~10× on exploration, −9pp answer quality [study] |
| ast-grep (CLI — no MCP or plugin surface) | syntax-pattern search, bulk codemods | matches instead of file reads; the CLI edits, the model doesn't [proxy] |
| TypeScript / Python LSP | definitions, references, type errors | replaces multi-KB grep+read chains [proxy] |
| Context7 | fresh library docs | indirect — fewer retry loops |
| Caveman (opt-in, measure first) | terse agent replies | ~65% of prose output [measured]; costlier per short task on 5.5 models [benchmark] |
| Ponytail (opt-in, measure first) | minimal code output | ~70% less code on local A/B [measured]; costlier per short task on 5.5 models [benchmark] |
| `/usage` (built in) | session cost, cache hit rate, per-plugin and per-server attribution | measurement: the first place to look |
| Session Report (official plugin) | forensic per-session token/cache/subagent/skill report | measurement — proves whether each layer nets out |
| Browser MCP (Playwright *or* Chrome DevTools, one at a time) | frontend testing / perf-network debugging | toggle per task — idle server is tool-surface tax, snapshots cost 10k+ tokens |
---

## Related

Three guides, one split — pick the layer you need:

- [Silverback](https://github.com/khasky/silverback) — the base, tool-agnostic ruleset every agent imports (one `AGENTS.md`). Token optimization sits on top of it.
- [Awesome Agent Skills](https://github.com/khasky/awesome-agent-skills) — portable `SKILL.md` skills every agent loads: code review, debugging, security and leak audits, code and text cleanup.
- **Claude Code Token Optimization** — *this repo:* the token-efficiency layer (LSP, `codebase-memory-mcp`, ast-grep, Context7, Caveman, Ponytail).
- [Agent MCP Integrations](https://github.com/khasky/agent-mcp-integrations) — MCP servers that connect agents to browsers, cloud, databases, infra, and domain APIs; the two browser servers referenced here are covered there in full.
- [Claude Code Security Audit](https://github.com/khasky/claude-code-security-audit) — the layered security-audit workflow (deep audit, continuous guardrails, scanners).

---

## License

Released under the [MIT license](LICENSE). The community tools this guide installs — codebase-memory-mcp, ast-grep, Caveman, Ponytail, Context7 — keep their own licenses; check each project before use.
