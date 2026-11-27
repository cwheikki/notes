# Periodic, Batch and Production Self-Improvement Loops for AI Agents

Scope: loops that run after a session, on a schedule, or continuously in production. Each one analyzes past runs (transcripts, traces, logs, user signals), finds bottlenecks, failures and friction, and feeds fixes back into prompts, CLAUDE.md or skills, harness config, eval suites, or model weights. Research date: 2026-11-27.

Source caveats: hamel.dev and dspy.ai were blocked by this session's network allowlist (DNS failure). Claims attributed to them are marked "(not fetched this session; prior knowledge)" and should be verified before they are quoted. arXiv IDs for the optimizer papers (OPRO, TextGrad, APE, ProTeGi, Trace, PromptWizard, MIPROv2) also come from prior knowledge unless marked otherwise.

Catalog at a glance (loop / cadence / input / output):
1. Post-session audit (/insights, session retrospective prompt): per session or on demand. Input: local transcripts. Output: CLAUDE.md rules, permission allowlists, workflow changes.
2. Periodic transcript mining (Clio-style, LangSmith Insights, Braintrust Topics): weekly or monthly. Input: many sessions or traces. Output: clustered failure modes and usage patterns, which become a roadmap and eval cases.
3. Error analysis and eval flywheel (open/axial coding, then binary judges, then a regression suite): weekly, plus on every change. Output: an eval suite that gates changes.
4. Automated prompt/program optimization (DSPy MIPROv2/GEPA, TextGrad, OPRO, APE, ProTeGi, Trace, PromptWizard): batch, run whenever the eval set or task changes. Output: optimized prompts or programs.
5. Production feedback and online learning (thumbs, implicit signals, A/B tests, canaries, Cursor Tab online RL): continuous. Output: model or policy updates, and keep/rollback decisions for prompt and harness versions.

---

## 1. Post-session audits (slow or wasted interactions, repeated tool failures, permission friction, user corrections and frustration)

### Takeaway
Claude Code ships a built-in post-hoc audit, `/insights`. It reads roughly the last 30 days of local transcripts and writes an HTML report of usage, friction patterns, and suggested CLAUDE.md rules and features. A 2026 release added an estimate of how many permission prompts auto mode could have absorbed. `/fewer-permission-prompts` is a narrower audit-to-fix loop: it mines transcripts for read-only commands and writes an allowlist. Anthropic has not published official documentation of the `/insights` internals; most of the detail comes from third-party write-ups.

