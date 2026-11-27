# Harness Mechanics and Community Recipes for Self-Improvement / Automation Loops (Claude Code, Codex CLI, Cursor, Gemini CLI, Aider)

Research date: 2026-11-27. Claude Code facts below come from the current official docs at code.claude.com (fetched directly). These docs carry version notes up to about v2.1.330. OpenAI, Cursor, Gemini (except one raw GitHub doc), the Anthropic CDN PDF and most third-party blogs could NOT be fetched from this sandbox (DNS-blocked). Those facts come from search-result snippets, so each one is flagged where precision matters.

---

## 1. Claude Code hooks: events, capabilities, decision JSON, config format

### Takeaway
Claude Code now has about 35 hook events and five handler types (`command`, `http`, `mcp_tool`, `prompt`, `agent`). The self-improvement loop primitives are: a `Stop` that blocks with `decision:"block"`/`reason` (or a `prompt`/`agent` hook that returns `{ok:false, reason}`), `SessionStart`/`UserPromptSubmit`/`PostToolUse` that inject `additionalContext`, `PreCompact` whose stdout becomes compaction instructions, `PostCompact` that receives `compact_summary`, and a `SessionEnd` that can't block and has a 1.5 s default timeout. Every hook receives `transcript_path`.

### Cited Findings

