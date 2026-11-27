# Memory- and Context-Based Self-Improvement Loops for AI Agents

Scope: ways an agent improves across sessions by distilling lessons into persistent context (prompts, instruction files, memory stores) without changing weights. Research date: 2026-11-27.

Source-access note: arxiv.org, huggingface.co, docs.devin.ai, dev.to and cdn.jsdelivr.net could not be reached from this research session (network allowlist). Anthropic docs and engineering pages, code.claude.com, platform.claude.com and github.com were fetched directly. Research-paper summaries marked **[abstract-level, not fetched]** come from prior knowledge of the paper's abstract and well-known results. They cite the canonical arXiv URL, but the report writer should treat exact numbers in those items as unverified in this session.

---

## Q1. Practitioner patterns: instruction files, product memory features, progress files, team-shared rules

### Takeaway
Practitioners have settled on two layers. The first is a **human-curated, version-controlled instruction file** (CLAUDE.md / AGENTS.md / rules dirs), grown by the rule "when the agent makes the same mistake twice or review catches something, add a line." The second is an **agent-written memory layer** (Claude Code auto memory, the Claude API memory tool, ChatGPT memory, Windsurf/Cascade memories) for preferences and facts that can't be derived from the code. For long-running work, **structured progress artifacts** (progress log + JSON feature list + git) act as cross-session memory. Vendors increasingly push durable knowledge back into reviewable files (rules, AGENTS.md, skills) and away from opaque auto-memories. Cursor removed its Memories feature, and Windsurf calls its memories legacy.

### Cited Findings