### Cited Findings
- `/insights` builds a usage report from local session history. Most write-ups say it covers about 30 days, and one says "last X sessions". It reads transcripts, not code, and outputs an interactive HTML report, usually at `~/.claude/usage-data/report.html` (one blog says `~/.claude/data/report.html`). Reported contents: session and message counts, tool-use counts, token usage, "friction" patterns, suggested CLAUDE.md rules, and Claude Code features to try. — [Pasquale Pillitteri guide](https://pasqualepillitteri.it/en/news/408/claude-code-insights-workflow-command); [Prosperin AI](https://prosperinai.substack.com/p/claude-code-insights-command); [Vincent Qiao](https://blog.vincentqiao.com/en/posts/claude-code-insights/); [Skool / Clief Notes](https://www.skool.com/cliefnotes/try-insights-in-claude-code-it-reads-the-seams-in-your-system); [UX Planet](https://uxplanet.org/how-to-get-the-most-out-of-claude-code-491cfc79ee19)
- Claude Code v2.1.281 reportedly added an estimate of how many permission prompts auto mode could have handled, so the audit now quantifies permission friction directly. — [Nerdschalk](https://nerdschalk.com/claude-code-shows-where-recent-sessions-ran-into-trouble/) (secondary source; I did not confirm the year or release note)
- Practitioner write-ups and team rules treat `/insights` as a periodic workflow audit, for example an SSW rule that recommends running it regularly, and "What I got out of Claude Code Insights without rebuilding my workflow". — [SSW Rules](https://www.ssw.com.au/rules/use-claude-insights); [Atomic Object](https://spin.atomicobject.com/?p=3673478); [MindStudio](https://www.mindstudio.ai/blog/claude-code-insights-command-workflow-audit); [30-Day Usage Insights playbook](https://ao92265.github.io/claude-code-playbook/docs/usage-insights/)
- Some users publish their `/insights` reports, which gives real examples of the output. — [nicolasrouanne insights reports](https://nicolasrouanne.github.io/claude/insights/README.html); [jtsternberg claude-usage-data](https://jtsternberg.github.io/claude-usage-data/)
- A counterpoint post, "Why the analytics developer had no analytics", critiques what `/insights` does and does not tell you. — [DEV: dorothyjb](https://dev.to/dorothyjb/insights-or-why-the-analytics-developer-had-no-analytics-35nc)
- Raw material for any custom audit: Claude Code stores transcripts as JSONL at `~/.claude/projects/<project>/<session-id>.jsonl`. Each line is a message, a tool use or result, a model-request record, or metadata. The docs warn that "the entry format is internal to Claude Code and changes between versions, so scripts that parse these files directly can break on any release". Retention defaults to 30 days via `cleanupPeriodDays`, and `CLAUDE_CONFIG_DIR` relocates storage. — [Claude Code docs: Manage sessions](https://code.claude.com/docs/en/sessions)
- Sanctioned ways to do post-session analysis: `claude -p --resume <session-id> --output-format json "summarize what we changed"` asks a finished session about itself. A `SessionEnd` hook receives `transcript_path` and can archive the transcript when the session ends. — [Claude Code docs: Manage sessions](https://code.claude.com/docs/en/sessions)
- A session-retrospective prompt can be run in-session or resumed headlessly, for example `claude -p --resume <id>` (as above). Unattended runs should use `--permission-prompts none` or `permissions.denyInsteadOfPrompting`. With `stream-json`, denials appear as `permission_denied` events and in the result's `permission_denials` list, which is itself a permission-friction signal. — [Claude Code docs: headless](https://code.claude.com/docs/en/headless)
- `fewer-permission-prompts` is a Claude Code skill that "scan[s] your transcripts for common read-only Bash and MCP tool calls, then add[s] a prioritized allowlist to project .claude/settings.json". It closes a friction loop end to end. — Claude Code built-in skill listing (observed in this environment's skill registry; no public URL found)

### Inferences
- Post-session audit signals to extract from JSONL, each tied to a fix:
  - Repeated identical tool errors (same tool and error string at least 3 times). Fix: a CLAUDE.md note or a tool/skill fix.
  - Permission prompts per session and prompts on read-only commands. Fix: allowlist, or auto mode.
  - User correction or frustration turns ("no", "stop", "that's wrong", "I said", interrupts, rewinds/checkpoint restores). Fix: CLAUDE.md rule or skill.
  - Long wall-clock gaps between tool calls, or very long Bash runs. Fix: background commands, faster test targets.
  - Compactions per session (a sign of context bloat). Fix: subagents, shorter CLAUDE.md.
  - Abandoned sessions (no final assistant turn). Fix: investigate the task type.
- Treat `/insights` as the lightweight personal loop and a custom `claude -p` job as the team loop. The custom job can aggregate across machines, but only via exported or archived transcripts (SessionEnd hook), because the docs say the raw format is unstable.
- Codex/Cursor equivalents: I found no source for a first-party "/insights"-style retrospective in Codex CLI or Cursor. Both store local session logs (Codex under `~/.codex/sessions`, from prior knowledge, not verified this session), so the same `claude -p`/LLM-batch recipe applies.

### Gaps
- No official Anthropic documentation page for `/insights` was found. Report path, window, and friction taxonomy come from third-party blogs and conflict in places.
- No source on how the Claude Code team uses `/insights` or transcript feedback internally (dogfooding), beyond the evals-post anecdote in section 3.
- No verified first-party equivalents in OpenAI Codex or Cursor.

---

## 2. Periodic transcript mining at scale (clustering failure modes and bottlenecks across many sessions)

### Takeaway
The dominant pattern is hierarchical, LLM-driven clustering of many conversations: extract facets, embed or cluster, name the clusters with an LLM, then let humans drill down. Anthropic's Clio, now called "Anthropic Insights", is the reference design with privacy layers. LangSmith Insights Agent and Braintrust Topics productize the same pattern for agent traces. These loops surface patterns that cannot be seen in any single conversation.

### Cited Findings
- Clio (Dec 12, 2024; the page notes the system is now called "Anthropic Insights") runs four steps, all performed by Claude rather than human analysts. (1) Extract facets per conversation: topic, number of turns, language. (2) Semantic clustering. (3) Cluster titles and summaries written to exclude private details. (4) A multi-level hierarchy explored in an interactive UI. — [Anthropic: Clio](https://www.anthropic.com/research/clio)
- Clio's privacy layers: Claude is instructed to omit private details during extraction; clusters need a minimum number of unique users or conversations; Claude audits cluster summaries for identifying details before analysts see them; access is restricted. Outputs are not used for automated enforcement. — [Anthropic: Clio](https://www.anthropic.com/research/clio)
- Clio findings from 1M claude.ai conversations: web and mobile app development was the largest category at over 10%, teaching and learning over 7%, business strategy and operations about 6%. It also found an SEO-spam network of automated accounts that broke no single-conversation rule. Clio surfaced classifier false negatives and false positives, with correlation r = 0.71 against existing classifiers. Anthropic used it to monitor new-feature rollouts (computer use) and election-related activity. — [Anthropic: Clio](https://www.anthropic.com/research/clio)
- LangSmith Insights Agent clusters production traces without predefined categories. It produces top-level categories with drill-down subcategories, uses a "thinking model" for the clustering step, and is aimed at two questions: how users use the agent, and how it fails. It can be combined with quantitative signals, for example clustering only thumbs-down traces. It is GA for Plus and Enterprise cloud. — [LangChain blog: Insights Agent and Multi-turn Evals](https://blog.langchain.com/insights-agent-multiturn-evals-langsmith); [LangSmith Insights docs](https://docs.langchain.com/langsmith/insights); [From Traces to Insights](https://blog.langchain.com/from-traces-to-insights-understanding-agent-behavior-at-scale)
- LangChain's newer "LangSmith Engine" reportedly groups multiple traces that show the same pattern into a single named issue, moving from clusters to an issue tracker. — [LangChain: Introducing LangSmith Engine](https://langchain.com/blog/introducing-langsmith-engine) (seen in search snippet only; not fetched)
- Braintrust Topics classifies each log by user intent, sentiment, and issues, embeds the summaries, and clusters them. One Braintrust page lists the method as UMAP + HDBSCAN + c-TF-IDF. Topics can be saved so new traces are auto-classified. Minimum data is inconsistent across Braintrust's own pages: about 200 logs on one, about 100 facet summaries on another. Loop is Braintrust's natural-language agent over logs; its "Patterns" feature investigates small trace sets in depth. — [Braintrust Topics docs](https://braintrust.dev/docs/observe/topics); [Topics architecture blog](https://www.braintrust.dev/blog/topics-architecture); [Analyzing production logs](https://www.braintrust.dev/foundations/analyzing-production-logs)
- Academic work on mining reverts: an MSR 2026 study of 33,580 agentic PRs (86,315 commits) from Claude, Copilot, Cursor, Devin, and Codex found reverts stem mainly from scope management and contextual understanding, rather than purely functional bugs. — [MSR 2026 Mining Challenge](https://2026.msrconf.org/details/msr-2026-mining-challenge/27/) (search snippet; not fetched)

### Inferences
- Recipe for a team-level Clio-lite:
  1. Weekly, export or collect transcripts (SessionEnd hook, then a shared bucket).
  2. Run a cheap model per session to extract facets into a JSON schema: task type, outcome (success/partial/abandoned), first failure point, tools that errored, user-correction count, permission-prompt count, wall-clock time, tokens/cost.
  3. Cluster the "first failure" free-text facet with an LLM or embeddings + HDBSCAN.
  4. Have an LLM name each cluster and check it for PII.
  5. Enforce a minimum cluster size, as Clio does, before anyone reads it.
  6. Rank clusters by frequency × cost.
  7. Emit the top 3–5 as tickets with exemplar session IDs.
- Clio's thresholds and summary audits carry over to internal agent transcripts, which often contain secrets, customer data, and code.

### Gaps
- No public details on the Claude Code team's own transcript-mining cadence or tools.
- Arize Phoenix and Langfuse clustering features were not researched in depth this session. Both offer trace-embedding clustering and annotation queues (prior knowledge; unverified).

---

## 3. Error analysis and eval flywheels (Hamel/Shreya method, LLM-as-judge alignment, failures to regression evals)

### Takeaway
The best-documented loop is: read traces, then open-code the first failure in each, then axial-code those notes into a failure taxonomy, then write targeted binary evaluators (code first, LLM judge where needed), then align judges to human labels, then run them in CI and on production samples, then repeat. Anthropic's Jan 2026 evals post endorses the same flywheel for agents. Start with 20–50 tasks drawn from real failures. Separate capability evals (low pass rate) from regression evals (about 100%). Read transcripts to verify that graders are fair.

### Cited Findings
- Anthropic, "Demystifying evals for AI agents" (Jan 9, 2026): "20-50 simple tasks drawn from real failures is a great start". Early on, effect sizes are large, so small samples suffice. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Same post on turning failures into evals: start from the manual pre-release checks you already run, and convert user-reported failures from bug trackers and support queues into test cases, prioritized by user impact. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Capability evals should start at a low pass rate ("a hill to climb"). Regression evals should sit near 100%. Saturated capability evals "graduate" into continuously run regression suites. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Grader hierarchy: prefer deterministic code graders (tests, static analysis, tool-call checks), use model graders (rubrics, assertions, pairwise) where needed and calibrate them against humans, and use human graders sparingly. "It's often better to grade what the agent produced, not the path it took." — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- "Read the transcripts!" and "Failures should seem fair". A 0% pass rate across many trials usually signals a broken task or grader, "not an incapable agent". — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Use pass@k when one success suffices and pass^k when consistency matters. Example: 75% per-trial success gives pass^3 ≈ 42%. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Claude Code example: the team began with fast iteration on feedback, then added evals for concision, file edits, and later over-engineering. These fed into production monitoring, A/B tests, and user research. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Grader bugs can dominate scores: CORE-Bench, where Opus 4.5 scored 42% until grading bugs and ambiguous specs were fixed and it reached 95%. Qodo underrated Opus 4.5 because one-shot evals missed gains on longer tasks. Descript grades on "don't break things, do what I asked, and do it well." Bolt built an eval system combining static analysis, browser agents, and LLM judges in about three months. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Shankar et al., "Who Validates the Validators?" (UIST 2024, arXiv 2404.12272). EvalGen helps users write criteria, generates candidate implementations (Python functions or LLM grader prompts), and selects those that best match the user's grades on a subset. The key finding is "criteria drift": grading outputs changes users' criteria, so "evaluation criteria cannot be fully fixed before humans judge outputs". — [arXiv 2404.12272](https://arxiv.org/abs/2404.12272); [Berkeley PDF](https://people.eecs.berkeley.edu/~bjoern/papers/shankar-validators-uist2024.pdf); [Arize breakdown](https://arize.com/blog/breaking-down-evalgen-who-validates-the-validators/)
- Open coding as applied by Husain and Shankar: read each trace and note the first thing that went wrong; passing traces get no note. Codes emerge from the data rather than a fixed checklist, because a predefined list only catches failures someone already imagined. Axial coding then groups the open codes into categories and maps their relationships. — [DesignGurus course summary (secondary)](https://www.designgurus.io/course-play/grokking-llm-evals/doc/open-coding-notes-on-what-went-wrong); [Simply Psychology: axial coding](https://www.simplypsychology.org/axial-coding.html)
- Hamel Husain's evals FAQ and "Your AI Product Needs Evals" (not fetched this session; prior knowledge; verify before quoting). Main points: review on the order of 100 traces or until "theoretical saturation" (no new failure types appear); one domain expert ("benevolent dictator") owns the labels; prefer binary pass/fail judges over 1–5 Likert scales; validate each LLM judge against human labels using TPR/TNR on held-out data rather than raw agreement; avoid generic off-the-shelf metrics (helpfulness, toxicity) in favor of application-specific failure modes; build a simple custom annotation/trace viewer; spend most eval effort on error analysis. Also argues for a three-level stack (unit tests/assertions, then human + model eval, then A/B tests) and that removing friction from looking at data is the highest-leverage investment. — [hamel.dev evals FAQ](https://hamel.dev/blog/posts/evals-faq/); [hamel.dev "Your AI Product Needs Evals"](https://hamel.dev/blog/posts/evals/); [hamel.dev LLM-as-judge](https://hamel.dev/blog/posts/llm-judge/)
- Clio's r = 0.71 correlation with existing classifiers is an example of measuring LLM-based analysis against an existing signal before trusting it. — [Anthropic: Clio](https://www.anthropic.com/research/clio)

### Inferences
- Data flywheel for agents:
  1. Production traces go to sampling, stratified toward thumbs-down, user corrections, high cost, and new clusters.
  2. Open coding, then axial coding, then a failure taxonomy ranked by frequency.
  3. Each top failure mode gets a binary evaluator: a code check if possible, otherwise an LLM judge aligned against at least 50–100 human labels (from Hamel's guidance, unverified).
  4. Each fixed failure becomes a regression case.
  5. CI runs the suite on every change to prompts, harness, tools, or models.
  6. The suite is re-run on a production sample on a schedule to catch drift.
  7. Repeat open coding periodically, because criteria drift (Shankar) means the taxonomy goes stale.
- Cadence: Anthropic gives no fixed cadence. Weekly error-analysis sessions and per-PR regression runs are a defensible default.

### Gaps
- Could not fetch hamel.dev, so exact numbers (100 traces, TPR/TNR thresholds) are unverified this session.
- Eugene Yan's posts on evals and LLM-judge alignment (eugeneyan.com) were not fetched.
- Arize/Phoenix and Langfuse annotation-queue specifics are unresearched.

---

## 4. Automated prompt and program optimization against evals

### Takeaway
Once a metric and a labeled set exist, optimizers can search over prompts, demos, or whole programs. GEPA (reflective prompt evolution, ICLR 2026 oral) is the current state of the art among prompt optimizers. It reads execution traces in natural language, proposes edits, and keeps a Pareto frontier. It reports beating GRPO RL with far fewer rollouts and beating MIPROv2 by more than 10pp. Older methods (APE, OPRO, ProTeGi, TextGrad) are useful baselines. Head-to-head comparisons are scarce and depend on setup.

### Cited Findings
- GEPA, "Reflective Prompt Evolution Can Outperform Reinforcement Learning" (arXiv 2507.19457, ICLR 2026 oral). v2 reports +6pp on average over GRPO across six tasks (up to +19–20pp) with up to 35x fewer rollouts, and more than 10pp over MIPROv2 (e.g., +12pp on AIME-2025). v1 reported +10% across four tasks, so the headline numbers differ by version. Evaluation is mainly on Qwen3 8B and by the authors themselves. Code: github.com/gepa-ai/gepa. — [arXiv 2507.19457](https://arxiv.org/abs/2507.19457); [ICLR 2026 oral](https://iclr.cc/virtual/2026/oral/10009494); [ICLR proceedings](https://proceedings.iclr.cc/paper_files/paper/2026/hash/0e9e708b6f48e14fd0ac29e167413f76-Abstract-Conference.html)
- Taxonomy from a 2025 arXiv comparison: APE and OPRO are gradient-free (generate and score candidates; OPRO uses the trajectory of past prompts and scores as a meta-prompt). TextGrad and AdalFlow use "textual gradients" (LLM critiques backpropagated through a computation graph); TextGrad's validation-based selection is greedy with a revert step. ProTeGi ("APO") uses textual gradients plus beam search. — [arXiv 2506.00400](https://arxiv.org/html/2506.00400v3); [APO survey arXiv 2502.16923](https://ar5iv.labs.arxiv.org/html/2502.16923)
- Cost-aware benchmark (about 2,500 labeled products, authors' own study): ProTeGi had the strongest absolute performance at lower API cost but higher compute time; a hybrid APE-OPRO was about 18% more cost-efficient than OPRO. — [arXiv 2507.15884](https://arxiv.org/pdf/2507.15884)
- Reimplementations use very different budgets (e.g., OPRO at 8 candidates × 4 steps vs. 8 × 20 rounds), so cross-paper numbers are not comparable. — [arXiv 2506.00400](https://arxiv.org/html/2506.00400v3); [SCULPT arXiv 2410.20788](https://arxiv.org/pdf/2410.20788)
- Primary papers (IDs from prior knowledge; not fetched this session):
  - APE: Zhou et al. 2022, "LLMs are Human-Level Prompt Engineers", [arXiv 2211.01910](https://arxiv.org/abs/2211.01910)
  - ProTeGi: Pryzant et al. 2023, [arXiv 2305.03495](https://arxiv.org/abs/2305.03495)
  - OPRO: Yang et al. 2023, "Large Language Models as Optimizers", [arXiv 2309.03409](https://arxiv.org/abs/2309.03409)
  - DSPy: Khattab et al. 2023, [arXiv 2310.03714](https://arxiv.org/abs/2310.03714)
  - MIPRO: Opsahl-Ong et al. 2024, [arXiv 2406.11695](https://arxiv.org/abs/2406.11695)
  - TextGrad: Yuksekgonul et al. 2024, [arXiv 2406.07496](https://arxiv.org/abs/2406.07496), later published in Nature
  - Trace/OptoPrime: Microsoft, Cheng et al. 2024, [arXiv 2406.16218](https://arxiv.org/abs/2406.16218)
  - PromptWizard: Microsoft 2024, [arXiv 2405.18369](https://arxiv.org/abs/2405.18369)
- DSPy optimizer guidance (dspy.ai blocked; prior knowledge, verify): BootstrapFewShot for about 10 examples; BootstrapFewShotWithRandomSearch for about 50; MIPROv2 (Bayesian search over instructions and demos) for 200+ examples or longer runs; GEPA (`dspy.GEPA`) uses textual feedback from the metric; BootstrapFinetune distills into weights; SIMBA is a newer stochastic introspective optimizer. All need a metric function and train/val split, and DSPy docs recommend a small train set with a larger validation set for prompt optimizers. — [DSPy optimizers docs](https://dspy.ai/learn/optimization/optimizers/)

### Inferences
- How this fits the periodic loop: the eval set from section 3 becomes the optimizer's train/val data, and the judges become the metric. GEPA's design (reflect on full traces, propose edits) is the automated version of the human "read transcripts, then fix the prompt" loop, which is why it is sample-efficient.
- Practical recipe: freeze a held-out test split the optimizer never sees. Run the optimizer offline (nightly or per release). Accept a candidate only if it beats the baseline on the held-out split and passes the regression suite. Ship behind an A/B test or canary (section 5). Log prompt versions.
- Agent harness prompts (system prompts, tool descriptions, CLAUDE.md, skills) can be optimized the same way. Cost per rollout is high, so GEPA-style low-rollout methods fit better than RL or large random search.

### Gaps
- No independent, same-budget benchmark of GEPA vs. MIPROv2 vs. TextGrad vs. OPRO on agentic coding tasks was found.
- Few published practitioner case studies of optimizers on production agent harnesses (as opposed to QA/classification tasks) were found.

---

## 5. Production feedback loops (explicit and implicit signals, A/B tests, canaries, online RL)

### Takeaway
The strongest public example of a closed production loop is Cursor's Tab online RL. It uses policy-gradient training on accept/reject signals from more than 400M requests per day and redeploys a new checkpoint every 1.5–2 hours to keep the data on-policy. The result was 21% fewer suggestions with a 28% higher accept rate. For agents, implicit signals (PR merges, code survival, reverts, user edits, restated requirements) are sparse but higher fidelity than thumbs. Research is combining them with dense trace-level rubrics.

### Cited Findings
- Cursor, "Improving Cursor Tab with online RL" (Sept 2025):
  - The Tab model handles more than 400M requests per day.
  - Policy-gradient reward example: +0.75 for accepted, −0.25 for rejected, 0 for no suggestion shown. This means a suggestion should be shown only when P(accept) > 25%. The production reward also accounts for suggestion size and jumps to other locations.
  - The goal was for the model itself to stop producing bad suggestions, rather than filtering them afterwards.
  - Because training needs on-policy data, Cursor ships new checkpoints frequently; rolling out a checkpoint and collecting data for the next step takes 1.5–2 hours.
  - Result: 21% fewer suggestions and a 28% higher accept rate. (Older than one year as of Nov 2026.)
  — [Cursor blog: tab-rl](https://cursor.com/blog/tab-rl)
- Anthropic: Claude Code's evals fed into production monitoring and A/B tests alongside user research. Production failures from support and bug trackers become eval cases. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- LangSmith Insights pairs quantitative feedback (thumbs-down) with clustering, to explain why users are unhappy. — [LangChain blog](https://blog.langchain.com/insights-agent-multiturn-evals-langsmith)
- Research on implicit signals for coding agents (2026 arXiv, seen via search snippets; not fetched):
  - Learning a critic from noisy human-agent traces, using 24 trace-observable rubric features plus sparse production outcomes (PR merges, code survival) that cover only 4–6% of data, versus 100% coverage for dense rubrics. — [arXiv 2603.03800](https://arxiv.org/pdf/2603.03800)
  - SWE-Touch treats user edits to the workspace as feedback carrying location and intent signal. — search result set including [arXiv 2602.19441](https://www.arxiv.org/pdf/2602.19441) (attribution of specific claims to specific papers in this result set is uncertain)
  - A feedback-labeling scheme with an "implicit_behavioral" class: no evaluative language, inferred from behavior such as restating requirements or supplying one's own solution. — same result set (uncertain attribution)
  - Reviewer engagement correlates most strongly with successful integration of agent PRs. — [MSR 2026](https://2026.msrconf.org/details/msr-2026-mining-challenge/27/)
- Claude Code headless output gives per-run metrics for production monitoring of scripted agents: `--output-format json` includes `total_cost_usd` and per-model cost, and the `system/api_retry` events expose retry and overload friction. — [Claude Code docs: headless](https://code.claude.com/docs/en/headless)

### Inferences
- Signal hierarchy for coding agents:
  - Explicit: thumbs up/down. Sparse and biased toward extremes.
  - Weak implicit: retries, regenerations, interrupts, rewinds.
  - Medium: user edits to agent output soon after it is produced, and the user restating the request.
  - Strong but delayed: PR merged, CI green, code survives N days without a revert.
  Combine them: use strong signals as labels for training or validating judges, and dense signals for monitoring.
- Canarying harness changes: version every system prompt, tool description, and model alias. Route a small share of traffic or users to the new version. Compare the same signal set plus regression-suite pass rate. Auto-roll back on regression. Cursor's rollout cadence shows this can run on an hours-scale loop when traffic is high.
- Devin (Cognition), Factory, and Sourcegraph Amp: no primary sources found in this session describing their specific feedback loops.

### Gaps
- No primary sources found for Devin, Factory, or Sourcegraph/Amp production feedback practices.
- No public numbers found on thumbs-feedback rates or noise for agent products.

---

## 6. Failure modes of these loops

### Takeaway
The main risks are:
- Optimizing to a broken or gameable metric (Goodhart's law, grader bugs, eval saturation).
- Judges that drift from human intent (criteria drift, uncalibrated LLM judges).
- Privacy exposure from mining transcripts.
- Noisy or sparse feedback.
- Reports that nobody turns into changes.
The cited sources give concrete mitigations for most of these.

### Cited Findings
- Grader bugs and ambiguous specs can make a strong agent look weak (CORE-Bench: 42% → 95% after fixes), and one-shot evals can miss gains on long tasks (Qodo). — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Saturation: once an agent passes all solvable tasks, improvements show up only as tiny score changes. SWE-bench Verified went from about 30% to over 80%. Mitigation: graduate saturated evals to regression and build new capability evals. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- A 0% pass rate across many trials usually means the task or grader is broken. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- Criteria drift: humans' criteria change as they grade, so fixed judges go stale, and LLM evaluators "inherit the flaws" of the models they judge and need human validation. — [arXiv 2404.12272](https://arxiv.org/abs/2404.12272)
- Privacy: Clio's mitigations are omission instructions, minimum cluster sizes, LLM audit of summaries, restricted access, data minimization and retention limits, and no automated enforcement. Anthropic acknowledges safeguards "might miss some private information" and that some users may perceive this as invasive. — [Anthropic: Clio](https://www.anthropic.com/research/clio)
- Local transcript retention defaults to 30 days (`cleanupPeriodDays`). A weekly or monthly job must run before the cleanup, or archive transcripts via a SessionEnd hook. `CLAUDE_CODE_SKIP_PROMPT_HISTORY` and `--no-session-persistence` suppress transcripts, which leaves gaps in any audit. — [Claude Code docs: sessions](https://code.claude.com/docs/en/sessions)
- Brittle parsing: the JSONL format is internal and "changes between versions". — [Claude Code docs: sessions](https://code.claude.com/docs/en/sessions)
- Running headless review jobs over untrusted repos: without `--bare`, `claude -p` runs project hooks and MCP servers "even in a folder you've never trusted", with no trust dialog. — [Claude Code docs: headless](https://code.claude.com/docs/en/headless)
- Benchmark noise in optimizer comparisons: results depend heavily on budget and setup, and GEPA's headline numbers changed between v1 and v2. — [arXiv 2507.19457](https://arxiv.org/abs/2507.19457); [arXiv 2506.00400](https://arxiv.org/html/2506.00400v3)
- On-policy staleness in online RL: once the policy is updated, old samples no longer reflect it, which forces a fast redeploy cadence. — [Cursor blog: tab-rl](https://cursor.com/blog/tab-rl)
- Sparse strong signals: PR merges and code survival covered only 4–6% of traces in one study. — [arXiv 2603.03800](https://arxiv.org/pdf/2603.03800) (snippet)

### Inferences
- Goodhart and overfitting mitigations:
  - Keep a frozen held-out set the optimizer never sees.
  - Rotate in fresh production-sampled cases.
  - Pair every LLM-judge metric with periodic human spot checks (TPR/TNR).
  - Track pass^k as well as pass@k for consistency.
  - Grade outcomes, not paths.
- Alert fatigue and improvements that never get applied: cap each report to the top 3–5 clusters ranked by frequency × cost. Have the job open PRs or issues (e.g., a CLAUDE.md diff or an allowlist PR, as `fewer-permission-prompts` does) instead of only producing reports. Track "suggestions applied" week over week. `/insights` suggestions are advisory and depend on the user acting on them.
- Self-reinforcing bias: an agent auditing its own transcripts may under-report its own failures. Use a separate model or prompt for the audit and anchor it on objective counts (errors, retries, denials) before any LLM judgment.

### Gaps
- No quantitative study found on how often auto-generated agent retrospective suggestions are actually adopted.
- No sources on judge drift across model upgrades (the judge model itself changing).

---

## 7. Concrete setup recipes

### Takeaway
All the pieces for a fully scripted loop are documented: transcripts at `~/.claude/projects/*/*.jsonl`, headless `claude -p` with JSON or schema output, `--permission-prompts none` for unattended runs, `--bare` for safe CI, SessionEnd hooks for archiving, and GitHub Actions support. The recipes below combine them. They are my synthesis built on the cited docs, not published Anthropic recipes.

### Cited Findings
- Headless flags:
  - `-p/--print`
  - `--output-format text|json|stream-json`
  - `--json-schema` (result in `structured_output`, which can be missing if Claude declines)
  - `--allowedTools` (permission-rule syntax, e.g. `Bash(git diff *)`)
  - `--permission-mode auto|dontAsk|acceptEdits`
  - `--permission-prompts none` (v2.1.259+)
  - `--bare` (skips hooks, skills, plugins, MCP, CLAUDE.md; needs `ANTHROPIC_API_KEY`; "will become the default for `-p`")
  - `--append-system-prompt`, `--max-budget-usd`, `--continue`, `--resume <id|path.jsonl>`, `--no-session-persistence`
  - Exit code 0 on success; JSON includes `total_cost_usd`
  — [Claude Code docs: headless](https://code.claude.com/docs/en/headless)
- Since v2.1.302, `claude -p` starts in Manual mode when no permission mode is set, so calls needing approval are denied unless allowed. — [Claude Code docs: headless](https://code.claude.com/docs/en/headless)
- `claude -p` and Agent SDK sessions are excluded from the session picker and from `--continue` (except with `-p --continue`). Audit jobs therefore don't pollute users' session lists, though they still write transcripts unless `--no-session-persistence` is set. — [Claude Code docs: sessions](https://code.claude.com/docs/en/sessions)
- GitHub Actions and GitLab CI/CD integrations are documented. — [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions); [GitLab CI/CD](https://code.claude.com/docs/en/gitlab-ci-cd)
- Claude Code also has scheduled tasks and `/loop` for recurring prompts (referenced from the headless docs). — [Claude Code docs: scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks)
- Start with 20–50 tasks from real failures for CI evals. — [Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### Inferences (recipes)

**Recipe A: weekly local bottleneck report (cron).**
1. Deterministic pre-pass, which is cheap and robust: a small Python script walks `~/.claude/projects/*/*.jsonl` modified in the last 7 days. It counts, per session: tool calls by name, tool_result entries flagged as errors, consecutive identical errors, compaction markers, user turns that match correction phrases ("no,", "stop", "wrong", "I said", "don't"), interrupts, and wall-clock duration. It writes `weekly_stats.json` plus a sample of the 20 worst sessions as excerpts. Parse defensively, because the format is internal and unstable.
2. LLM pass: run
   ```
   claude --bare -p "$(cat audit_prompt.md)" --allowedTools "Read,Write" --permission-prompts none --output-format json --max-budget-usd 2 --no-session-persistence
   ```
   The prompt says: read weekly_stats.json and the excerpts; cluster the friction into at most 5 bottlenecks ranked by frequency × time lost; for each, give the evidence (session IDs), a root-cause hypothesis, and a concrete fix (a CLAUDE.md line, an allowlist rule, a skill, or a hook). Write `reports/YYYY-WW.md`.
3. Crontab: `0 9 * * MON cd ~/audit && ./run_audit.sh`. Retention is 30 days by default, so weekly is safe. For monthly runs, raise `cleanupPeriodDays` or archive transcripts with a SessionEnd hook.
4. Closing the loop: have step 2 also emit a proposed CLAUDE.md or `.claude/settings.json` diff that a human reviews and applies. Track last week's fixes and whether their target metric moved.

**Recipe B: per-session retrospective via SessionEnd hook.** The hook receives `transcript_path`. It copies the transcript to an archive. Optionally it launches a background `claude --bare -p` with a short retrospective prompt ("list repeated tool failures, permission prompts, and user corrections in this transcript; propose at most 2 CLAUDE.md rules") and appends the result to `retros.md`. Have the hook detach the background job so it doesn't block exit.

**Recipe C: CI eval gate in GitHub Actions.**
- Keep `evals/` with 20–50 tasks drawn from real failures. Each task has a repo fixture, a prompt, and a code grader (tests or diff checks), plus an optional rubric LLM judge.
- On PRs that touch prompts, CLAUDE.md, skills, tools, or model config: run each task k=3 times with `claude --bare -p ... --output-format json --permission-mode dontAsk --allowedTools ...`, grade, and compute pass@k and pass^k plus cost.
- Fail the job if the regression suite drops below a threshold. Post capability-suite deltas as a PR comment.

**Recipe D: monthly team transcript mining (Clio-lite).** Archive transcripts centrally. Extract facets per session with a cheap model under a JSON schema (`--json-schema`). Cluster the facets. Enforce a minimum cluster size and a PII audit. Produce a ranked failure taxonomy. Turn the top clusters into eval cases (Recipe C) and backlog items.

**Recipe E: offline prompt optimization.** Use the Recipe C dataset plus a judge as the metric. Run DSPy GEPA or MIPROv2 on the harness prompt or a skill's instructions. Validate on a frozen held-out split. Ship via canary, then promote.

### Gaps
- I did not fetch the exact GitHub Actions workflow YAML (anthropics/claude-code-action inputs) this session, so workflow syntax is not given verbatim.
- No published, Anthropic-authored "weekly transcript audit" recipe was found. Recipes A–E are a synthesis.