**Config format (settings.json)**
- Hooks can live in `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed policy settings, a plugin's `hooks/hooks.json`, or skill/subagent frontmatter. They nest as `"hooks": { "<EventName>": [ { "matcher": "...", "hooks": [ { "type": "command", "command": "...", "timeout": 30 } ] } ] }`. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Handler fields: `type` (required: `command` | `http` | `mcp_tool` | `prompt` | `agent`); `if` (permission-rule syntax such as `"Bash(git *)"`, evaluated only on PreToolUse/PostToolUse/PostToolUseFailure/PermissionRequest/PermissionDenied/PreSkillShellCommand); `timeout` in seconds (defaults: 600 for command/http/mcp_tool, 30 for prompt, 60 for agent); `statusMessage`, `once` (skill frontmatter only), `async`, `asyncRewake`, `shell`, `onFailure`. Command handlers also take `command` and `args`; http handlers take `url`, `headers`, `allowedEnvVars`; MCP handlers take `server`, `tool`, `input`; prompt/agent handlers take `prompt` (with a `$ARGUMENTS` placeholder) and `model`. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Matcher syntax: `"*"`, `""` or an omitted matcher matches everything. Letters, digits, `_ - , |` and spaces form exact-match lists such as `Edit|Write`. Any other character turns the matcher into a JS regex such as `mcp__memory__.*`. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- `disableAllHooks: true` turns hooks off (managed hooks stay on unless it is set in managed settings). Matched hooks run in parallel, and identical duplicates in one file run once. `/hooks` views the configuration. — [Hooks reference](https://code.claude.com/docs/en/hooks); [Commands](https://code.claude.com/docs/en/commands)
- Env vars available to hooks: `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, `CLAUDE_EFFORT`, `CLAUDE_CODE_REMOTE` (`"true"` on the web), `CLAUDE_PLUGIN_OPTION_<KEY>`. `CLAUDE_ENV_FILE` (append `export X=Y` lines to persist env vars for later Bash calls) is available to SessionStart, Setup, CwdChanged and FileChanged hooks. — [Hooks reference](https://code.claude.com/docs/en/hooks)

**Common input (stdin JSON)**
- `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `prompt_id`, `permission_mode`, `scratchpad_dir` (v2.1.257+), `effort` (object with `level`), plus `agent_id`/`agent_type` inside subagents. Tool events add `tool_name`, `tool_input`, `tool_use_id`. — [Hooks reference](https://code.claude.com/docs/en/hooks)

**Exit codes / output**
- Exit 0 means success, and stdout is parsed as JSON if it is a single JSON object. Exit 2 is a blocking error on events that can block, with stderr as the reason. Any other code is a non-blocking error unless `onFailure: "block"` is set. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Universal JSON fields: `continue` (false stops Claude), `stopReason`, `suppressOutput`, `systemMessage` (warning shown to the user), `terminalSequence`. A top-level `decision`/`reason` is used by UserPromptSubmit, UserPromptExpansion, PostToolUse, PostToolUseFailure, PostToolBatch, Stop, SubagentStop, ConfigChange and PreCompact. `hookSpecificOutput` (requires `hookEventName`) carries `permissionDecision` (`allow`/`deny`), `permissionDecisionReason`, `updatedInput` (PreToolUse), `additionalContext`, and `retry` (PermissionDenied). — [Hooks reference](https://code.claude.com/docs/en/hooks)
- `additionalContext` is injected as a system reminder, capped at 10,000 chars per string. It is accepted on SessionStart, SubagentStart, UserPromptSubmit, UserPromptExpansion, PreToolUse, PostToolUse, PostToolUseFailure, PostToolBatch, Stop, SubagentStop and PostModelSwitch. — [Hooks reference](https://code.claude.com/docs/en/hooks)

**Full event list (matcher / exit-2 effect)** — all from [Hooks reference](https://code.claude.com/docs/en/hooks):
- `SessionStart`: matcher `startup|resume|clear|compact|fork`. Non-blocking. Plain stdout or `additionalContext` is added to context, and `sessionTitle` can be set. Input includes `source`, `model`, `agent_type`, `session_title`.
- `Setup`: fires on `--init-only`, or on `--init`/`--maintenance` with `-p`. Matcher `init|maintenance`.
- `UserPromptSubmit`: no matcher. Can block (`decision:"block"`). Can add `additionalContext`, `sessionTitle`, `suppressOriginalPrompt`. Input has `prompt`.
- `UserPromptExpansion`: a slash command expands. Matcher is the command name. Can block.
- `PreSkillShellCommand`: before a skill's `` !`cmd` `` injection runs. Can block.
- `PreToolUse`: matcher is the tool name. Can block, allow/deny via `permissionDecision`, or rewrite via `updatedInput`.
- `PermissionRequest` / `PermissionDenied` (auto mode; `retry: true`).
- `PostToolUse`, `PostToolUseFailure`: non-blocking. Stderr or `additionalContext` reaches Claude.
- `PostToolBatch`: after a parallel tool batch. Exit 2 stops the loop before the next model call.
- `Notification`: matcher `permission_prompt|idle_prompt|auth_success|...`.
- `MessageDisplay`, `SubagentStart`, `SubagentStop` (can block the subagent from stopping), `TaskCreated`, `TaskCompleted` (can block completion), `Stop`, `StopFailure` (matcher is the error type, e.g. `rate_limit`), `TeammateIdle` (keeps an agent-team teammate working).
- `InstructionsLoaded` (CLAUDE.md or `.claude/rules/*.md` loaded; matcher `session_start|nested_traversal|path_glob_match|include|compact`), `ConfigChange` (can block), `CwdChanged`, `DirectoryAdded`, `FileChanged` (matcher is literal filenames), `WorktreeCreate`, `WorktreeRemove`.
- `PreCompact` (matcher `manual|auto`; can block), `PostCompact`, `PreModelSwitch` (can block), `PostModelSwitch`, `Elicitation`, `ElicitationResult`.
- `SessionEnd`: matcher/reason is `clear|resume|logout|prompt_input_exit|other`. Not blocking.

**Stop / SubagentStop details (core of "keep working until X" loops)**
- Stop input adds `stop_hook_active` (true when Claude is already continuing because of a stop hook), `last_assistant_message`, `background_tasks` and `session_crons`. The docs advise using `last_assistant_message` rather than parsing `transcript_path`, because the transcript "isn't guaranteed to include the final message at Stop time on all versions." SubagentStop adds `agent_id`, `agent_type`, `agent_transcript_path`, `last_assistant_message`. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- `{"decision":"block","reason":"..."}` prevents stopping, and `reason` becomes Claude's next instruction. Exit 2 with stderr also works. `hookSpecificOutput.additionalContext` on Stop shows as non-error "Stop hook feedback". — [Hooks reference](https://code.claude.com/docs/en/hooks)
- Loop safety: check `stop_hook_active`. Claude Code also enforces an **8-consecutive-continuation cap**. After 8 blocks in a row it overrides the next block, the counter resets whenever Claude calls a tool, and `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` raises the cap. — [Hooks reference](https://code.claude.com/docs/en/hooks)

**Prompt and agent hooks (model-judged gates)**
- `type: "prompt"`: a single-turn model call. `$ARGUMENTS` is replaced with the hook input JSON. The model responds with `{"ok": true|false, "reason": "...", "impossible": true|false}`. On Stop, `ok:false` feeds `reason` back as Claude's next instruction. `impossible:true` lets the turn end. The default timeout is 30 s, `model` is optional, and `continueOnBlock` defaults to false. The docs give this example: `{"hooks":{"Stop":[{"hooks":[{"type":"prompt","prompt":"You are evaluating whether Claude should stop working. Context: $ARGUMENTS ... Respond with JSON: {\"ok\": true} ... or {\"ok\": false, \"reason\": \"...\"}","timeout":30}]}]}}`. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- `type: "agent"` (experimental): spawns a subagent that can use Read/Grep/Glob etc. for up to 50 turns, then returns `{ok, reason}`. The default timeout is 60 s. Docs example: `{"type":"agent","prompt":"Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS","timeout":120}` on `Stop`. The docs say "For production, prefer command hooks." — [Hooks reference](https://code.claude.com/docs/en/hooks)

**PreCompact / PostCompact / SessionEnd (memory-capture points)**
- PreCompact input: `trigger` (`manual`|`auto`) and `custom_instructions` (the `/compact` argument or null). On exit 0 without blocking, **stdout becomes compaction instructions** (keep it under 10,000 chars). Exit 2 or `decision:"block"` blocks compaction. Blocking auto-compaction that was triggered by a context-limit error makes the request fail. `systemMessage`/`continue` are discarded. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- PostCompact input: `trigger` and `compact_summary` (the generated summary). No decision control. A good point to persist the summary to a file. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- SessionEnd: `reason` ∈ `clear|resume|logout|prompt_input_exit|other` (`bypass_permissions_disabled` was removed in v2.1.234). It "can't block session termination", and its JSON output is discarded. **The default timeout is 1.5 seconds.** You can raise it with a per-hook `timeout` (overall budget up to 60 s) or `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS=5000`. On SIGTERM of `claude -p`, Claude Code runs SessionEnd hooks before exiting. — [Hooks reference](https://code.claude.com/docs/en/hooks); [Headless](https://code.claude.com/docs/en/headless)
- The sessions docs explicitly suggest: "A `SessionEnd` hook can archive the transcript when a session ends." — [Sessions](https://code.claude.com/docs/en/sessions)

**Async hooks**
- `"async": true` (command hooks only) lets Claude continue while the hook runs. The hook can't block or decide, and its `additionalContext`/`systemMessage` are delivered on the next turn. `asyncRewake`: an async hook that exits 2 wakes Claude immediately even when the session is idle, and `timeout` is enforced. In `-p` mode, async hooks still running at teardown are killed. — [Hooks reference](https://code.claude.com/docs/en/hooks)

**Docs-provided cost/feedback examples**
- PreToolUse `updatedInput` example that rewrites `npm test|pytest|go test` to `... 2>&1 | tail -n 100`. — [Costs](https://code.claude.com/docs/en/costs)
- The docs give a PostToolUse `additionalContext` example: "This file is generated. Edit src/schema.ts and run `bun generate` instead." — [Hooks reference](https://code.claude.com/docs/en/hooks)

### Inferences
- **Stop-hook test gate (sketch built from documented fields):** a command hook on `Stop` that reads stdin. If `stop_hook_active` is true, it exits 0 (or allows N retries through a counter file). Otherwise it runs `npm test`. On failure it prints `{"decision":"block","reason":"Tests failing:\n<last 50 lines>"}`. On success it exits 0. For a no-script version, use the documented `type:"agent"` Stop example.
- **SessionEnd summary writer:** because SessionEnd has a 1.5 s default budget and can't call the model synchronously in a useful way, the robust pattern is to set `timeout` (up to 60) or the env var, then fire-and-forget something like `nohup claude -p --resume "$session_id" "summarize learnings into docs/learnings/$(date +%F).md" &`. Alternatively, copy `transcript_path` to an archive and process it later in batch. This is an inference; the docs only confirm archiving the transcript.
- **PreCompact learnings saver:** a PreCompact command hook can (a) print compaction instructions such as "Preserve all decisions, failed approaches and user corrections" (stdout is used as instructions) and (b) copy `transcript_path` aside. A PostCompact hook can append `compact_summary` to a `LEARNINGS.md`.
- `/goal` (section 3) is literally a session-scoped prompt-based Stop hook, so a Stop hook in settings is the persistent equivalent.

### Gaps
- Could not see the full per-event `hookSpecificOutput` field list (the page is about 300 KB, and the middle section was skipped). Fields beyond those quoted above (e.g. for `TaskCompleted`, `TeammateIdle`) are unverified.
- The exact `asyncRewake` config syntax isn't shown in the fetched portion.
- No official worked "Stop hook runs tests" bash example was found. The docs' test example is the `agent`-type Stop hook.

---

## 2. Transcripts: location, format, analysis tools

### Takeaway
Transcripts are JSONL files at `~/.claude/projects/<project>/<session-id>.jsonl`, where `<project>` is the cwd path with non-alphanumerics replaced by `-`. They are kept 30 days by default (`cleanupPeriodDays`). Anthropic warns that the entry format is internal and changes between versions, and recommends `claude -p --resume <id> --output-format json`, `/export`, the hook `transcript_path`, or the Agent SDK instead. `/insights` is the built-in transcript analyzer. ccusage (cost) and claude-code-log (HTML rendering) are the main community tools.

### Cited Findings
- "By default, Claude Code stores transcripts as JSONL at `~/.claude/projects/<project>/<session-id>.jsonl`, where `<project>` is your working directory path with non-alphanumeric characters replaced by `-`." Names over 200 chars are truncated and suffixed with a hash. — [Sessions](https://code.claude.com/docs/en/sessions)
- "Each line is a JSON object: a message, a tool use or result, a record of a request Claude Code sent the model, or a metadata entry. The entry format is internal to Claude Code and changes between versions, so scripts that parse these files directly can break on any release. To build on session data, use `/export` or the script interfaces instead." — [Sessions](https://code.claude.com/docs/en/sessions)
- Config knobs: `CLAUDE_CONFIG_DIR` (move storage), `CLAUDE_CODE_PROJECT_DIR_NAME` (v2.1.234+, requires `CLAUDE_CONFIG_DIR`), `cleanupPeriodDays` (30-day default), `CLAUDE_CODE_SKIP_PROMPT_HISTORY`, `--no-session-persistence` (for `-p`), `claude purge`. — [Sessions](https://code.claude.com/docs/en/sessions)
- Script interfaces: `claude -p --output-format json|stream-json`; `claude -p --resume <session-id> --output-format json "summarize what we changed" | jq -r '.result'`; the hook `transcript_path`; the Agent SDK. `--resume` also accepts an absolute path to a `.jsonl` file. — [Sessions](https://code.claude.com/docs/en/sessions); [Headless](https://code.claude.com/docs/en/headless)
- `/export [filename]` writes plain text. On v2.1.327+ it refuses an existing filename. — [Commands](https://code.claude.com/docs/en/commands)
- Auto memory lives alongside the transcripts at `~/.claude/projects/<project>/memory/` and is excluded from the retention sweep. — [Memory](https://code.claude.com/docs/en/memory)
- **`/insights`**: "Generate an HTML report analyzing your recent sessions on this machine: which projects you work in, how you use Claude Code, where things go wrong, and features to try." It analyzes up to 200 not-yet-seen sessions per run, skips very short ones, and reports "friction points such as misunderstood requests or buggy code". The report is written to `~/.claude/usage-data/report.html` with timestamped copies, deleted after `cleanupPeriodDays`. Its tokens count against your plan. It is not available in cloud sessions. — [Costs](https://code.claude.com/docs/en/costs); [Commands](https://code.claude.com/docs/en/commands)
- `/team-onboarding`: "Generate a team onboarding guide from your Claude Code usage history... past 30 days." — [Commands](https://code.claude.com/docs/en/commands)
- `/usage` (aliases `/cost`, `/stats`) shows attribution to skills/subagents/plugins/MCP, recommendations such as "Trim ~/code/api/CLAUDE.md · 49k tokens per request", and per-`/loop` token rows. On v2.1.305+ pressing `f` applies fixable recommendations. — [Costs](https://code.claude.com/docs/en/costs)
- `/fewer-permission-prompts` (bundled skill): "Scan your transcripts for common read-only Bash and MCP tool calls, then add a prioritized allowlist to project `.claude/settings.json`". This is a built-in transcript-mining self-improvement loop. — [Commands](https://code.claude.com/docs/en/commands)
- `/doctor prompt-audit [path]` (v2.1.283+) audits CLAUDE.md, CLAUDE.local.md, AGENTS.md, rules, skills, commands, subagents and output styles for outdated or conflicting instructions. — [Memory](https://code.claude.com/docs/en/memory); [Commands](https://code.claude.com/docs/en/commands)
- Community: **claude-code-log** (daaain) is a Python CLI that converts transcript JSONL into HTML. Run with no arguments, it processes all of `~/.claude/projects/`, and it offers date filtering and a TUI. **ccusage** (ryoppippi) reports daily/monthly/session usage and cost. Others: ccost, ccstats (Rust, also reads Codex logs), claude-usage-analyzer ("why" usage is high). — [ecosyste.ms: claude-code-log](https://awesome.ecosyste.ms/projects/github.com%2Fdaaain%2Fclaude-code-log); [ccusage docs (mintlify)](https://www.mintlify.com/ryoppippi/ccusage/family/overview); [ccost](https://github.com/cc-friend/ccost); [claude-usage-analyzer](https://github.com/SingggggYee/claude-usage-analyzer)
- OpenTelemetry export "works on every setup and is the only option that streams per-user token and cost metrics into your own observability stack in near real time." — [Costs](https://code.claude.com/docs/en/costs) / [Monitoring](https://code.claude.com/docs/en/monitoring-usage.md)

### Inferences
- A "nightly session review" recipe should prefer `claude -p --resume <id> "extract learnings"` or `/insights` over hand-parsing JSONL, given Anthropic's explicit format-instability warning. If direct parsing is needed, do it defensively (filter by `type`) and pin the CLI version.
- `/insights` + `/fewer-permission-prompts` + `/doctor prompt-audit` + the `/usage` `f` fix together form a first-party "reflect on my history, then change my config" loop that needs no custom code.

### Gaps
- The exact JSONL field names (`uuid`, `parentUuid`, `type`, `message`, `toolUseResult`, `gitBranch`, etc.) are not documented officially (deliberately), so I could not verify them against docs. Community tools rely on them but they may change.
- I couldn't fetch the ccusage or claude-code-log READMEs directly (GitHub web blocked), so current flags and commands are unverified.

---

## 3. Scheduling & automation: headless, Agent SDK, GitHub Actions, /loop, /goal, cloud scheduled tasks

### Takeaway
There are five layers: (1) `claude -p` headless with JSON/stream-JSON output; (2) the Agent SDK (Python/TS); (3) in-session `/loop` (cron or self-paced, session-scoped, 30-day expiry) and `/goal` (a condition-based evaluator Stop hook); (4) cloud "scheduled tasks" (formerly "routines": schedule, API `/fire`, or GitHub-event triggers, min 1-hour interval, research preview, created via `/schedule` or claude.ai/code/routines) plus Desktop scheduled tasks; (5) GitHub Actions via `anthropics/claude-code-action@v1`.

### Cited Findings

**Headless (`claude -p`)**
- `-p`/`--print` runs non-interactively. Commonly paired flags: `--continue`, `--resume <id|name|path>`, `--allowedTools "Read,Edit,Bash"` (permission-rule syntax such as `Bash(git diff *)`), `--permission-mode auto|dontAsk|acceptEdits`, `--permission-prompts none` (v2.1.259+), `--output-format text|json|stream-json`, `--json-schema '<schema>'` (puts the result in `structured_output`), `--verbose --include-partial-messages` for token streaming, `--append-system-prompt[-file]`, `--system-prompt`, `--max-budget-usd`, `--max-turns`, `--mcp-config`, `--agents`, `--settings`, `--plugin-dir`, `--forward-subagent-text`. — [Headless](https://code.claude.com/docs/en/headless); [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- `--bare` skips auto-discovery of hooks, skills, commands, subagents, plugins, MCP, auto memory and CLAUDE.md. It requires `ANTHROPIC_API_KEY` (no OAuth), and hooks don't run even if passed via `--settings`. It "is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release." **Implication: a self-improvement loop that relies on hooks or CLAUDE.md must NOT use `--bare`.** — [Headless](https://code.claude.com/docs/en/headless)
- JSON output includes `result`, `session_id`, `total_cost_usd` and a per-model cost breakdown. stream-json emits `system/init` (model, tools, `plugins`, `plugin_errors`, `mcp_servers`, `mcp_server_errors`), `system/api_retry`, `stream_event`, and `result` messages. — [Headless](https://code.claude.com/docs/en/headless)
- Since v2.1.302, `claude -p` starts in Manual mode when nothing sets a permission mode, and calls that need approval are denied unless a `PermissionRequest` hook or `--permission-prompt-tool` approves them. — [Headless](https://code.claude.com/docs/en/headless)
- Without `--bare`, `-p` runs project `.claude/settings.json` hooks and `.mcp.json` servers "even in a folder you've never trusted" (no trust dialog). — [Headless](https://code.claude.com/docs/en/headless)
- Sessions started with `-p` or the SDK are hidden from the picker and `claude --continue`, but are resumable by ID. `claude -p --continue` includes them. — [Sessions](https://code.claude.com/docs/en/sessions)
- Skills and custom commands work in `-p` (`claude -p "/skill-name ..."`). — [Headless](https://code.claude.com/docs/en/headless)

**Agent SDK**
- "The Agent SDK gives you the same tools, agent loop, and context management that power Claude Code. It's available as a CLI for scripts and CI/CD, or as Python and TypeScript packages." The GitHub Action is built on the SDK. — [Headless](https://code.claude.com/docs/en/headless); [GitHub Actions](https://code.claude.com/docs/en/github-actions)

**`/loop` and session cron**
- `/loop [interval] [prompt]` is a bundled skill. `/loop 5m check the deploy` runs on a fixed cron. `/loop check the deploy` self-paces (Claude picks a 1 min to 1 h delay each iteration and can end the loop via `ScheduleWakeup` with `stop: true`). A bare `/loop` runs the built-in maintenance prompt (continue unfinished work → tend the PR's review comments, CI and conflicts → cleanup passes) or your `loop.md`. — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- `loop.md`: `.claude/loop.md` (project, takes precedence) or `~/.claude/loop.md` (user). Edits take effect next iteration, and the file is truncated past 25,000 bytes. — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- `/loop 20m /review-pr 1234` reruns a skill each iteration, but only skills Claude may invoke itself (not ones with `disable-model-invocation: true`). — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- Tools: `CronCreate` (5-field cron, prompt, recurring or once), `CronList`, `CronDelete` (8-char IDs). Max 50 tasks per session. Times are local. Recurring tasks expire **30 days** after creation (7 days before v2.1.330). Jitter is up to half the interval, max 30 min. `CLAUDE_CODE_DISABLE_CRON=1` disables the scheduler. Tasks fire only while the session is open and idle, are restored on `--resume` (except self-paced loops), and have no catch-up. — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- Comparison table: Cloud (no machine needed, fresh clone, min 1 h), Desktop (local, min 1 min when Claude sets it), `/loop` (open session required, min 1 min). — [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)

**`/goal`**
- `/goal <condition>` keeps Claude working across turns until a small fast model judges the condition met or impossible. The condition can be up to 4,000 chars. "`/goal` is a wrapper around a session-scoped prompt-based Stop hook." The evaluator doesn't run tools; it judges only what is in the conversation. Works with `-p`: `claude -p "/goal CHANGELOG.md has an entry for every PR merged this week"`. The evaluator model can be changed via `ANTHROPIC_DEFAULT_HAIKU_MODEL`. `/goal clear` clears it. Idle check-ins are controlled by `CLAUDE_CODE_GOAL_CHECKIN_MINUTES`. — [Goal](https://code.claude.com/docs/en/goal)
- The docs compare `/goal` (stops when the condition is met), `/loop` (time interval) and a Stop hook (your script decides; persists across sessions in settings). — [Goal](https://code.claude.com/docs/en/goal)

**Cloud scheduled tasks (formerly routines)**
- "A cloud scheduled task, formerly a routine, is a saved Claude Code configuration: a prompt, one or more repositories, and a set of connectors, packaged once and run automatically" on Anthropic cloud (or a self-hosted environment). Triggers: **Schedule** (presets or custom cron; "Expressions that run more often than once per hour are rejected"; one-offs allowed), **API** (POST to `https://api.anthropic.com/v1/claude_code/routines/<trig_id>/fire` with a bearer token and an optional `text` field; beta header `experimental-cc-routine-2026-04-01`), and **GitHub** events (e.g. `pull_request.opened`, filters on base/head branch and labels; hourly caps). The feature is in research preview. Plans: Pro, Max, Ultra, Team, Enterprise. Create at claude.ai/code/routines or via `/schedule` (alias `/routines`), e.g. `/schedule daily PR review at 9am`. Runs default to auto mode, and Claude pushes to `claude/`-prefixed branches. — [Routines / cloud scheduled tasks](https://code.claude.com/docs/en/routines); [Scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks); [Commands](https://code.claude.com/docs/en/commands)
- `/autofix-pr [prompt]` spawns a cloud session that watches the branch's PR and pushes fixes for CI failures and review comments. — [Commands](https://code.claude.com/docs/en/commands)

**GitHub Actions (`anthropics/claude-code-action@v1`)**
- Quick setup is `/install-github-app` (github.com only). It installs the Claude GitHub App and writes `claude.yml` (plus an optional `claude-code-review.yml`) and a secret (`ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`). — [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- Without a `prompt` input the action runs in interactive mode (responds to `@claude`, or a custom `trigger_phrase`). With `prompt` it runs in automation mode on any event, including `schedule`. Inputs: `prompt`, `claude_args`, `anthropic_api_key`, `claude_code_oauth_token`, `github_token`, `plugin_marketplaces`, `plugins`, `settings`, `trigger_phrase`, `use_bedrock`/`use_vertex`/`use_foundry`. Bots are rejected unless listed in `allowed_bots` ("keeps bots from triggering Claude in a loop"). Migration from beta: `direct_prompt`→`prompt`, `mode` removed, `max_turns`/`model` move into `claude_args`, `custom_instructions`→`--append-system-prompt`. — [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- Docs' scheduled example: `on: schedule: - cron: "0 9 * * *"` with `prompt: "Generate a summary of yesterday's commits and open issues"` and `claude_args: --model ... --allowedTools "mcp__github__list_commits,mcp__github__list_issues"`. A plugin-skill example: `plugin_marketplaces: "https://github.com/anthropics/claude-plugins-official.git"`, `plugins: "code-review@claude-plugins-official"`, `prompt: "/code-review:code-review --comment ..."`. GitHub disables schedules in public repos after 60 days of inactivity. — [GitHub Actions](https://code.claude.com/docs/en/github-actions)

### Inferences
- The simplest "weekly self-improvement" automation options: (a) a cloud scheduled task with prompt "Review merged PRs and review comments from the last week; propose CLAUDE.md/rules updates as a PR"; (b) the same as a GitHub Actions cron job with `prompt`; (c) a local cron/launchd job running `claude -p "/reflect" --permission-mode acceptEdits` (non-bare so hooks and CLAUDE.md load).
- For "keep going until done" inside a session, `/goal` replaces a hand-rolled Ralph Stop hook for most users. A Stop hook remains the persistent, scriptable option.

### Gaps
- I didn't fetch Desktop scheduled-task docs in detail, or the Agent SDK pages (hooks in SDK, `canUseTool`, session APIs).
- I didn't verify the cloud scheduled-task per-account run caps (the docs mention "hourly limit" and per-task/per-account GitHub caps without numbers in the excerpt).

---

## 4. Skills, slash commands, subagents, plugins, output styles, CLAUDE.md hierarchy, auto memory, /memory

### Takeaway
Memory has two tracks: human-written CLAUDE.md (hierarchical, `@imports` up to 4 hops, `.claude/rules/` with `paths:` scoping, can read AGENTS.md since v2.1.277) and Claude-written **auto memory** (`~/.claude/projects/<project>/memory/MEMORY.md` index + topic files; first 200 lines/25 KB loaded; on by default; `autoMemoryEnabled`). Subagents can have their own persistent memory via `memory: user|project|local`. Skills (`SKILL.md`) have absorbed custom commands, support `` !`cmd` `` context injection and `context: fork`, and hot-reload. Plugins bundle skills, agents, hooks and MCP, and are distributed via marketplaces (`/plugin marketplace add owner/repo`).

### Cited Findings

**CLAUDE.md hierarchy**
- Locations: managed policy (`/Library/Application Support/ClaudeCode/CLAUDE.md`, `/etc/claude-code/CLAUDE.md`, `C:\Program Files\ClaudeCode\CLAUDE.md`, or a `claudeMd` key in managed-settings.json); user `~/.claude/CLAUDE.md`; project `./CLAUDE.md` or `./.claude/CLAUDE.md`; local `./CLAUDE.local.md` (gitignore it). Files above the cwd load at launch and are concatenated root→cwd, with CLAUDE.local.md after CLAUDE.md at each level. Subdirectory CLAUDE.md files load on demand when Claude touches files there. — [Memory](https://code.claude.com/docs/en/memory)
- Imports: `@path/to/import`, relative to the importing file, "maximum depth of four hops". Escape spaces with a backslash, and paths inside code spans are not imported. External imports prompt for a one-time approval. — [Memory](https://code.claude.com/docs/en/memory)
- Guidance: "target under 200 lines per CLAUDE.md file". Multi-step procedures should move to a skill or path-scoped rule. — [Memory](https://code.claude.com/docs/en/memory)
- `.claude/rules/*.md` (recursive, symlinks allowed) with optional `paths:` frontmatter globs, so a rule loads only when matching files are touched. User-level `~/.claude/rules/` is also supported. `claudeMdExcludes` skips files by glob. — [Memory](https://code.claude.com/docs/en/memory)
- AGENTS.md: since v2.1.277, Claude Code reads `AGENTS.md` when there is no CLAUDE.md/CLAUDE.local.md on the path. The **Project instructions** setting takes `claude-md-or-agents-md` (default), `claude-md-and-agents-md`, `claude-md` or `managed-only`, set via `/config` or `pluginConfigs["cc-plugin-agents-md@builtin"].options.instructionFiles`. A shared-file pattern is a CLAUDE.md containing `@AGENTS.md`. `/import [codex|gemini|cursor]` copies other agents' config. — [Memory](https://code.claude.com/docs/en/memory); [Commands](https://code.claude.com/docs/en/commands)
- CLAUDE.md is delivered as a user message after the system prompt, with "no guarantee of strict compliance". The docs point to hooks for things that must happen, and to the `InstructionsLoaded` hook for debugging. — [Memory](https://code.claude.com/docs/en/memory)
- `/init` generates a CLAUDE.md. `CLAUDE_CODE_NEW_INIT=1` adds an interactive flow for skills, hooks and personal memory. — [Commands](https://code.claude.com/docs/en/commands)

**Auto memory (built-in self-improvement)**
- "Auto memory lets Claude accumulate knowledge across sessions without you writing anything." Four note `type`s go in frontmatter: `user`, `feedback` ("corrections you give Claude and approaches you confirm"), `project`, `reference`. It skips anything derivable from code or already in CLAUDE.md. — [Memory](https://code.claude.com/docs/en/memory)
- Storage: `~/.claude/projects/<project>/memory/`, with `MEMORY.md` as the index ("one line per memory, loaded at session start") plus topic files read on demand. All worktrees of a repo share it, and it is machine-local. "the first 200 lines or 25KB of `MEMORY.md`, whichever comes first, load at session start". Claude Code warns or errors when the index exceeds limits, and adds a `modified` ISO timestamp in frontmatter. — [Memory](https://code.claude.com/docs/en/memory)
- Toggle via `/memory` (saves `autoMemoryEnabled` in `~/.claude/settings.json`), per-project `"autoMemoryEnabled": false`, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. Relocate with `autoMemoryDirectory`. It is on by default locally and off by default in self-hosted environments. — [Memory](https://code.claude.com/docs/en/memory)
- "When you ask Claude to remember something... Claude saves it to auto memory. To add instructions to CLAUDE.md instead, ask Claude directly, like 'add this to CLAUDE.md'." `/memory` lists and opens all memory files and the auto-memory folder. — [Memory](https://code.claude.com/docs/en/memory)

**Subagents**
- Defined in `.claude/agents/*.md` (project), `~/.claude/agents/` (user), the `--agents` JSON flag, plugin `agents/`, or managed settings. Directories are scanned recursively. Frontmatter: `name`, `description` (required), `tools`, `disallowedTools`, `model` (`sonnet|opus|haiku|fable|inherit|<id>`), `permissionMode`, `maxTurns`, `autoCompactWindow`, `skills`, `mcpServers`, `hooks`, `memory`, `background`, `omitClaudeMd`, `effort`, `isolation: worktree`, `color`, `initialPrompt`, `experimental`. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- **Persistent subagent memory:** `memory: user` → `~/.claude/agent-memory/<agent>/`; `project` → `.claude/agent-memory/<agent>/` (recommended, VCS-shareable); `local` → `.claude/agent-memory-local/<agent>/`. The subagent's prompt gets the first 200 lines/25 KB of its `MEMORY.md` plus instructions to curate it, and Read/Write/Edit are auto-enabled. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- `/agents` now just reminds you to ask Claude, or to edit `.claude/agents/` directly (the interactive UI ended at v2.1.197). — [Commands](https://code.claude.com/docs/en/commands)

**Skills and slash commands**
- Locations: `~/.claude/skills/<name>/SKILL.md`, `.claude/skills/<name>/SKILL.md`, nested `<subdir>/.claude/skills/`, plugin `skills/` (invoked `/plugin:skill`), enterprise. Frontmatter: `name`, `description`, `when_to_use`, `argument-hint`, `arguments`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `disallowed-tools`, `model`, `effort`, `context: fork`, `agent`, `background`, `hooks`, `paths`, `shell`. — [Skills](https://code.claude.com/docs/en/skills)
- `` !`git diff HEAD` `` injects command output before the prompt is sent (fenced ` ```! ` for multi-line). Placeholders: `$ARGUMENTS`, `$0`/`$ARGUMENTS[N]`, `$name`, `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PROJECT_DIR}`. — [Skills](https://code.claude.com/docs/en/skills)
- "Custom commands have been merged into skills. Your existing `.claude/commands/` files keep working." Skills hot-reload (except in bare mode), and `/reload-skills` forces a reload. — [Skills](https://code.claude.com/docs/en/skills)
- Bundled skills/commands relevant to improvement loops: `/loop`, `/batch` (5–30 parallel worktree subagents → PRs), `/simplify` (four parallel review agents), `/code-review` (alias `/review`; `--fix`, `--comment`, `ultra`), `/verify` and `/run` (build and drive the app), `/run-skill-generator` ("Teach `/run` and `/verify` how to build and launch your project"), `/skill-doctor` (per-skill context cost and usage frequency), `/doctor` (alias `/checkup`), `/debug`, `/fewer-permission-prompts`, `/claude-api ... prompt-audit|build-eval|hillclimb`, `/recap`, `/insights`, `/team-onboarding`. — [Skills](https://code.claude.com/docs/en/skills); [Commands](https://code.claude.com/docs/en/commands)
- Skills from MCP servers aren't supported in `-p`. — [Headless](https://code.claude.com/docs/en/headless)

**Plugins / marketplaces / output styles**
- Plugin skills are namespaced `/plugin-name:skill-name`. Install from a marketplace with `/plugin marketplace add <owner/repo>` then `/plugin install <name>` (example: compound-engineering). `/plugin [list|install|enable|disable]`, `/reload-plugins`. The official marketplace is `anthropics/claude-plugins-official`. — [Commands](https://code.claude.com/docs/en/commands); [GitHub Actions](https://code.claude.com/docs/en/github-actions); [Compound Engineering README](https://raw.githubusercontent.com/EveryInc/compound-engineering-plugin/HEAD/README.md)
- Plugin hooks live in `hooks/hooks.json`, and plugins get `CLAUDE_PLUGIN_ROOT`/`CLAUDE_PLUGIN_DATA` (persistent data dir). — [Hooks reference](https://code.claude.com/docs/en/hooks)
- `/output-style [style]` lists or switches output styles (v2.1.269+). It is also usable in `-p`. — [Commands](https://code.claude.com/docs/en/commands); [Headless](https://code.claude.com/docs/en/headless)
- `/plugin-authoring` loads reference material for writing "mods" (v2.1.287+). — [Commands](https://code.claude.com/docs/en/commands)

### Inferences
- Built-in self-improvement now spans auto memory (automatic `feedback`-type notes from corrections), `/insights` (friction analysis), `/fewer-permission-prompts` (settings tuning from transcripts), `/doctor prompt-audit` (instruction hygiene), `/skill-doctor` (skill pruning), `/run-skill-generator`, and `/claude-api hillclimb|build-eval` (eval-driven prompt iteration). A writer can present these as "zero-config" loops before custom hooks.
- The Anthropic `skill-creator` skill (create, test and iterate on skills with evals) appears in this environment as `anthropic-skills:skill-creator`. It is distributed via Anthropic's skills/plugins, not as a bundled CLI command (observation from the session's skill listing; not verified in docs).

### Gaps
- Didn't fetch the plugins/marketplace reference (`.claude-plugin/plugin.json`, `marketplace.json` schema) or the output-styles page. Field names for those are unverified here.
- Not verified: whether `skill-creator` is bundled in the CLI or only available via `anthropics/skills` / the official marketplace.

---

## 5. Equivalents in Codex CLI, Cursor, Gemini CLI, Aider

### Takeaway
All of the major harnesses have converged on Claude-Code-like primitives: an instructions file (AGENTS.md, GEMINI.md, `.cursor/rules/*.mdc`), JSON-on-stdin lifecycle hooks with exit-2 blocking, and a memory feature. Codex hooks mirror Claude Code's event names (SessionStart, PreToolUse, PostToolUse, UserPromptSubmit, PreCompact, PostCompact, Stop, SubagentStart/Stop). Cursor uses camelCase events, and its `stop` hook can auto-submit a `followup_message` bounded by `loop_limit`. Gemini CLI has BeforeAgent/AfterAgent (retry), BeforeTool/AfterTool, PreCompress, SessionStart/End. Cursor also has cloud-agent "Automations" (schedule or event triggers).

### Cited Findings

**OpenAI Codex CLI** (official docs unreachable from the sandbox; based on search snippets of developers.openai.com)
- Hooks are read from `~/.codex/hooks.json`, `~/.codex/config.toml`, `<repo>/.codex/hooks.json` and `<repo>/.codex/config.toml` (inline `[[hooks.SessionStart]]` / `[[hooks.SessionStart.hooks]]` tables). All sources load. Hooks are on by default and disabled via `[features] hooks = false` (legacy alias `codex_hooks`). `/hooks` reviews and trusts hooks. — [Codex hooks](https://developers.openai.com/codex/hooks) (via search snippet)
- Events: PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, UserPromptSubmit, SubagentStop, Stop (turn scope); SessionStart, SubagentStart (thread scope). SessionEnd is mentioned in a localized page only. Stop includes `stop_hook_active`, and the docs include a `stop_continue.py` example (continuation semantics unverified). — [Codex hooks](https://developers.openai.com/codex/hooks); [Codex hooks (es)](https://developers.openai.com/es-419/docs/hooks) (snippets)
- AGENTS.md discovery: Codex home (`~/.codex`, or `CODEX_HOME`) is checked first, where `AGENTS.override.md` beats `AGENTS.md`. Then, per directory from the repo root down: `AGENTS.override.md` → `AGENTS.md` → fallback names (e.g. `TEAM_GUIDE.md`, `.agents.md`). — [Codex AGENTS.md guide](https://developers.openai.com/codex/guides/agents-md) (snippet)
- Memories: enabled via `[features] memories = true` in `~/.codex/config.toml`, with `memories.use_memories` controlling injection into future sessions. A third party reports that summaries are stored in `~/.codex/memories/`. — [Codex memories](https://developers.openai.com/codex/memories.md) (snippet); [mem0 write-up](https://mem0.ai/blog/how-memory-works-in-codex-cli)
- Plugin marketplace support: `codex plugin marketplace add EveryInc/compound-engineering-plugin` then `codex plugin add compound-engineering@compound-engineering-plugin`. — [Compound Engineering README](https://raw.githubusercontent.com/EveryInc/compound-engineering-plugin/HEAD/README.md)
- A reported bug: Codex CLI 0.77.0 didn't load `~/.codex/AGENTS.md` by default. — [openai/codex#8759](https://github.com/openai/codex/issues/8759)

**Cursor** (official docs unreachable; snippets plus forum posts)
- Hooks: `.cursor/hooks.json` (project) or `~/.cursor/hooks.json` (global), format `{"version": 1, "hooks": {"<event>": [{"command": "...", "matcher": "...", "timeout": 30}]}}`. A `"type": "prompt"` entry is supported. Events include beforeSubmitPrompt, beforeShellExecution, afterFileEdit, preCompact, afterAgentResponse, afterAgentThought, stop, plus sessionStart, sessionEnd, subagentStop, preToolUse, postToolUse. — [Cursor hooks docs](https://cursor.com/docs/agent/hooks) (snippet); [GitButler deep dive](https://blog.gitbutler.com/cursor-hooks-deep-dive)
- `stop` is the hook that can continue the agent: output `{"followup_message": "..."}` is auto-submitted as a new user turn. It is bounded by `loop_limit`, and the script receives `loop_count`. Forum users report flakiness (Windows; v2.4; cloud agents don't run `stop`/`afterAgentResponse` hooks). — [Cursor forum: stop hook](https://forum.cursor.com/t/let-a-stop-hook-run-end-of-turn-work-without-rendering-below-the-final-answer/165472); [Cursor forum: cloud agents](https://forum.cursor.com/t/cursor-cloud-agents-do-not-run-afteragentresponse-or-stop-hooks/159929); [Cursor forum: Windows](https://forum.cursor.com/t/stop-hook-followup-message-not-captured-on-windows-execution-log-shows-despite-valid-json-on-stdout/155078)
- Rules: `.cursor/rules/*.mdc` with `description`, `globs`, `alwaysApply` frontmatter. AGENTS.md is read for plain markdown, and `.cursorrules` is legacy. Forum reports of `alwaysApply` regressions in 3.0.16. — [Cursor rules docs](https://cursor.com/docs/rules) (snippet); [morphllm guide](https://www.morphllm.com/cursor-rules-best-practices); [forum](https://forum.cursor.com/t/alwaysapply-true-rules-and-cursorrules-both-silently-treated-as-requestable-instead-of-auto-injected-cursor-3-0-16-macos/157431)
- Memories: current docs describe memories as persistent named notes (`MEMORIES.md` by default), enabled by default. The 3.8 changelog (06-18-26) adds deleting memory files in the UI and letting an automation prune outdated memories. — [Cursor memories docs](https://cursor.com/docs/context/memories) (snippet); [Cursor 3.8 changelog](https://cursor.com/changelog/06-18-26) (snippet)
- Automations (launched around 03-05-26) run cloud agents (formerly "background agents") on a schedule (presets/cron) or on GitHub, GitLab, Slack, webhook or Linear events. — [Cursor Automations docs](https://cursor.com/docs/cloud-agent/automations); [Cursor blog](https://cursor.com/blog/automations); [changelog 03-05-26](https://cursor.com/changelog/03-05-26) (snippets)
- Plugin install: `/add-plugin compound-engineering`. — [Compound Engineering README](https://raw.githubusercontent.com/EveryInc/compound-engineering-plugin/HEAD/README.md)

**Gemini CLI** (fetched from the official repo's raw docs)
- Events: `SessionStart` (inject context), `SessionEnd` (advisory), `BeforeAgent` (block the turn or add context), `AfterAgent` ("Retry / Halt"), `BeforeModel` (block or mock the response), `AfterModel` (block or redact), `BeforeToolSelection` (filter tools), `BeforeTool` (block or rewrite args), `AfterTool` (block the result or add context), `PreCompress` (advisory), `Notification` (advisory). — [Gemini CLI hooks docs](https://raw.githubusercontent.com/google-gemini/gemini-cli/main/docs/hooks/index.md)
- Config: the `hooks` object in settings.json, merged from `.gemini/settings.json` > `~/.gemini/settings.json` > `/etc/gemini-cli/settings.json` > extensions. Entries have `matcher` (regex for tool events, exact for lifecycle events) and a `hooks` array of `{type:"command", command, name, description, timeout}`. **`timeout` is in milliseconds** (default 60000). Exit 0 means stdout is JSON, exit 2 blocks with the stderr reason. Stdout must be pure JSON. `/hooks panel|enable-all|disable-all|enable <name>|disable <name>`. Project hooks are fingerprinted. — [Gemini CLI hooks docs](https://raw.githubusercontent.com/google-gemini/gemini-cli/main/docs/hooks/index.md)
- Hooks are enabled by default as of v0.26.0+ per the Google developers blog. — [Google Developers Blog](https://developers.googleblog.com/tailor-gemini-cli-to-your-workflow-with-hooks/) (snippet)

**Aider**
- No sources gathered this session (budget and network limits). From prior knowledge (UNVERIFIED): Aider uses `CONVENTIONS.md` loaded via `--read` or `.aider.conf.yml`, has `--auto-test`/`--test-cmd` and `--auto-lint`/`--lint-cmd` loops that feed failures back to the model, supports scripting via `aider --message "..."` / `--yes-always`, and has no lifecycle-hook system comparable to the others.

### Inferences
- The cross-harness "portable self-improvement" pattern is: keep one `AGENTS.md` (Claude Code reads it natively since v2.1.277, Codex and Cursor read it), plus per-harness Stop-equivalent hooks (Claude `Stop` → `decision:block`, Codex `Stop`, Cursor `stop` → `followup_message`, Gemini `AfterAgent` → retry).
- Gemini's `timeout` is in milliseconds while Claude Code's is in seconds. That is an easy porting error to flag in setup sketches.

### Gaps
- Codex Stop-hook output schema, Codex `SessionEnd` existence, and Codex memory file layout are not verified against fetched docs (developers.openai.com is blocked in the sandbox).
- The Cursor hook event list and the `stop` return schema are not verified against official docs. Forum evidence only.
- Gemini `AfterAgent` exact JSON for retry, GEMINI.md hierarchy, `/memory add`, `save_memory` tool and extension manifest (`gemini-extension.json`) were not fetched this session. From prior knowledge (UNVERIFIED): GEMINI.md is loaded hierarchically from `~/.gemini/GEMINI.md` and project dirs, `/memory add|show|refresh` exist, and the `save_memory` tool appends to `~/.gemini/GEMINI.md`.
- Aider is entirely unverified.

---

## 6. Community examples and recipes

### Takeaway
The best-documented community loops are: (1) **Ralph** (Geoffrey Huntley's `while true` bash loop feeding PROMPT.md into a fresh agent each iteration, later formalized as Anthropic's `ralph-loop` plugin using a Stop hook with `--max-iterations`/`--completion-promise`); (2) **Compound Engineering** (Every / Kieran Klaassen: `/ce-brainstorm → /ce-plan → /ce-work → /ce-compound`, with learnings written to `docs/solutions/`); (3) Boris Cherny's team habit of adding to a shared CLAUDE.md whenever Claude errs, often via `@claude` on PRs through the GitHub Action; (4) `/reflect`-style plugins (claude-reflect) that queue corrections via hooks and apply them to CLAUDE.md on demand; (5) hook collections (disler/claude-code-hooks-mastery).

### Cited Findings

**Ralph loop**
- Huntley: "Ralph is a Bash loop." It was first written up July 14, 2025 ("Ralph Wiggum as a 'software engineer'"). Each iteration gets a fresh context, and memory persists only via files (git history, progress files, PRD). A typical form is `while :; do cat PROMPT.md | claude -p ...; done` or `claude -p "$(cat PROMPT.md)"` with an iteration cap. — [DreamHost: Ralph from first principles](https://www.dreamhost.com/blog/ralph-wiggum/); [Steve Kinney](https://stevekinney.com/writing/the-ralph-loop); [The Register, 2026-01-27](https://www.theregister.com/2026/01/27/ralph_wiggum_claude_loops/); [Ralph Playbook](https://claytonfarr.github.io/ralph-playbook)
- Anthropic's official plugin (released December 2025 per secondary sources) is in `anthropics/claude-plugins-official/plugins/ralph-loop`. Commands: `/ralph-loop`, `/cancel-ralph`. Options: `--max-iterations <n>` ("Stop after N iterations (default: unlimited)") and `--completion-promise <text>` ("Phrase that signals completion"). `hooks/stop-hook.sh` "creates the self-referential feedback loop by blocking normal session exit". Quick-start example: `... --completion-promise "COMPLETE" --max-iterations 50`. The README recommends always setting `--max-iterations`. — [ralph-loop README](https://raw.githubusercontent.com/anthropics/claude-plugins-official/main/plugins/ralph-loop/README.md)
- Secondary: the plugin keeps state in `.claude/ralph-loop.local.md`. Some practitioners prefer the raw bash loop because it gives fresh context each iteration, whereas the plugin keeps one session. — search snippets of [awesomeclaude.ai](https://awesomeclaude-ai.nuxt.dev/ralph-wiggum); [leanware](https://leanware.co/insights/ralph-wiggum-ai-coding) (state-file name not confirmed by the README)

**Compound Engineering (Every.to, Kieran Klaassen)**
- The loop is "brainstorm, plan, build, review, then **capture what you learned**." Commands: `/ce-brainstorm`, `/ce-plan`, `/ce-work`, `/ce-compound`. "Capture the learning into `docs/solutions/` so the next loop starts smarter." Philosophy: "Each unit of engineering work should make subsequent units easier -- not harder." — [EveryInc/compound-engineering-plugin README](https://raw.githubusercontent.com/EveryInc/compound-engineering-plugin/HEAD/README.md)
- Install: Claude Code `/plugin marketplace add EveryInc/compound-engineering-plugin` + `/plugin install compound-engineering`; Cursor `/add-plugin compound-engineering`; Codex `codex plugin marketplace add ...` + `codex plugin add compound-engineering@compound-engineering-plugin`. Older posts use `/workflows:plan`, `/workflows:compound` naming, since renamed. Every says 36 skills, maintained by Kieran Klaassen and Trevin Chow. — [README](https://raw.githubusercontent.com/EveryInc/compound-engineering-plugin/HEAD/README.md); [Every install page](https://every.to/compound-engineering/install/); [Every guide](https://every.to/guides/compound-engineering)
- Long-form sources: [Compound engineering: the definitive guide](https://every.to/p/compound-engineering-the-definitive-guide); [Compound Engineering Camp](https://every.to/source-code/compound-engineering-camp-every-step-from-scratch); [AI Engineer talk "The Era of Compound Engineering"](https://ai.engineer/talks/the-era-of-compound-engineering)

**Boris Cherny (Claude Code creator) tips** (secondary summaries of an X thread commonly dated Jan 31, 2026; original not fetched)
- The team shares one checked-in CLAUDE.md that everyone contributes to several times a week: "Anytime Claude does something incorrectly, they add it to the CLAUDE.md so it knows not to do it next time." In code review he tags `@claude` on coworkers' PRs "to add learnings to the CLAUDE.md file as part of the PR", using the Claude Code GitHub Action (installed via `/install-github-action` per one source; the official command is `/install-github-app`). He reportedly said their CLAUDE.md is about 2.5k tokens. — [InfoQ, 2026-01](https://www.infoq.com/news/2026/01/claude-code-creator-workflow); [Push to Prod](https://getpushtoprod.substack.com/p/how-the-creator-of-claude-code-actually); [gist summary](https://gist.github.com/changtimwu/48a6daaa6d8343174a5b8a2eab60a70d)

**"How Anthropic teams use Claude Code"**
- Product Design team: "set up autonomous loops where Claude Code writes the code for the new feature, runs tests, and iterates continuously". They also "automated Pull Request comments through GitHub Actions". Security Engineering moved to TDD with Claude. Knowledge "consolidates... via MCP and CLAUDE.md files". Growth Marketing uses two sub-agents over ad-performance CSVs. — [claude.com blog](https://claude.com/blog/how-anthropic-teams-use-claude-code); full PDF at [www-cdn.anthropic.com](https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf) (not fetchable here)
- Data Infrastructure: new data scientists use Claude Code to navigate the codebase, and it "reads their Claude.md files". — search snippet of the [PDF](https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf)
- Anthropic engineering best-practices post: CLAUDE.md files are sometimes run through a prompt improver and tuned with emphasis ("IMPORTANT", "YOU MUST"). — [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) (snippet)

**/reflect and session-review tools**
- **claude-reflect** (BayramAnnakov): hooks detect correction patterns and queue them to `~/.claude/learnings-queue.json`. The user runs `/reflect` to review and apply them to CLAUDE.md, and `/reflect --scan-history` mines past sessions for missed learnings. Routing: global → `~/.claude/CLAUDE.md`, project → `./CLAUDE.md`, personal → `./CLAUDE.local.md`. Flags include `--review` and `--dedupe`. — search snippets of [claude-reflect SKILL.md](https://cdn.jsdelivr.net/gh/BayramAnnakov/claude-reflect@main/SKILL.md); [Smithery listing](https://smithery.ai/skills/BayramAnnakov/claude-reflect)
- SpecWeave `reflect`: a Stop hook analyzes the transcript and updates CLAUDE.md, capped by `maxLearningsPerSession`. — [skills.sh: specweave/reflect](https://skills.sh/anton-abyzov/specweave/reflect) (snippet)
- Reflection prompt gists ("reflect on the work session and extract learnings"; "knowledge curator... keep CLAUDE.md lean"). — [martinschenk gist](https://gist.github.com/martinschenk/ad7f287ddc28069f0047b61d7856cfa0); [a-c-m gist](https://gist.github.com/a-c-m/f4cead5ca125d2eaad073dfd71efbcfc); [feiskyer reflection skill](https://skills.sh/feiskyer/claude-code-settings/reflection)

**Hook collections**
- disler/claude-code-hooks-mastery: a community reference implementation covering all lifecycle hooks (uv-run Python scripts such as `.claude/hooks/user_prompt_submit.py`). It is listed on awesome-claude-code, with star counts reported between about 1.2k and 3.6k at different snapshots. Companion: disler/claude-code-hooks-multi-agent-observability. — [ecosyste.ms entry](https://awesome.ecosyste.ms/projects/github.com%2Fdisler%2Fclaude-code-hooks-mastery); [sourcepulse](https://www.sourcepulse.org/projects/10592574)
- Others: [karanb192/claude-code-hooks](https://raw.githubusercontent.com/karanb192/claude-code-hooks/HEAD/README.md), [shanraisshan/claude-code-hooks](https://github.com/shanraisshan/claude-code-hooks); cross-harness bridge [agent-hooks-bridge](https://pivanov.github.io/agent-hooks-bridge/adapters/cursor.html). — search results

### Inferences
- Short setup sketches per pattern:
  - *Ralph (fresh context):* `while :; do claude -p "$(cat PROMPT.md)" --permission-mode acceptEdits; done`, with an iteration counter and a `grep -q DONE progress.md && break` exit. *Ralph (in-session):* `/plugin install ralph-loop@claude-plugins-official`, then `/ralph-loop "<task>" --completion-promise "COMPLETE" --max-iterations 50`. *Native:* `/goal <condition>`.
  - *Compound:* install the plugin, run `/ce-plan` → `/ce-work` → `/ce-compound` per feature, and make sure CLAUDE.md points at `docs/solutions/` (or the plan step greps it).
  - *Boris-style:* `/install-github-app`, then comment `@claude add a rule to CLAUDE.md so this doesn't happen again` on PRs.
  - *Reflect:* a `.claude/skills/reflect/SKILL.md` with `disable-model-invocation: true`, a body that says "review this session's corrections; propose diffs to CLAUDE.md/rules; keep it under 200 lines", and optionally `` !`git diff` ``. Trigger it manually, from a SessionEnd hook via background `claude -p --resume`, or weekly via `/schedule`.
- Built-in auto memory (with `feedback` notes) now overlaps substantially with claude-reflect's correction capture. Write-ups should note this to avoid double-writing learnings.

### Gaps
- Couldn't fetch the original Huntley post, the Boris X thread, the HN threads, or the Anthropic PDF text. From prior knowledge (UNVERIFIED; confirm before quoting): the PDF describes the Data Science/ML team's "slot machine" approach (commit, let Claude run about 30 min, accept or restart), RL Engineering's checkpoint-heavy workflow, and teams asking Claude at session end to summarize work and suggest CLAUDE.md improvements.
- I didn't verify the GitHub star counts or current maintenance status of the community repos.
- I found no authoritative list of "widely shared hook collections" beyond the awesome-list mentions.