**Claude Code: CLAUDE.md (human-written layer)**
- Claude Code docs: "Each Claude Code session begins with a fresh context window. CLAUDE.md files and auto memory carry knowledge across sessions." CLAUDE.md is written by you and loaded every session. Auto memory is written by Claude and loads "up to the first 200 lines or 25KB of MEMORY.md" — [Claude Code docs: How Claude remembers your project](https://code.claude.com/docs/en/memory)
- Official trigger criteria for adding a line to CLAUDE.md: "Claude makes the same mistake a second time"; "A code review catches something Claude should have known about this codebase"; "You type the same correction or clarification into chat that you typed last session"; "A new teammate would need the same context to be productive." The doc frames CLAUDE.md as "the place you write down what you'd otherwise re-explain" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Size and quality guidance: "target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence." Imports "don't reduce its context cost." On contradictions: "if two instructions contradict each other, Claude may pick one arbitrarily. Review ... periodically to remove outdated or conflicting instructions." Write instructions "concrete enough to verify", e.g. "Run `npm test` before committing" rather than "Test your changes" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Multi-step procedures, or rules that matter only for part of the codebase, should move to a skill or a path-scoped rule (`.claude/rules/*.md` with `paths:` glob frontmatter). Path-scoped rules load only when Claude reads, writes or edits a matching file — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Scopes: managed policy CLAUDE.md (`/etc/claude-code/CLAUDE.md` on Linux, or `claudeMd` in managed-settings.json; it can't be excluded), project `./CLAUDE.md` or `./.claude/CLAUDE.md` (shared via source control), user `~/.claude/CLAUDE.md` and `~/.claude/rules/`, and local `CLAUDE.local.md` (gitignored). Files are concatenated from the filesystem root down to the working directory rather than overriding each other. Subdirectory CLAUDE.md files load on demand — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- CLAUDE.md is "context, not enforced configuration". To block an action regardless, use a PreToolUse hook or managed `permissions.deny` — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- AGENTS.md interop: Claude Code (v2.1.277+) reads AGENTS.md when there's no CLAUDE.md. It can be set to read both (`claude-md-and-agents-md`) — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Built-in curation aid: `/doctor prompt-audit` (v2.1.283+) audits CLAUDE.md, CLAUDE.local.md, AGENTS.md, rules, skills, commands, subagents and output styles for outdated or conflicting instructions — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- `/init` bootstraps a CLAUDE.md. With `CLAUDE_CODE_NEW_INIT=1` it runs an interactive flow that explores with a subagent and "presents a reviewable proposal before writing any files" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)

**Claude Code: auto memory (agent-written layer)**
- Claude saves four note types, recorded in frontmatter `type`: `user` (role, preferences), `feedback` ("corrections you give Claude and approaches you confirm"), `project` (deadlines and decisions not derivable from code or git), and `reference` (pointers to external systems). It "skips anything it can derive from the codebase" and anything CLAUDE.md already says. "Claude doesn't save something every session" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Storage: `~/.claude/projects/<project>/memory/` holds a `MEMORY.md` index ("one line per memory, loaded at session start") plus one topic file per memory, which is read on demand. The store is machine-local and shared across worktrees of a repo, and is not synced across machines. It's excluded from transcript cleanup — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Built-in bloat control: after a write, Claude Code checks MEMORY.md against the 200-line / 25KB limits. Near a limit, it reminds Claude to "keep one line per entry, move detail into topic files, and merge or drop stale entries." Over the limit, it returns an error telling Claude to rewrite the index, "because everything past the limit is dropped on the next load" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Staleness support: when Claude writes a memory file with YAML frontmatter, Claude Code records a `modified` ISO-8601 timestamp, which "shows how current the fact is" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Controls: on by default locally, toggled via `/memory` (`autoMemoryEnabled`), `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`, or a custom `autoMemoryDirectory`. Subagents can have their own separate memory via a `memory` field. Main-session auto memory isn't loaded into subagents, except forks — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- Routing user intent: "always use pnpm, not npm" goes to auto memory. To put something in CLAUDE.md, ask "add this to CLAUDE.md" or edit via `/memory` — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
- The `#` quick-add shortcut from 2025-era Claude Code isn't mentioned anywhere on the current memory docs page (see Gaps). The current documented paths are `/memory`, asking Claude directly, and auto memory — [Claude Code memory docs](https://code.claude.com/docs/en/memory)

**Claude Code team practice (Boris Cherny)**
- Reported practice: when Claude does something wrong, the team adds it to the shared CLAUDE.md so it isn't repeated. The file is checked into git, and the whole team contributes "several times a week." In PR review he tags Claude (via the GitHub action) to add the lesson to CLAUDE.md, e.g. a reviewer comment asking for a rule preferring literal unions over enums. His personal CLAUDE.md is reportedly a couple of lines pointing at the team file, and the team file is reportedly ~2.5k tokens. **Caveat:** these are secondary write-ups of his posts on X, not verified against the originals — [InfoQ, Jan 2026](https://www.infoq.com/news/2026/01/claude-code-creator-workflow); [Push to Prod](https://getpushtoprod.substack.com/p/how-the-creator-of-claude-code-actually); [Karo Zieminski](https://karozieminski.substack.com/p/boris-cherny-claude-code-workflow)

**Claude API memory tool**
- Tool type `memory_20250818`, name `memory`. It runs client-side: Claude requests file operations under `/memories` and your handler maps them to storage you control. Commands: `view`, `create`, `str_replace`, `insert`, `delete`, `rename`. Status is stable and available on Claude 4+ models. It's ZDR-eligible because storage is client-side — [Claude API docs: Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- Auto-injected system prompt text: "IMPORTANT: ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING ELSE. MEMORY PROTOCOL: 1. Use the `view` command ... to check for earlier progress. 2. ... As you make progress, record status / progress / thoughts etc in your memory. ASSUME INTERRUPTION: Your context window might be reset at any moment..." — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- Suggested anti-clutter prompt: "always try to keep its content up-to-date, coherent and organized. You can rename or delete files that are no longer relevant. Do not create new files unless necessary." The docs also suggest scoping what gets written ("Only write down information relevant to <topic>") — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- Pairs with context editing (client-side clearing of tool results) and server-side compaction: "compaction keeps the active context small ... and memory preserves the information that must survive summarization." SDK helpers exist: `BetaAbstractMemoryTool` and `BetaLocalFilesystemMemoryTool` (Python/TS), `betaMemoryTool` (TS), `BetaMemoryToolHandler` (Java) — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- Documented "multisession software development pattern": an initializer session sets up a progress log, a feature checklist and an init-script reference. Later sessions read them first, and each session ends by updating the progress log. "Mark a feature complete only after end-to-end verification" — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)

**Long-running agent harness (progress files)**
- Anthropic Engineering, "Effective harnesses for long-running agents" (Justin Young, Nov 26, 2025). An initializer agent (first session only) creates `init.sh`, `claude-progress.txt` and an initial git commit. Coding agents in later sessions do one feature per session, commit with descriptive messages and update progress notes — [Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- The feature list is JSON: 200+ features for a claude.ai clone, all initially failing, each with `category`, `description`, `steps` and `passes`. Agents may only flip `passes`: "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality." JSON was chosen because the model is "less likely to inappropriately change or overwrite JSON" than Markdown — [Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Session start ritual: `pwd`, read the git log and progress file, pick the highest-priority failing feature, run init.sh and a basic end-to-end test before new work. Failure modes addressed: declaring victory early, leaving undocumented or broken state, marking features done without testing, and wasting time figuring out how to run the app — [Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

**Other products**
- **Cursor Memories** (introduced around v0.51, mid-2025): auto-generated rules from chat, managed under Settings → Rules, and only available with privacy mode off. Cursor staff on the forum say it was **intentionally removed starting in 2.1.x** (around Nov 2025). Users were told to "Export memories" and paste them into User/Project Rules. Not confirmed against an official changelog — [Cursor docs (legacy)](https://docs.cursor.com/context/memories); [Cursor forum: Are my memories gone?](https://forum.cursor.com/t/are-my-memories-gone/144057); [Cursor forum: Rules vs Memories](https://forum.cursor.com/t/rules-vs-memories-and-global-vs-project/137149)
- **Windsurf Cascade memories**: Cascade auto-generates memories during conversations when it finds context worth keeping. They're workspace-scoped, stored locally, not committed to the repo, and retrieved when judged relevant. The docs recommend rules (`.devin/rules`, legacy `.windsurf/rules`) or AGENTS.md for durable knowledge. Windsurf docs are now served under docs.devin.ai as "Devin Desktop" after Cognition's acquisition, and one localized page says memories apply only to the legacy Cascade agent, with a migration wizard to skills. Seen via search snippets only; the pages couldn't be fetched — [Windsurf docs: Memories](https://docs.windsurf.com/windsurf/cascade/memories.md); [Devin Desktop docs: Cascade memories](https://docs.devin.ai/desktop/cascade/memories.md)
- **Devin Knowledge**: couldn't be fetched (docs.devin.ai blocked). See Gaps — [Devin docs: Knowledge](https://docs.devin.ai/product-guides/knowledge)
- **ChatGPT memory**: not fetched this session. From prior knowledge, there are two mechanisms: "saved memories" (explicit, user-viewable and deletable; launched Feb 2024) and "reference chat history" (implicit recall across past chats; April 2025). Temporary Chat bypasses memory, and project-scoped memory came later. Verify against [OpenAI Memory FAQ](https://help.openai.com/en/articles/8590148-memory-faq) **[not fetched]**

**Community self-improving CLAUDE.md tooling**
- Community `/reflect`-style tools (no official Anthropic `/reflect`):
  - **claude-reflect** (Bayram Annakov): hooks detect corrections ("no, use X") and queue them in a JSON file. The user runs `/reflect` to approve each item (y/n) before anything is written to CLAUDE.md.
  - Gists with "reflect / deep reflect" prompts whose stated priority is "Merge over add — consolidate, don't accumulate."
  - A self-improve skill run from a SessionEnd hook that applies only high-confidence changes to the repo CLAUDE.md and avoids editing `~/.claude/CLAUDE.md`.
  - A Stop-hook system that updates a "Skill Memories" section.
  - Sources: [claude-reflect on skills.sh](https://skills.sh/bayramannakov/claude-reflect/claude-reflect); [hattenbachd gist](https://gist.github.com/hattenbachd/c8da26f9270d724e809ebc29804309da); [a-c-m reflection gist](https://gist.github.com/a-c-m/f4cead5ca125d2eaad073dfd71efbcfc); [yorrick self-improve skill](https://skillsmp.com/creators/yorrick/claude-code-plugins/self-improve-skill-skills-reflect). Not code-reviewed for maintenance or safety.
- A practitioner write-up ("We outgrew CLAUDE.md: building a knowledge layer that compounds") argues for moving beyond a single file to a structured knowledge layer — [DEV Community](https://dev.to/szkjn/we-outgrew-claudemd-building-a-knowledge-layer-that-compounds-33f) (title from search only; not fetched)

### Inferences
- The consensus loop has five steps: **observe a correction → decide its scope → write a concise, verifiable rule → review it (PR or approval prompt) → prune periodically.** The vendor shift from opaque auto-memories (Cursor removal, Windsurf "legacy") toward reviewable files suggests that auditability and team sharing won out over automatic capture for coding agents. Claude Code keeps both layers but separates them: CLAUDE.md for rules, auto memory for preferences and non-derivable facts.
- Progress files plus git are a distinct pattern: **task-state memory** rather than **lesson memory**. They share the same "read at start, write at end" ritual and the same failure mode (an agent trusting stale progress and declaring victory).
- Using JSON for state an agent must not rewrite casually (the feature list) is a general guardrail against over-eager memory edits.

### Gaps
- The current status of Claude Code's `#` memory shortcut couldn't be confirmed. It's absent from the current memory doc, which suggests it was removed or de-emphasized; the version wasn't found.
- Devin Knowledge (auto-suggested knowledge items, triggers, repo pinning) and the current ChatGPT memory docs couldn't be fetched.
- Boris Cherny quotes weren't verified against primary posts on X.
- A Claude Code "auto dream" memory-consolidation feature is referenced in a dev.to post titled "Does Claude Code need sleep? Inside the unreleased auto dream feature" ([dev.to](https://dev.to/akari_iku/does-claude-code-need-sleep-inside-the-unreleased-auto-dream-feature-2n7m)), but the page was blocked. Its status and mechanics are unverified, and the current memory docs and hooks docs don't mention it.

---

## Q2. Research systems

### Takeaway
The research systems converge on one recipe: **act → get feedback → reflect into natural-language lessons or reusable workflows → store them in a structured, itemized memory → retrieve or inject them next time.** The main design axes are:
- the unit of memory: raw observations, insights or rules, workflows or templates, or self-edited blocks;
- how memory gets into the prompt: whole-context injection versus retrieval;
- how updates happen: full rewrite versus incremental deltas. ACE shows that full rewrites cause context collapse.

Anthropic's Nov 2026 "components of agent memory" post adds empirical results on writing, reading and offline "dreaming" consolidation.

### Cited Findings

**ACE: Agentic Context Engineering (2025)**
- "Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models," arXiv:2510.04618. Authors: Qizheng Zhang, Changran Hu, ..., James Zou, Kunle Olukotun (Stanford, SambaNova, UC Berkeley). It treats context as an evolving "playbook" — [GitHub ace-agent/ace](https://github.com/ace-agent/ace); [arXiv](https://arxiv.org/abs/2510.04618); [VentureBeat](https://venturebeat.com/ai/ace-prevents-context-collapse-with-evolving-playbooks-for-self-improving-ai)
- It names two failure modes of prior prompt or context optimizers:
  - **Brevity bias**: optimization collapses toward short, generic prompts that drop domain detail.
  - **Context collapse**: "arises when an LLM is tasked with fully rewriting the accumulated context at each adaptation step ... compress[ing] the context into much shorter, less informative summaries, causing a dramatic loss of information."
  - Sources: [arXiv HTML v2 (via search snippet)](https://arxiv.org/html/2510.04618v2); [VentureBeat](https://venturebeat.com/ai/ace-prevents-context-collapse-with-evolving-playbooks-for-self-improving-ai)
- Architecture: **Generator** (produces trajectories and surfaces strategies and pitfalls) → **Reflector** (extracts insights, kept separate from curation) → **Curator** (turns lessons into structured **delta updates**, merged deterministically with de-duplication and pruning). Each playbook bullet has an ID, `helpful`/`harmful` counters and advice text. The "grow-and-refine" principle governs updates. It works offline (training-set adaptation) and online (test-time). No labeled supervision is needed: it uses natural execution feedback — [GitHub ace-agent/ace](https://github.com/ace-agent/ace)
- Results reported in the repo README:
  - AppWorld: +10.6% average. It matches the top-ranked production agent (IBM CUGA, GPT-4.1-based) on average and beats it on test-challenge, using a smaller open-source model.
  - Finance (FiNER + XBRL Formula): +8.6%.
  - Adaptation latency is 86.9% lower on average: −82.3% vs GEPA offline, −91.5% vs Dynamic Cheatsheet online.
  - It uses 75.1% fewer rollouts than GEPA, and token cost is 83.6% lower than Dynamic Cheatsheet.
  - Source: [GitHub ace-agent/ace](https://github.com/ace-agent/ace)
- Quoted authors' view: context "should not merely become shorter; it should become better organized" — [search summary of ACE coverage](https://www.altexsoft.com/blog/agentic-context-engineering/)

**Anthropic: "The components of agent memory" (Nov 6, 2026, Lance Martin)**
- Defines agent memory as knowledge "gained from experience" and "retained between sessions," excluding weights, in-context learning and compaction. The hierarchy, bottom to top: **transcripts** (searchable log) → **memory notes** (agent-written) → **memory representations** (e.g., linked wikis) → **injected context** (loaded every session; "keep small") — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- Write finding: tell Claude *what memory is for*, not how to use it. A prescriptive "record only failures" rule *reduced* the memory gain by 0.234 (Opus 5) and 0.150 (Opus 4.7) — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- Read finding: weaker models skip reading memory. Sonnet 4.6 read it less than half the time, and prompting it to read memory first nearly doubled its CL-Bench score (0.330 → 0.641). Sonnet 5 reads memory unprompted about 9 out of 10 times — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- **"Dreaming"** (offline consolidation, by analogy to hippocampal replay): tests notes against transcripts, rewrites wrong notes, merges duplicates and builds wiki links. Wiki scores on a 350,000-file corpus rose 13% → 48% over 128 dreaming sessions. In a Pokémon run, a corrected note got the agent past Mt. Moon, which the raw note never did. Dreaming can promote high-value notes into injected context, while "moving some details down the hierarchy and leaving links behind" — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- Failure observations:
  - A wiki grew to 1.77× the size of its source corpus, with many unlinked pages (bloat).
  - An incorrect injected note "can be carried into every session."
  - Concurrent agents writing one shared table erased each other's lines (41/57 contract facts survived). One writer per page fixed it (57/57).
  - Sonnet 3.7 deleted nine files during a costly reorganization.
  - Source: [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)

**Sleep-time compute (Letta + UC Berkeley, Apr 2025)**
- "Sleep-time Compute: Beyond Inference Scaling at Test-time" (arXiv:2504.13171, Kevin Lin, Charlie Snell et al.). Precomputing over context while idle cut the test-time compute needed for equal accuracy by about 5× on Stateful GSM-Symbolic and Stateful AIME. Its effectiveness correlated with how predictable the query was — [arXiv HTML](https://arxiv.org/html/2504.13171v1); [Letta blog, Apr 21 2025](https://letta.com/blog/sleep-time-compute)
- Letta productized this as "sleep-time agents" (v0.7.0): a primary agent plus a background agent that reorganizes shared memory blocks. The docs label it experimental. The paper's benchmarks measure reasoning cost and accuracy, not memory consolidation as such — [Letta docs](https://docs.letta.com/guides/agents/architectures/sleeptime/index.md); [Letta forum best-practices](https://forum.letta.com/t/sleeptime-agents-for-memory-consolidation-best-practices-guide/154)

**Systems summarized from abstracts (arXiv blocked; [abstract-level, not fetched])**
- **Generative Agents** (Park et al., 2023): a memory stream of observations is retrieved by recency × importance × relevance. Periodic **reflection** synthesizes higher-level insights, which are stored back into the stream (reflections on reflections form a tree). Ablations showed reflection matters for believable behavior — [arXiv:2304.03442](https://arxiv.org/abs/2304.03442)
- **Reflexion** (Shinn et al., 2023): after a failed trial, the agent writes verbal self-reflection into an episodic memory buffer (a small, bounded window of recent reflections) that conditions the next trial. It reported 91% pass@1 on HumanEval. The memory is within-task across trials, not general cross-task — [arXiv:2303.11366](https://arxiv.org/abs/2303.11366)
- **ExpeL** (Zhao et al., AAAI 2024): gathers successful and failed trajectories on training tasks. It extracts natural-language **insights** by comparing success/failure pairs and success sets, and maintains them with ADD / EDIT / UPVOTE / DOWNVOTE operations (insights whose votes fall to zero are removed). At test time it injects the insights and retrieves similar successful trajectories as examples — [arXiv:2308.10144](https://arxiv.org/abs/2308.10144)
- **Agent Workflow Memory (AWM)** (Wang et al., 2024): induces reusable, abstracted **workflows** (sub-routines with variables) from past trajectories and adds them to agent memory, offline or online. It reported relative success-rate gains of about 24.6% on Mind2Web and about 51.1% on WebArena — [arXiv:2409.07429](https://arxiv.org/abs/2409.07429)
- **MemGPT / Letta** (Packer et al., 2023): an OS-inspired virtual context. Core memory blocks sit in context and archival/recall storage sit outside it. The LLM **self-edits** memory through function calls (e.g., core_memory_append/replace, archival insert and search) and pages data in and out — [arXiv:2310.08560](https://arxiv.org/abs/2310.08560)
- **Dynamic Cheatsheet** (Suzgun et al., 2025): a test-time, persistent, self-curated "cheatsheet" of reusable strategies and code snippets, updated after each query (cumulative or retrieval-synthesis variants). It reported large gains on AIME and Game of 24 for GPT-4o and Claude 3.5 Sonnet. ACE treats it as a baseline that suffers from full-rewrite collapse — [arXiv:2504.07952](https://arxiv.org/abs/2504.07952)
- **Buffer of Thoughts** (Yang et al., NeurIPS 2024): a meta-buffer of high-level "thought templates" distilled from solved problems, with a buffer-manager that updates and dedups templates. It reported gains on Game of 24 and other reasoning tasks at about 12% of the cost of multi-query methods — [arXiv:2406.04271](https://arxiv.org/abs/2406.04271)
- **A-MEM** (Xu et al., 2025): Zettelkasten-style agentic memory. Each note gets keywords, tags and context descriptions, links to related notes, and new memories can trigger **memory evolution** (updating attributes of linked older notes) — [arXiv:2502.12110](https://arxiv.org/abs/2502.12110)
- **Mem0** (Chhikara et al., 2025): an extract → update pipeline. For each candidate fact, an LLM chooses ADD / UPDATE / DELETE / NOOP against similar existing memories; Mem0^g adds a graph variant. It reported higher LOCOMO accuracy than full-context and OpenAI memory baselines with much lower latency and token use. These are vendor-authored benchmarks — [arXiv:2504.19413](https://arxiv.org/abs/2504.19413)
- **Zep / Graphiti** (Rasmussen et al., 2025): a temporally-aware knowledge graph. Edges are **bi-temporal** (valid/invalid times), and contradicting facts are invalidated rather than deleted. It reported gains over MemGPT on DMR and LongMemEval. Vendor-authored — [arXiv:2501.13956](https://arxiv.org/abs/2501.13956); [Graphiti GitHub](https://github.com/getzep/graphiti)

### Inferences
- The systems differ mainly in the **unit of memory**:
  - insights or rules: ExpeL, ACE, Dynamic Cheatsheet, CLAUDE.md
  - procedures or workflows: AWM, Buffer of Thoughts, and in Claude Code, skills
  - episodic reflections: Reflexion, Generative Agents
  - facts about the user or world: Mem0, Zep, ChatGPT memory, auto memory
  - Coding-agent instruction files map most closely to ExpeL/ACE-style insight lists. Skills map to AWM-style workflows.
- ACE's helpful/harmful counters and ExpeL's upvote/downvote are the main research precedents for **evidence-weighted pruning** of rules. Practitioner tooling rarely implements them, and it could: for example, by tracking which CLAUDE.md lines were cited or violated.
- Anthropic's finding that prescriptive write rules hurt the memory gain (with frontier models) cuts against the heavy "only save X" templates in community tools. Its read-rate finding also explains why injected context (CLAUDE.md, the MEMORY.md index) is more reliable than memory the agent must choose to fetch, especially for smaller models.

### Gaps
- Exact numbers for the abstract-level papers (AWM, ExpeL, Dynamic Cheatsheet, BoT, Mem0, Zep) couldn't be re-verified this session because arXiv was unreachable.
- The specific ACE context-collapse example (a context shrinking from about 18k tokens to a few hundred with an accuracy drop) is widely cited, but it wasn't found in the sources reached. Don't quote exact figures without checking the paper.

---

## Q3. How should memories be curated? (write, retrieve, consolidate, prune, dedup, resolve conflicts, scope)

### Takeaway
Treat memory as a small, layered, reviewed artifact:
- **Write:** concise, verifiable, scoped lessons, and only for repeated or non-derivable knowledge.
- **Retrieve:** a small injected index plus on-demand topic files or path-scoped rules.
- **Consolidate:** offline ("dreaming", sleep-time), against ground truth (transcripts or tests).
- **Change by delta, not rewrite:** incremental edits, with dedup and evidence counters.
- **Resolve conflicts:** with timestamps or invalidation and one-writer-per-page.
- **Scope:** explicitly to org, team/project, user, local, or path.

### Cited Findings
- **Writing criteria.**
  - CLAUDE.md: add on a repeated mistake, a review catch, a repeated correction, or onboarding need. Keep rules concrete and verifiable — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Auto memory skips derivable info and anything CLAUDE.md already says — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Anthropic: give "simple guidance," say what memory is for, and let the model judge salience — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- **Retrieval layering.**
  - A small always-injected layer (CLAUDE.md, the first 200 lines/25KB of MEMORY.md), with topic files read on demand and path-scoped rules loaded on matching file access — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - The memory tool supports "just-in-time context retrieval" — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
  - Anthropic: transcripts form the foundation, with representations and search above them and injected context kept smallest — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- **Consolidation.**
  - Dreaming checks notes against transcripts, rewrites wrong notes, merges duplicates and adds links. Focus it on information likely to change, and tune its cadence to access patterns — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
  - Letta sleep-time agents reorganize memory blocks in the background — [Letta blog](https://letta.com/blog/sleep-time-compute)
- **Incremental, not rewrite.** ACE's Curator emits delta updates merged deterministically, with dedup and pruning, to avoid context collapse — [GitHub ace-agent/ace](https://github.com/ace-agent/ace). Community reflect prompts say "Merge over add — consolidate, don't accumulate" — [hattenbachd gist](https://gist.github.com/hattenbachd/c8da26f9270d724e809ebc29804309da)
- **Pruning and expiration.**
  - Memory tool: "Periodically delete memory files that haven't been accessed in a long time," and cap file sizes and `view` output — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
  - Claude Code's near-limit reminder tells Claude to "merge or drop stale entries" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - ExpeL removes insights whose votes fall to zero — [arXiv:2308.10144](https://arxiv.org/abs/2308.10144) [abstract-level]
- **Conflict resolution.**
  - Contradictory CLAUDE.md lines lead Claude to "pick one arbitrarily". Audit them with `/doctor prompt-audit` — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - `modified` timestamps on auto-memory files — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Zep invalidates superseded facts with bi-temporal edges — [arXiv:2501.13956](https://arxiv.org/abs/2501.13956) [abstract-level]
  - Mem0 decides UPDATE/DELETE against similar memories — [arXiv:2504.19413](https://arxiv.org/abs/2504.19413) [abstract-level]
  - One writer per page kept 57/57 facts vs 41/57 with a shared table — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
- **Scoping.**
  - Claude Code layers: managed/org → project (git-shared) → user → local (gitignored) → path-scoped rules → per-repo auto memory → per-subagent memory. `claudeMdExcludes` handles monorepos — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Windsurf memories are per-workspace and local, with rules/AGENTS.md for shared knowledge — [Windsurf docs](https://docs.windsurf.com/windsurf/cascade/memories.md)
  - Memory-tool `/memories` maps to per-user storage chosen by the application — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- **Review gate.** The team-shared CLAUDE.md is edited via PRs, with Claude tagged in review — [InfoQ](https://www.infoq.com/news/2026/01/claude-code-creator-workflow). claude-reflect requires per-item human approval — [skills.sh claude-reflect](https://skills.sh/bayramannakov/claude-reflect/claude-reflect)

### Inferences
Curation checklist synthesized from the sources:
1. **Gate the write.** Only write a memory for something repeated (two or more occurrences), non-derivable, or review-caught.
2. **Classify the scope.** Decide org, project, user, local or path before writing, and default to the narrowest scope that fixes the problem.
3. **Choose the right container.** Short always-true facts go in CLAUDE.md. Procedures go in skills. File-type-specific rules go in path rules. Personal facts go in auto memory. Task state goes in progress JSON.
4. **Edit, don't append.** Search for an existing related line and edit or merge it rather than appending.
5. **Record provenance.** Note the date, source and reason, so later pruning can judge staleness.
6. **Consolidate periodically.** Offline, verify memories against transcripts, tests or code, dedupe, and demote detail into linked files.
7. **Keep the injected layer under budget.** For example, CLAUDE.md under about 200 lines and the MEMORY.md index under 200 lines/25KB.

### Gaps
- There's no published quantitative study of optimal CLAUDE.md size versus adherence beyond Anthropic's qualitative "longer files reduce adherence."
- No standard schema for rule provenance in practitioner files was found.

---

## Q4. Failure modes

### Takeaway
The main risks:
- **Bloat** reduces adherence and costs context.
- **Collapse/brevity** from full rewrites erases hard-won detail.
- **Stale or contradictory rules** get trusted silently and propagate into every session.
- **Overfitting** to one-off corrections.
- **Concurrent-writer clobbering.**
- **Memory poisoning**: a single malicious write persists indefinitely, and that makes it more dangerous than one-shot prompt injection.
- **Privacy leakage** from storing sensitive data.

### Cited Findings
- **Bloat.**
  - "Longer files consume more context and reduce adherence"; target under 200 lines. Imports don't reduce cost — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Past the MEMORY.md limit, content "is dropped on the next load" — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - A dreaming wiki grew to 1.77× its source corpus — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
  - Community tools warn that accumulated notes bloat CLAUDE.md (one suggests under 300 lines) — [search summary of community reflect tools](https://gist.github.com/hattenbachd/c8da26f9270d724e809ebc29804309da)
- **Context collapse and brevity bias.** Full rewrites compress context into "much shorter, less informative summaries, causing a dramatic loss of information." Optimizers drift toward generic, short prompts — [ACE arXiv HTML (via search snippet)](https://arxiv.org/html/2510.04618v2); [VentureBeat](https://venturebeat.com/ai/ace-prevents-context-collapse-with-evolving-playbooks-for-self-improving-ai). The mitigation is delta updates plus grow-and-refine — [GitHub ace-agent/ace](https://github.com/ace-agent/ace)
- **Stale or contradictory rules.**
  - Contradictions lead Claude to pick one rule arbitrarily — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - "a note that's wrong from the start or that drifts as the environment changes can be read, and trusted." An incorrect injected note "can be carried into every session" — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
  - Long-running harness: agents saw earlier progress and declared the job finished prematurely — [Anthropic Engineering: harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- **Overfitting to one-off corrections / over-prescriptive memory.**
  - Anthropic found a narrow "record only failures" rule lowered memory gains (−0.234 Opus 5, −0.150 Opus 4.7) — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory)
  - Claude Code's "same mistake a second time" trigger is an explicit anti-overfitting threshold — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Session-end tools restrict themselves to "high-confidence" changes and to repo-local files — [yorrick self-improve skill](https://skillsmp.com/creators/yorrick/claude-code-plugins/self-improve-skill-skills-reflect)
- **Destructive reorganization and overwrites.** Sonnet 3.7 deleted nine files during a reorganization. Concurrent writers to a shared table erased each other's lines — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory). Using JSON for the feature list reduces inappropriate edits — [Anthropic Engineering: harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- **Memory poisoning / persistent prompt injection.**
  - **MINJA** (Memory INJection Attack) injects malicious records into an agent's memory bank through ordinary queries alone, with no direct memory access, using bridging steps and progressive shortening. Reported injection success rate >95% and attack success rate >70% on most datasets. A later paper cautions these were measured under idealized conditions — [arXiv:2503.03704](https://arxiv.org/abs/2503.03704); [The Register](https://www.theregister.com/2025/03/11/minja_attack_poisons_ai_model_memory/)
  - 2026 work argues memory poisoning "requires only one successful write," that existing prompt-injection defenses don't cover it, and describes sleeper attacks that re-emerge across later conversations — [arXiv:2606.04329](https://arxiv.org/html/2606.04329); [TechRadar](https://techradar.com/pro/security/experts-find-ai-agents-can-be-tricked-into-remembering-fake-facts-for-months-so-how-do-we-stop-it)
  - Johann Rehberger demonstrated persistent memory injection and exfiltration in ChatGPT (2024, "SpAIware") and other deployed assistants. These are cited in academic work, but his write-ups weren't fetched — [arXiv:2606.04329](https://arxiv.org/html/2606.04329); [Embrace The Red blog](https://embracethered.com/blog/) [not fetched]
  - **AgentPoison** (Chen et al., NeurIPS 2024) backdoors agent memory or RAG knowledge bases with optimized triggers — [arXiv:2407.12784](https://arxiv.org/abs/2407.12784) [abstract-level]
  - Product mitigations in Claude Code:
    - CLAUDE.md imports that resolve outside the working directory trigger a one-time approval dialog "to protect you from files other people commit to a shared project."
    - Symlinked external rules need the same approval.
    - Network-path symlinks aren't followed.
    - Cowork sessions skip out-of-tree user-scope imports.
    - Repo-supplied `autoMemoryDirectory` follows workspace-trust rules.
    - Source: [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Memory tool: validate every path to prevent `/memories/../../secrets.env`-style traversal, including URL-encoded `%2e%2e%2f` — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- **Privacy.**
  - Memory tool: "Claude usually refuses to write sensitive information to memory files. For stronger guarantees, add validation that strips sensitive data before your handler writes the file." Storage is client-side and ZDR-eligible — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
  - Claude Code auto memory is machine-local, not synced across machines, and kept outside transcript cleanup (it persists until deleted) — [Claude Code memory docs](https://code.claude.com/docs/en/memory)
  - Cursor Memories required privacy mode off — [Cursor forum](https://forum.cursor.com/t/rules-vs-memories-and-global-vs-project/137149)

### Inferences
- Persistence amplifies every other failure. A bad rule in the injected layer is effectively a permanent system-prompt edit. Self-improvement loops therefore need the same controls as code: review, provenance, tests or evals, and rollback (git).
- Automatic write paths (Stop/SessionEnd hooks that edit CLAUDE.md without review) are the highest-risk configuration for poisoning. Any content the agent read during the session, such as web pages, issues or tool output, can steer what gets "learned."

### Gaps
- No quantitative real-world incidence data on memory poisoning in coding-agent instruction files was found.
- Rehberger's specific ChatGPT findings weren't fetched.

---

## Q5. Concrete setup recipes

### Takeaway
The well-supported building blocks in Claude Code are hooks, skills/commands, auto memory, rules and git. The safe default loop:
1. **Capture** corrections cheaply during the session (UserPromptSubmit or Stop hook → queue file).
2. **Distill** at a natural boundary (PreCompact or end of session) into *proposed* diffs.
3. **Review** via `/reflect` or a PR.
4. **Consolidate** periodically (a scheduled job that dedupes and prunes against the 200-line budget).

### Cited Findings
- **Hook events available in Claude Code** include SessionStart, UserPromptSubmit, Stop, SubagentStop, PreCompact, PostCompact, InstructionsLoaded, SessionEnd and others — [Claude Code hooks docs](https://code.claude.com/docs/en/hooks)
  - **Stop**: receives `transcript_path` and `last_assistant_message`. It can block (exit 2 or `decision: "block"` keeps Claude going) and can feed context back via `hookSpecificOutput.additionalContext` — [hooks docs](https://code.claude.com/docs/en/hooks)
  - **PreCompact** (matcher `manual`/`auto`): can block compaction, and plain stdout "becomes instructions for the compaction summary". It's a natural point to tell the summarizer to preserve lessons, or to snapshot the transcript for later distillation — [hooks docs](https://code.claude.com/docs/en/hooks)
  - **SessionEnd**: cannot block. Hooks share a **1.5-second default budget**, which a per-hook `timeout` can raise to 60 s, so heavy LLM distillation should be backgrounded or deferred. Matcher reasons: `clear`, `resume`, `logout`, `prompt_input_exit`, `other` — [hooks docs](https://code.claude.com/docs/en/hooks)
  - **SessionStart** (matchers `startup`, `resume`, `clear`, `compact`, `fork`): stdout on exit 0 is added to context, which is useful for injecting a progress file or pending-lesson queue — [hooks docs](https://code.claude.com/docs/en/hooks)
  - **InstructionsLoaded** fires when CLAUDE.md or rules load (reasons include `session_start`, `path_glob_match`, `compact`), which is useful for logging which rules were active — [hooks docs](https://code.claude.com/docs/en/hooks)
- **claude-reflect pattern**: hooks detect correction phrases and queue them to JSON, then `/reflect` walks the queue with y/n approval before writing CLAUDE.md — [skills.sh claude-reflect](https://skills.sh/bayramannakov/claude-reflect/claude-reflect); [jeremylongshore plugin SKILL.md](https://raw.githubusercontent.com/jeremylongshore/claude-code-plugins-plus-skills/HEAD/plugins/community/claude-reflect/SKILL.md)
- **Session-end self-improve skill pattern**: run from a SessionEnd hook, apply only high-confidence changes, repo CLAUDE.md only — [yorrick self-improve skill](https://skillsmp.com/creators/yorrick/claude-code-plugins/self-improve-skill-skills-reflect)
- **PR-review loop**: tag Claude on PRs to add lessons to the shared CLAUDE.md via the GitHub integration — [InfoQ](https://www.infoq.com/news/2026/01/claude-code-creator-workflow)
- **Long-running harness recipe**: initializer prompt creates `init.sh`, `claude-progress.txt`, `feature_list.json` (only `passes` mutable) and an initial commit. Each session starts by reading the git log and progress file, running init.sh and a smoke test, doing one feature, committing, and updating progress — [Anthropic Engineering: harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents); the same pattern in [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- **API-side recipe**: `tools=[{"type":"memory_20250818","name":"memory"}]` with `BetaLocalFilesystemMemoryTool(base_path=...)` or a custom handler with path validation, size caps and expiry. Pair it with compaction or context editing — [Memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- **Consolidation job**: Anthropic's dreaming verifies notes against transcripts, merges duplicates, adds links and promotes or demotes between layers — [Anthropic Engineering](https://anthropic.com/engineering/components-of-agent-memory). Letta sleep-time agents run in the background — [Letta docs](https://docs.letta.com/guides/agents/architectures/sleeptime/index.md)
- **Built-in audit**: `/doctor prompt-audit` finds outdated or conflicting instructions across CLAUDE.md, rules and skills — [Claude Code memory docs](https://code.claude.com/docs/en/memory)

### Inferences (recipe sketches assembled from the above; not tested here)
- **Recipe A: capture → review (lowest risk).**
  - A `UserPromptSubmit` command hook greps the prompt for correction cues ("no,", "don't", "instead", "always", "never", "wrong") and appends `{ts, session, prompt}` to `.claude/lessons-queue.jsonl`.
  - A `/reflect` skill or command reads the queue plus CLAUDE.md and `.claude/rules/`, clusters items, and **keeps only items seen two or more times or explicitly flagged**. For each one it proposes the target file (project CLAUDE.md, a path rule, a skill, or CLAUDE.local.md) and a minimal edit-not-append diff, then asks for approval and clears processed items.
- **Recipe B: PreCompact lesson preservation.** A `PreCompact` hook prints: "In the summary, preserve under 'Lessons' any user corrections, failed approaches and confirmed conventions from this session." Optionally it copies `transcript_path` to `.claude/transcripts/` for later distillation.
- **Recipe C: SessionEnd deferred distillation.** Because of the 1.5 s budget, the SessionEnd hook only enqueues `transcript_path`, for example by spawning `nohup claude -p "/reflect --from-transcript <path> --propose-only" &`, or by writing to a queue that a scheduled job processes. The output is a branch or PR with proposed CLAUDE.md diffs, never a direct commit to main.
- **Recipe D: Stop-hook self-check (in-session).** A `Stop` hook with a prompt- or agent-type handler checks whether the last turn violated any CLAUDE.md rule, and returns `additionalContext` or blocks once to make Claude fix it. Logging which rules fire gives ACE-style helpful/harmful evidence for pruning.
- **Recipe E: periodic consolidation (weekly).**
  - A scheduled headless run (`claude -p` in CI or cron) runs `/doctor prompt-audit`, then a "dream" prompt. The prompt dedupes and merges CLAUDE.md, rules and MEMORY.md, removes lines contradicted by current code, demotes detail into topic files or skills to stay under about 200 lines, and checks auto-memory `modified` dates for staleness.
  - Output is a PR. One writer per file avoids clobbering.
- **Recipe F: team-shared rules in PRs.** The project CLAUDE.md and `.claude/rules/` live in git and are reviewed like code. Personal preferences go in `CLAUDE.local.md` or `~/.claude/CLAUDE.md`. Org guidance goes in managed CLAUDE.md, with hard enforcement via `permissions.deny` and PreToolUse hooks rather than prose.
- **Security defaults for all recipes:**
  - never auto-commit learned rules;
  - strip secrets before writing;
  - treat text from tool or web output as untrusted when proposing rules;
  - keep learned rules in git for rollback.

### Gaps
- No official Anthropic `/reflect` or "self-improving CLAUDE.md" feature was found. The "auto dream" consolidation feature is unverified (see Q1 gaps).
- The hooks doc page was truncated, so exact per-event JSON input schemas for SessionEnd and PreCompact, and whether prompt/agent handler types work on each event, weren't confirmed.
- No controlled evaluation was found that compares auto-written versus human-reviewed CLAUDE.md lessons on coding-agent performance.
