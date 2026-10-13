# CCDV-F Preparation Plan — Claude Certified Developer – Foundations

> A practical, beginner-friendly plan for passing **CCDV-F** while working full-time.
> Companion file: [Roadmap.md](Roadmap.md) (timeline, milestones, gates).

| | |
|---|---|
| **Plan length** | 10 weeks + 1 optional buffer week (Week 0 setup → Week 10 exam) |
| **Weekly effort** | ~7.5 hours (4 × 60 min on weekdays, 2.5 h Saturday, 1 h Sunday review). About 80 hours in total. |
| **Prepared** | 2026-10-13 |
| **Facts checked against** | Official CCDV-F Exam Guide v1.0 (effective July 2026), read in full on 2026-10-13 |

---

## 0. Read this first: eligibility and source status

> **Registration is currently partner-only.** The official FAQ (checked 2026-10-13) says certification is
> "available to people at Claude Partner Network organizations" and needs "a partner email address on a
> recognized company domain." Confirm you are eligible **before** you plan an exam date. The skills in this plan are worth having either way.

### Sources reviewed

| Source | Accessible? | What it gave this plan |
|---|---|---|
| [claudecertificationguide.com/ccdv-f][site-ccdvf] | ✅ Yes | Exam facts and all 25 sub-skill weights. Its own CCDV-F prep track, lessons and mock exam are marked **"Coming soon"**, so it has **no CCDV-F study content yet**. It is an independent site, not affiliated with Anthropic. It points to its free [Architect (Foundations) curriculum][site-learn], which overlaps with CCDV-F. |
| Local repo `D:\GitHub\ai-engineering-from-scratch\certifications\claude` | ✅ Yes | A 15-lesson CCDV-F route ([track file][track]), labs with tests, quizzes, a 16-question [diagnostic][diag], a 53-question [full mock][mock], and research notes. Repo status: "Local preview", last verified 2026-08-09. It is independent and not affiliated with Anthropic. |
| **Official** [CCDV-F Exam Guide v1.0 (PDF)][guide] | ✅ Yes (text extracted locally) | The authoritative blueprint, skill descriptions, format, policies and 3 sample questions. **All three sources give the same weights.** |
| Official [Certification FAQ][faq] | ✅ Yes | Eligibility, closed-book rule, ~135 min seat time, retired practice exam. |
| Official [CCDV-F prep path (Partner Academy)][prep-path] | ⚠️ Partly | The page loads, but its course list is rendered by JavaScript and needs a partner sign-in. The page says the path has 5 courses, but **I could not read the individual course names.** |

---

## 1. Exam at a glance (official, Exam Guide v1.0)

| Item | Detail |
|---|---|
| Items | **53** multiple-choice and multiple-response questions. Each item says how many answers to select. |
| Time | **120 min**. Plan for about **135 min** of seat time (FAQ). |
| Pass mark | Scaled score **720** on a 100–1,000 scale. This is criterion-referenced and is **not** a raw percentage. |
| Fee | $125 USD per attempt, before any partner-tier discount |
| Delivery | Pearson VUE, online proctored or at a test centre. **Closed book**: no notes, documentation, AI tools or browser translation. |
| Results | Pass/fail with a scaled score, plus percent-correct by domain |
| Retakes | Wait 14 days after the 1st fail, 30 days after the 2nd and 90 days after the 3rd. Maximum 4 attempts in a rolling 12 months. |
| Validity | 12 months. On-time renewal is a free, non-proctored assessment. A lapsed credential means sitting the full exam again. |
| Reschedule | Up to 24 h before the appointment. Later changes forfeit the fee. |
| Official practice | The old practice exam is **retired**. The guide has **only 3 sample questions**. |

**Time budget in the exam:** about 2.3 min per item. Do a first pass of about 85 min and flag doubtful items, then review for about 35 min.

---

## 2. Exam topics, prioritised by verified weight

The weights below are from the official guide. "Expected items" means weight × 53, rounded.

### Domain priority

| Priority | Domain | Weight | ≈ Items | Study share |
|:-:|---|--:|--:|---|
| **P1** | 2. Applications and Integration | **33.1%** | ~17–18 | Covered across Weeks 1, 3, 7 and 8. One-third of the exam. |
| **P1** | 5. Model Selection and Optimization | **16.8%** | ~9 | Weeks 1–3 |
| **P1** | 1. Agents and Workflows | **14.7%** | ~8 | Week 5 |
| **P2** | 6. Prompt and Context Engineering | **11.0%** | ~6 | Week 2 |
| **P2** | 8. Tools and MCPs | **10.6%** | ~6 | Week 4 |
| **P2** | 7. Security and Safety | **8.1%** | ~4 | Week 6 |
| **P3** | 3. Claude Code | **3.1%** | ~2 | Week 7. Also feeds Configuration Management (D2), so it is worth more than 3.1% suggests. |
| **P3** | 4. Eval, Testing, and Debugging | **2.6%** | ~1–2 | Week 6. Also used throughout as a working skill. |

### All 25 sub-skills, ranked by weight

Use this list to self-assess. Rate yourself 1–5 in Week 0, Week 5 and Week 9.

| # | Sub-skill (domain) | Weight | What the guide says is tested | Main repo lesson(s) |
|--:|---|--:|---|---|
| 1 | Claude Application Design (D2) | 8.6% | How Claude interprets instructions across Claude Code, Desktop, claude.ai, the API and SDKs; content boundaries; schema design; session hygiene; plugin management | [01][L01], [08][L08], [09][L09], [30][L30] |
| 2 | Software Engineering Foundations (D2) | 7.4% | REST, JSON, async, version control, SDLC integration, code review, small- and large-scale refactoring | ⚠️ **Gap.** No dedicated lesson. See §8. |
| 3 | Claude API Mechanics (D2) | 6.8% | Messages, tools, streaming, vision, thinking, caching, third-party vendors, data access patterns, Batch API, choosing realtime or batch | [08][L08], [02][L02] |
| 4 | Technical Fundamentals (D5) | 6.1% | Integrating with SDKs that wrap REST APIs; websockets | [08][L08]. ⚠️ No coverage of websockets. |
| 5 | Agent Construction with Claude (D1) | 5.3% | Agent SDK, custom loops and harnesses, managed agents (self-hosted vs Anthropic-hosted), hooks | [12][L12], [10][L10] |
| 6 | LLM Fundamentals (D5) | 5.2% | Tokens, context windows, sampling, non-determinism; fast mode, extended and adaptive thinking, effort; zero-, single- and multi-shot prompting | [01][L01], [02][L02], [03][L03] |
| 7 | Agent Patterns and Frameworks (D1) | 4.9% | Tool-use loops, sub-agents, memory, context-window management; **Strands, LangGraph, PydanticAI** | [10][L10], [12][L12], [16][L16]\*. ⚠️ No coverage of the named frameworks. |
| 8 | Prompt Engineering (D6) | 4.6% | Clear instructions, few-shot examples, system vs user placement, output constraints, iteration, input sanitisation | [03][L03] |
| 9 | Agent Architecture (D1) | 4.5% | Choosing a workflow or an agent; manager/supervisor hierarchies; subagents | [10][L10], [16][L16]\* |
| 10 | Tool Implementation (D8) | 4.4% | Function calling, tool descriptions, errors, harness dispatch, client-side vs server-side tools, approvals, building a tool set | [10][L10], [18][L18]\* |
| 11 | Configuration Management (D2) | 4.1% | CLAUDE.md, settings.json, model version pinning, prompt versioning, plugin dependencies | [15][L15], [19][L19]\* |
| 12 | Agentic Customization (D8) | 4.1% | Choosing among built-in tools, custom tools, Skills and MCP | [11][L11], [15][L15] |
| 13 | Context Engineering (D6) | 3.8% | Context drift and bloat, tool-output pruning, compaction, isolation with subagents | [04][L04], [17][L17]\* |
| 14 | Understanding Requirements (D2) | 3.4% | Turning business needs into functional and infrastructure requirements | [22][L22]\* (not in the CCDV-F route) |
| 15 | AI Application Security (D7) | 3.2% | Prompt injection, jailbreaks, untrusted input, data leakage, PII, authN/authZ | [13][L13] |
| 16 | Claude Code Operation (D3) | 3.1% | Rules, Skills, Commands, Agents, memory; sessions, headless, streaming and auto mode; the CLAUDE.md hierarchy; init; settings.json | [15][L15], [19][L19]\* |
| 17 | Systems Life Cycle (D2) | 2.8% | Developing, implementing, operating and maintaining IT systems | [14][L14], [28][L28]\* (light coverage) |
| 18 | Cost and Token Management (D5) | 2.8% | Tracking token usage, cost models, prompt caching, cache checkpointing | [02][L02], [04][L04] |
| 19 | Model Selection and Trade-offs (D5) | 2.7% | When to use Opus, Sonnet or Haiku; quality, latency and cost; breaking changes between model releases | [02][L02] |
| 20 | Output Handling (D6) | 2.6% | Structured output, validation, defensive parsing, scepticism toward confident output | [05][L05], [09][L09] |
| 21 | Debugging and Error Handling (D4) | 2.6% | Error types, choosing a recovery strategy, trace analysis, telling integration failures apart from model failures | [14][L14] |
| 22 | Guardrails and Safe Deployment (D7) | 2.3% | Content policy, layered guardrails, least privilege, IAM | [13][L13], [12][L12] |
| 23 | MCP Server Development (D8) | 2.1% | Authoring and deploying servers; tools, resources and prompts; stdio and other transports; client vs server | [11][L11] |
| 24 | Identity, Secrets, and Key Management (D7) | 1.6% | API keys across environments, authentication, access approval, monitoring | [13][L13] |
| 25 | Claude Hooks (D7) | 1.0% | Using hooks to block destructive actions | [12][L12] |

\* Lessons marked with an asterisk exist in the repo but are **not** in the official CCDV-F route ([tracks/ccdv-f.json][track]). This plan adds them because they cover blueprint gaps.

---

## 3. Before you start (Week 0, about 4 h)

**Prerequisite self-check.** The guide recommends the following. None of it is mandatory.

- [ ] I can write and run Python 3.10+ (or TypeScript) and use `pip`/`venv`.
- [ ] I can call a REST API, read JSON and handle HTTP errors.
- [ ] I can use git (branch, commit, PR) and a terminal.
- [ ] I have at least 6 months of hands-on use of Claude or a similar LLM. If not, add the 1-week buffer and do every lab yourself.

**If you are a beginner, close gaps in Week 0** with the repo phases first:
[Prompt engineering basics][P-prompt], [Structured outputs][P-struct], [The agent loop][P-loop].

**Setup tasks**

1. Read the [official exam guide][guide] in full (30 min). Answer the 3 sample questions.
2. Install Python, `pip install anthropic`, and [Claude Code][cc-overview]. Store your API key **only** in an environment variable (`ANTHROPIC_API_KEY`), never in files. You will practise this rule in the Security week.
3. Read the repo's [GETTING_STARTED.md][gs]. You can optionally use the `/claude-certification` tutor skill in a Claude Code session opened at the repo root.
4. Take the [CCDV-F diagnostic][diag]: 16 questions, 30 min. Record your score per domain.
5. Read lesson [00 – Certification strategy][L00].
6. Create `error-log.md` with these columns: *date · question/topic · my answer · correct · why I was wrong · doc link*. You will use it every week.

> Do your own work outside the repo, for example in `D:\GitHub\InterviewPrep\ccdv-f-labs\`. The repo convention is `learning-artifacts/claude/`. Either way, do **not** edit the reference `outputs/` folders.

---

## 4. Weekly study schedule

### Standard week template (~7.5 h)

| Slot | Time | Activity |
|---|---|---|
| Mon | 60 min | Read the lesson `docs/en.md` and the official docs pages for the week. |
| Tue | 60 min | Read and run the lesson lab: `python code/main.py`, then `python -m unittest discover -s code/tests -v`. |
| Wed | 60 min | Second lesson or official docs, and make flashcards. |
| Thu | 60 min | Lesson quiz(zes) in `quiz.json`, then update the error log. |
| Sat | 2.5 h | **Hands-on exercise** (below). This is the most important block of the week. |
| Sun | 60 min | Spaced revision: re-test last week's flashcards and errors, and write a 1-page summary. |

Rest days: Friday, plus any day you work late. If you miss a weekday slot, move it to Sunday. Never skip the Saturday build.

---

### Week 1 — LLM & Model Fundamentals · *D5 (16.8%) + part of D2 App Design*

- **Lessons:** [01 Product & model landscape][L01] → [02 Model selection & token economics][L02]
- **Official docs:** [Models overview][d-models], [Choosing a model][d-choose], [Pricing][d-pricing], [Context windows][d-ctx], [Extended thinking][d-think], [Steering thinking (adaptive)][d-adaptive], [Effort][d-effort], [Token counting][d-tokens]
- **Key concepts:** tokens; context window; sampling and non-determinism; how Opus, Sonnet and Haiku differ; thinking modes and effort; cost = input + output tokens × price; breaking changes between model versions.
- **🔧 Exercise — model router and cost calculator**
  1. Write `cost.py`. Given input tokens, output tokens and a model tier, it returns the cost using the official pricing page.
  2. Use the token-counting endpoint on 3 prompts of different sizes and record the counts.
  3. Write a routing rule: classification → smallest tier; complex reasoning → highest tier. Justify each choice in writing.
  - ✅ *Done when:* you can explain one quality, latency and cost trade-off for each tier without notes.

### Week 2 — Prompt & Context Engineering · *D6 (11%) + D5 Cost/Caching*

- **Lessons:** [03 Prompting & task decomposition][L03] → [04 Context, memory & caching][L04] → [05 Output evaluation & validation][L05]
- **Official docs:** [Prompt engineering overview][d-prompt], [Prompt caching][d-cache], [Compaction][d-compact], [Effective context engineering (Anthropic Engineering)][e-context]
- **Key concepts:** system vs user placement; few-shot examples; output constraints; input sanitisation; context drift and bloat; pruning; compaction; isolating context with subagents; cache breakpoints and when caching pays off.
- **🔧 Exercise — prompt caching experiment**
  1. Build a prompt with a large, stable system block (for example, a 5k-token policy document) and a small user question that varies.
  2. Call it 5 times with `cache_control` on the stable block. Log `usage` (cache creation vs cache read tokens) and the latency of each call.
  3. Move the variable text above the cached block and observe that the cache misses.
  - ✅ *Done when:* you can say why a prefix must be stable for caching to work, and you have calculated the cost saved.

### Week 3 — Claude API Mechanics & Technical Fundamentals · *D2 API (6.8%) + D5 Tech (6.1%) + D6 Output*

- **Lessons:** [08 Messages API & lifecycle][L08] → [09 Structured output & defensive parsing][L09]
- **Official docs:** [Messages API][d-messages], [Streaming][d-stream], [Vision][d-vision], [Batch processing][d-batch], [Files API][d-files], [Structured outputs][d-structured], [Errors][d-errors], [Rate limits][d-rate], [Claude on Vertex AI][d-vertex] (third-party vendor example)
- **Key concepts:** content blocks; `stop_reason` values; multi-turn state; streaming events; when to use realtime or batch (batch runs within a 24 h window at lower cost); image input; retrying with backoff on 429/529 errors; using an SDK instead of raw REST; websocket basics (gap, see §8).
- **🔧 Exercise — streaming CLI chat**
  1. Build a CLI chat that keeps conversation history, streams tokens and prints `stop_reason` and `usage`.
  2. Add retries with exponential backoff, and make every `stop_reason` path explicit, including `max_tokens`.
  3. Submit a 20-item Message Batch (for example, summarise 20 short texts) and poll for the results.
  4. Ask for structured JSON output, then validate it with `pydantic` or `jsonschema`. Reject and re-ask when the output is invalid.
  - ✅ *Done when:* you can whiteboard the request → response → tool → continuation state machine.

### Week 4 — Tools & MCP · *D8 (10.6%)*

- **Lessons:** [10 Tool use & agentic loops][L10] → [11 MCP server design][L11] → *(supplementary)* [18 Tool contracts & progressive discovery][L18]
- **Official docs:** [Tool use overview][d-tools], [MCP intro][mcp-intro], [Build an MCP server][mcp-build], [MCP specification][mcp-spec], [MCP connector][d-mcpconn], [Agent Skills][d-skills], [Writing tools for agents (Anthropic Engineering)][e-tools]
- **Key concepts:** a `tool_use` block is a *proposal* that your code must validate and execute; tool descriptions and schemas; returning errors as tool results; client-side vs server-side tools; approval gates; MCP tools, resources and prompts; stdio vs Streamable HTTP; choosing among built-in tools, custom tools, Skills and MCP.
- **🔧 Exercise — inventory MCP server (mirrors official Sample Q3)**
  1. Write a small MCP server (Python SDK, stdio) with 2 tools (`get_stock`, `reserve_item`) and 1 resource (`catalog`).
  2. Connect it to Claude Code with `claude mcp add` and call it.
  3. Make `reserve_item` return a structured error for an unknown SKU, and check that Claude recovers.
  4. Write half a page on when you would use a Skill or a custom tool for the same need instead.
  - ✅ *Done when:* the server works from Claude Code and you can explain why MCP suits a capability shared across several apps.

### Week 5 — Agents & Workflows · *D1 (14.7%)*

- **Lessons:** [12 Agent SDK & hooks][L12] → *(supplementary)* [16 Multi-agent orchestration][L16] → [17 SDK sessions, subagents & context][L17]
- **Deep dives:** [The agent loop][P-loop], [Anthropic workflow patterns][P-wf], [Agent framework trade-offs (LangGraph)][P-fw]
- **Official docs:** [Agent SDK overview][d-sdk], [Building effective agents (Anthropic Engineering)][e-agents], [Subagents][cc-subagents]
- **Key concepts:** choose a workflow when the path is known and an agent when the plan has to adapt; manager/subagent hierarchies; custom loops vs the SDK; managed agents (self-hosted vs Anthropic-hosted); memory; framework abstractions (Strands, LangGraph, PydanticAI, which you study yourself, see §8).
- **🔧 Exercise — research agent with a subagent and a hook**
  1. Build an agent with the Agent SDK that reads a folder of notes and writes a summary report.
  2. Hand off a "fact-check" step to a subagent with its own isolated context.
  3. Add a `PreToolUse` hook that blocks writes outside `./out/`.
  4. Rebuild the same task as a fixed 3-step workflow (no agent) and compare cost, steps and reliability.
  - ✅ *Done when:* you can defend "workflow vs agent" for 3 made-up scenarios.

> **Mid-point check (end of Week 5):** re-take the [diagnostic][diag] and compare it with Week 0. Re-rate the 25 sub-skills.

### Week 6 — Security & Safety + Debugging · *D7 (8.1%) + D4 (2.6%)*

- **Lessons:** [13 Application security & secrets][L13] → [14 Evals, testing, debugging & observability][L14]
- **Deep dives:** [Prompt injection defence][P-inj], [Eval-driven agent development][P-eval]
- **Official docs:** [Mitigate jailbreaks][d-jail], [Hooks (Claude Code)][cc-hooks], [IAM / permissions][cc-iam], [Develop tests][d-tests], [Errors][d-errors]
- **Key concepts:** treat retrieved content as untrusted and keep it separate from instructions (official Sample Q2); least privilege; layered guardrails; PII handling; managing secrets across dev and prod; using hooks for *deterministic* controls; classifying errors and choosing a recovery; using traces to tell integration-layer faults from model faults.
- **🔧 Exercise — attack and defend your Week 3 or Week 5 app**
  1. Feed it a document that contains hidden "ignore previous instructions" text, and record what happens.
  2. Add separation (XML-tagged untrusted content), allow-listed tools and a hook that blocks the sensitive action.
  3. Write 10 eval cases (5 normal, 5 adversarial) and a tiny runner that reports pass/fail.
  4. Break the integration on purpose (wrong key, malformed tool result, timeout) and write down how each one shows up in the logs or trace.
  - ✅ *Done when:* every adversarial case is blocked by code, not by prompt wording.

### Week 7 — Claude Code, Configuration & Engineering Foundations · *D3 (3.1%) + D2 Config (4.1%), SE (7.4%), Requirements (3.4%), Life Cycle (2.8%)*

- **Lessons:** [15 Claude Code for dev teams][L15] → *(supplementary)* [19 Memory, rules, Skills & CI][L19] → [22 Business discovery & requirements][L22]
- **Official docs:** [Claude Code overview][cc-overview], [Memory / CLAUDE.md][cc-memory], [Settings][cc-settings], [Slash commands][cc-slash], [Skills][cc-skills], [Plugins][cc-plugins], [Headless / programmatic use][cc-headless], [MCP in Claude Code][cc-mcp]
- **Key concepts:** CLAUDE.md hierarchy and precedence; scopes in settings.json; Rules, Skills, Commands and Agents; sessions; headless and streaming output; auto mode; `/init`; pinning model versions; versioning prompts; plugin dependencies; code review and refactoring with Claude Code; turning requirements into a spec.
- **🔧 Exercise — team-ready repo**
  1. In a practice repo, run `/init`. Then write a project CLAUDE.md, a user-level preference and a project `settings.json` with permission rules.
  2. Write one custom Skill (for example, "review-migration") and one slash command.
  3. Run Claude Code headless in a script to review a diff, and capture the JSON output.
  4. Pin a model version in config, and keep your prompts in a versioned `prompts/` folder with a changelog.
  5. Write a 1-page requirements brief (functional + infrastructure) for your capstone.
  - ✅ *Done when:* you can say which config wins when user, project and local settings conflict, and you can cite the doc for it.

### Week 8 — Capstone: ship a Claude application you can defend · *all domains, especially D2 App Design (8.6%)*

- **Lesson:** [30 Developer application capstone][L30], which ships with a [reference architecture][L30-arch], [runbook][L30-run] and [eval plan][L30-eval]
- **Build** one app that brings Weeks 1–7 together. This is what the official guide asks for: "Build and operate at least one Claude application that exercises the API, integrates one or more tools, applies basic prompt and context engineering, and includes simple security and evaluation practices."
  - API with streaming and retries · ≥1 custom tool or your Week 4 MCP server · a structured output contract with validation · caching where it pays · a hook or other deterministic guardrail · secrets kept in env vars · 10+ eval cases · README with an architecture decision and a runbook.
- ✅ *Done when:* lesson 30's tests pass, your app runs end to end, and you can explain each design choice in 2 minutes.

### Week 9 — Mock tests & remediation

See §6 for full details.

| Day | Activity |
|---|---|
| Mon–Tue | Review the error log and redo all lesson quizzes you got wrong. |
| **Sat** | **Mock A**: the repo's [full CCDV-F mock][mock] (53 items, 120 min, timed, closed book). |
| Sun + next Mon–Wed | Analyse Mock A by domain, then re-study your 3 weakest sub-skills (docs + lab). |
| Thu | **Mock B**: the self-assembled weighted mock (see §6). |

### Week 10 — Final revision & exam

| Day | Activity |
|---|---|
| Mon–Wed | Rapid revision: 1-page cheat sheets per domain from memory, then check them against the docs. |
| Thu | Light review of the error log only. Complete the readiness checklist (§9). |
| Fri | Rest. Check your ID and the test-centre route, or run the online-proctor system test. |
| **Sat/Sun** | **Exam.** |

**Buffer week (optional):** use it for missed weeks, holidays or a weak Mock B. Reschedule the exam **at least 24 h** before the appointment so you do not lose the fee.

---

## 5. Revision strategy

| Technique | When | How |
|---|---|---|
| **Error log** | Every quiz and mock | Write down *why* the wrong option was tempting, not just what the right answer is. |
| **Spaced repetition** | Every Sunday | Re-test topics from 1 and 3 weeks ago. Use Anki or a simple flashcard sheet. |
| **Teach-back** | End of each week | Explain the week's main decision aloud in 2 minutes (for example, "batch vs realtime" or "workflow vs agent"). |
| **Decision tables** | Weeks 1–8 | Keep a one-page "if X, choose Y because Z" table. Exam items are scenario judgements, not trivia. |
| **Domain cheat sheets** | Week 10 | One page per domain, written from memory first. |
| **Weight-proportional review** | Weeks 9–10 | Spend about a third of revision time on D2 Applications & Integration. |

**How to answer the exam's question style** (based on the official samples): pick the answer that **enforces** a control in code or architecture over one that **asks** the model nicely. Pick the option that fits the stated constraint (cost, latency, reuse) over the "most powerful" option. Watch for distractors such as "use a bigger model" or "change the temperature".

---

## 6. Mock tests

| Mock | Source | Items | When | Notes |
|---|---|--:|---|---|
| Diagnostic | Repo [ccdv-f/diagnostic.json][diag] | 16 | Week 0 and Week 5 | 30 min. Covers all 8 domains. |
| Official samples | [Exam Guide §8][guide] | 3 | Week 0 and Week 10 | The only official practice items. They show the item style. |
| Lesson quizzes | `quiz.json` in each lesson | 6 per lesson | Every week | 15 route lessons × 6 = 90 items |
| **Mock A** | Repo [ccdv-f/mock-01.json][mock] | **53** | Week 9 Sat | Timed, 120 min. 34 single-answer + 19 multi-answer items. The domain mix follows the blueprint (see below). |
| **Mock B** | Self-assembled | **53** | Week 9 Thu | Pull lesson-quiz items you have *not* recently reviewed, weighted per the table below. Add items you write yourself from the error log. Run it timed. |
| Cross-practice | Repo [CCAR-F mock][mock-ccarf] | 60 | Optional | Architect Foundations covers agents, tools/MCP, Claude Code and context. It is good extra practice for D1, D3, D6 and D8. |

**Mock A's domain mix matches the blueprint**, which I checked by counting `domain` tags. That makes its per-domain scores a useful signal:

| Domain | Weight | Target items (×53) | Mock A items | Mock B target |
|---|--:|--:|--:|--:|
| Applications & Integration | 33.1% | 17.5 | 17 | 17 |
| Model Selection & Optimization | 16.8% | 8.9 | 9 | 9 |
| Agents & Workflows | 14.7% | 7.8 | 8 | 8 |
| Prompt & Context Engineering | 11.0% | 5.8 | 6 | 6 |
| Tools & MCPs | 10.6% | 5.6 | 6 | 6 |
| Security & Safety | 8.1% | 4.3 | 4 | 4 |
| Claude Code | 3.1% | 1.6 | 2 | 2 |
| Eval, Testing & Debugging | 2.6% | 1.4 | 1 | 1 |

**Scoring rules:** score multi-answer items as an exact set (no partial credit), the same way the repo does. **Practice percentages are not scaled scores.** As a study heuristic only, aim for **≥ 80% overall and ≥ 70% in every domain** on both mocks before you sit the exam. If you fall short, use the buffer week.

---

## 7. Hands-on exercise summary

| Week | Exercise | Domains |
|---|---|---|
| 1 | Model router and cost calculator | D5 |
| 2 | Prompt caching experiment | D5, D6 |
| 3 | Streaming CLI chat + Batch + validated JSON | D2, D5, D6 |
| 4 | Inventory MCP server + choosing between Skill, tool and MCP | D8, D2 |
| 5 | SDK agent + subagent + hook vs a fixed workflow | D1, D7 |
| 6 | Prompt-injection attack/defence + 10-case evals + fault injection | D7, D4 |
| 7 | Team-ready Claude Code repo (CLAUDE.md, settings, Skill, headless, pinning) | D3, D2 |
| 8 | Capstone app with runbook and eval plan | All |

---

## 8. Gaps in the existing study materials

I found these by comparing the official blueprint text with the CCDV-F route lessons, using keyword searches across `lessons/*/docs/en.md`. Fill each gap with the official docs or by building something yourself.

| # | Gap | Blueprint skill (weight) | Evidence | How to close it |
|--:|---|---|---|---|
| 1 | **Agent frameworks** (Strands, LangGraph, PydanticAI) are named in the blueprint but not taught | Agent Patterns & Frameworks (4.9%) | 0 mentions in the certification lessons. LangGraph appears only in repo phases [11/16][P-lg] and [11/17][P-fw]. Strands and PydanticAI appear nowhere in the repo. | Read [11/17 framework trade-offs][P-fw]. Read each framework's quick-start. Rebuild the Week 5 workflow in one framework (1–2 h). Focus on *when* to use an abstraction framework rather than API details. |
| 2 | **Software Engineering Foundations** has no dedicated lesson | SE Foundations (7.4%), the 2nd-largest sub-skill | "Refactor" has 0 matches in the certification lessons. Version control appears only in 15, 18 and 19. Async appears only in 08 and 12. | Week 7: use Claude Code for a small and a large refactor behind a PR and review it. Practise async SDK calls (`AsyncAnthropic`). Revisit REST status codes and idempotency. |
| 3 | **Websockets** | Technical Fundamentals (6.1%) | 0 matches in the certification lessons. Only appears in phases [11/13 production app][P-prod] and 13/14 MCP apps. | Learn how websockets differ from HTTP streaming (SSE). Know which one Claude streaming uses (see the [streaming docs][d-stream]). |
| 4 | **Requirements & Systems Life Cycle** are taught only outside the CCDV-F route | Requirements (3.4%) + Life Cycle (2.8%) | Lessons [22][L22] and [28][L28] cover them but are not in the CCDV-F track. | This plan adds lesson 22 in Week 7. Skim lesson 28. |
| 5 | **Claude Code depth** (settings.json, Skills, CI) sits outside the route | Claude Code Op (3.1%) + Config Mgmt (4.1%) | `settings.json` appears only in lesson [19][L19], which is not in the CCDV-F route. | This plan adds lesson 19 in Week 7. Use the official [settings][cc-settings] and [memory][cc-memory] docs as the source of truth. |
| 6 | Multi-agent, sessions and tool-contract lessons sit outside the route | Agent Architecture (4.5%), Context Eng. (3.8%), Tool Impl. (4.4%) | Lessons [16][L16], [17][L17] and [18][L18] are not in the track. | This plan adds them as supplementary lessons in Weeks 4–5. |
| 7 | **Little practice material** | All | Only 1 full CCDV-F mock and a 16-item diagnostic. The official practice exam is retired. The community site's CCDV-F mock is "Coming soon". | Mock B (§6), extra items you write yourself, and the CCAR-F mock for overlapping domains. |
| 8 | Light coverage of **PII handling** and **token budgeting / cache checkpointing** | AI App Security (3.2%), Cost & Token Mgmt (2.8%) | "Token budget" has 0 matches. PII is covered only briefly (lesson 03). | Weeks 2 and 6 exercises plus the [prompt caching][d-cache] and [token counting][d-tokens] docs. |
| 9 | **Product details may be out of date** | Every product-mechanics item | The repo was last verified 2026-08-09, and its own ledger warns about drift. | Check exact flags, fields and model names against the official docs (§10), not the lessons. |

The official CCDV-F prep path on the Partner Academy may cover some of these gaps. Its course list was not readable without a partner sign-in, so this plan does not rely on it.

---

## 9. Final readiness checklist

Tick every item before exam week. If you cannot tick it, use the buffer week.

**Knowledge**
- [ ] I rated myself ≥ 4/5 on all 25 sub-skills, or ≥ 3/5 on those under 2%.
- [ ] I can explain without notes: batch vs realtime; workflow vs agent; Skill vs custom tool vs MCP vs built-in tool; system vs user prompt placement; caching prerequisites; stdio vs Streamable HTTP; the CLAUDE.md and settings precedence order.
- [ ] I can list the main `stop_reason` values and what my code should do for each.
- [ ] I can describe 3 prompt-injection defences that are **enforced in code**.
- [ ] I can give one reason each to choose Opus, Sonnet or Haiku, and name the risk of upgrading the model version.

**Hands-on**
- [ ] All 8 weekly exercises are done, and the code runs.
- [ ] The capstone app runs end to end, with tests, evals, a hook or guardrail, and a runbook.
- [ ] I have written and run an MCP server and connected it to Claude Code.
- [ ] I have configured Claude Code (CLAUDE.md, settings.json, a Skill, headless mode).

**Practice scores (heuristic, not scaled)**
- [ ] Mock A ≥ 80% overall, every domain ≥ 70%
- [ ] Mock B ≥ 80% overall, every domain ≥ 70%
- [ ] Every item in the error log has been re-tested and answered correctly.
- [ ] I finished each mock with ≥ 15 min left for review.

**Logistics**
- [ ] I confirmed eligibility (Claude Partner Network org + partner email domain).
- [ ] Booked through the Anthropic Partner Academy → Pearson VUE. The name matches my government ID exactly.
- [ ] Online: system test passed, desk cleared, no second monitor. Test centre: route planned, arriving early.
- [ ] I know that changes within 24 h forfeit the fee and that the exam is closed book.
- [ ] I set aside about 135 min of uninterrupted time.

---

## 10. Links

### Source materials
- Community guide: [claudecertificationguide.com/ccdv-f][site-ccdvf] · [Architect Foundations curriculum][site-learn]
- Local repo: [certifications/claude README][readme] · [GETTING_STARTED][gs] · [CCDV-F track][track] · [Diagnostic][diag] · [Mock 01][mock] · [Official blueprint map][bp-map] · [Source verification ledger][ledger]
- Online: [GitHub repo][gh-repo] · [Website certification page][web-cert]

### Official Anthropic: certification
- [CCDV-F Exam Guide v1.0 (PDF)][guide] · [Certification FAQ][faq] · [Exam prep courses][prep] · [CCDV-F prep path][prep-path] · [Anthropic Academy][academy] · [Pearson VUE – Anthropic][pvue]

### Official Anthropic: Claude Platform docs
- [Docs home][d-home] · [Messages API][d-messages] · [Models overview][d-models] · [Choosing a model][d-choose] · [Pricing][d-pricing] · [Context windows][d-ctx] · [Token counting][d-tokens]
- [Prompt engineering][d-prompt] · [Prompt caching][d-cache] · [Compaction][d-compact] · [Extended thinking][d-think] · [Steering thinking][d-adaptive] · [Effort][d-effort]
- [Streaming][d-stream] · [Vision][d-vision] · [Batch processing][d-batch] · [Files API][d-files] · [Structured outputs][d-structured] · [Errors][d-errors] · [Rate limits][d-rate] · [Claude on Vertex AI][d-vertex]
- [Tool use][d-tools] · [MCP connector][d-mcpconn] · [Agent Skills][d-skills] · [Agent SDK][d-sdk] · [Develop tests][d-tests] · [Mitigate jailbreaks][d-jail]

### Official Anthropic: Claude Code docs
- [Overview][cc-overview] · [Memory (CLAUDE.md)][cc-memory] · [Settings][cc-settings] · [Hooks][cc-hooks] · [Slash commands][cc-slash] · [Subagents][cc-subagents] · [Skills][cc-skills] · [Plugins][cc-plugins] · [Headless][cc-headless] · [MCP][cc-mcp] · [IAM][cc-iam]

### Model Context Protocol
- [Introduction][mcp-intro] · [Build a server][mcp-build] · [Specification][mcp-spec]

### Anthropic Engineering & cookbook
- [Building effective agents][e-agents] · [Writing tools for agents][e-tools] · [Effective context engineering][e-context] · [Claude cookbooks][cookbook]

---

*This is an independent study plan. It is not affiliated with or endorsed by Anthropic. Exam facts come from the official Exam Guide v1.0 (July 2026) and FAQ, as checked on 2026-10-13. Check them again before you register, because the program can change.*

<!-- ===== Link references ===== -->
[site-ccdvf]: https://claudecertificationguide.com/ccdv-f
[site-learn]: https://claudecertificationguide.com/learn
[guide]: https://everpath-course-content.s3-accelerate.amazonaws.com/instructor%2F6nizmqk8tpzpfjvt6qmmav7rh%2Fpublic%2F1783542875%2FClaude+Certified+Developer+%E2%80%93+Foundations+Exam+Guide.pdf
[faq]: https://anthropic-partners.skilljar.com/page/faq-certifications
[prep]: https://anthropic-partners.skilljar.com/page/claude-certification-exam-prep-courses
[prep-path]: https://anthropic-partners.skilljar.com/path/claude-certified-developer-foundations
[academy]: https://anthropic.skilljar.com/
[pvue]: https://www.pearsonvue.com/us/en/anthropic.html
[gh-repo]: https://github.com/rohitg00/ai-engineering-from-scratch
[web-cert]: https://aiengineeringfromscratch.com/certifications

[readme]: ../../ai-engineering-from-scratch/certifications/claude/README.md
[gs]: ../../ai-engineering-from-scratch/certifications/claude/GETTING_STARTED.md
[track]: ../../ai-engineering-from-scratch/certifications/claude/tracks/ccdv-f.json
[diag]: ../../ai-engineering-from-scratch/certifications/claude/assessments/ccdv-f/diagnostic.json
[mock]: ../../ai-engineering-from-scratch/certifications/claude/assessments/ccdv-f/mock-01.json
[mock-ccarf]: ../../ai-engineering-from-scratch/certifications/claude/assessments/ccar-f/mock-01.json
[bp-map]: ../../ai-engineering-from-scratch/certifications/claude/research/official-blueprint-map.md
[ledger]: ../../ai-engineering-from-scratch/certifications/claude/research/source-verification-ledger.md

[L00]: ../../ai-engineering-from-scratch/certifications/claude/lessons/00-certification-strategy/docs/en.md
[L01]: ../../ai-engineering-from-scratch/certifications/claude/lessons/01-claude-product-and-model-landscape/docs/en.md
[L02]: ../../ai-engineering-from-scratch/certifications/claude/lessons/02-model-selection-and-token-economics/docs/en.md
[L03]: ../../ai-engineering-from-scratch/certifications/claude/lessons/03-prompting-and-task-decomposition/docs/en.md
[L04]: ../../ai-engineering-from-scratch/certifications/claude/lessons/04-context-knowledge-memory-and-caching/docs/en.md
[L05]: ../../ai-engineering-from-scratch/certifications/claude/lessons/05-output-evaluation-and-validation/docs/en.md
[L08]: ../../ai-engineering-from-scratch/certifications/claude/lessons/08-messages-api-and-application-lifecycle/docs/en.md
[L09]: ../../ai-engineering-from-scratch/certifications/claude/lessons/09-structured-output-and-defensive-parsing/docs/en.md
[L10]: ../../ai-engineering-from-scratch/certifications/claude/lessons/10-tool-use-and-agentic-loops/docs/en.md
[L11]: ../../ai-engineering-from-scratch/certifications/claude/lessons/11-mcp-server-design-and-integration/docs/en.md
[L12]: ../../ai-engineering-from-scratch/certifications/claude/lessons/12-claude-agent-sdk-and-hooks/docs/en.md
[L13]: ../../ai-engineering-from-scratch/certifications/claude/lessons/13-application-security-and-secrets/docs/en.md
[L14]: ../../ai-engineering-from-scratch/certifications/claude/lessons/14-evals-testing-debugging-and-observability/docs/en.md
[L15]: ../../ai-engineering-from-scratch/certifications/claude/lessons/15-claude-code-for-development-teams/docs/en.md
[L16]: ../../ai-engineering-from-scratch/certifications/claude/lessons/16-multi-agent-orchestration-and-delegation/docs/en.md
[L17]: ../../ai-engineering-from-scratch/certifications/claude/lessons/17-agent-sdk-sessions-subagents-and-context/docs/en.md
[L18]: ../../ai-engineering-from-scratch/certifications/claude/lessons/18-tool-contracts-errors-and-progressive-discovery/docs/en.md
[L19]: ../../ai-engineering-from-scratch/certifications/claude/lessons/19-claude-code-memory-rules-skills-and-ci/docs/en.md
[L22]: ../../ai-engineering-from-scratch/certifications/claude/lessons/22-business-discovery-requirements-and-slas/docs/en.md
[L28]: ../../ai-engineering-from-scratch/certifications/claude/lessons/28-stakeholder-communication-adrs-and-lifecycle/docs/en.md
[L30]: ../../ai-engineering-from-scratch/certifications/claude/lessons/30-developer-application-capstone/docs/en.md
[L30-arch]: ../../ai-engineering-from-scratch/certifications/claude/lessons/30-developer-application-capstone/outputs/architecture.md
[L30-run]: ../../ai-engineering-from-scratch/certifications/claude/lessons/30-developer-application-capstone/outputs/runbook.md
[L30-eval]: ../../ai-engineering-from-scratch/certifications/claude/lessons/30-developer-application-capstone/outputs/eval-plan.json

[P-prompt]: ../../ai-engineering-from-scratch/phases/11-llm-engineering/01-prompt-engineering
[P-struct]: ../../ai-engineering-from-scratch/phases/11-llm-engineering/03-structured-outputs
[P-loop]: ../../ai-engineering-from-scratch/phases/14-agent-engineering/01-the-agent-loop
[P-wf]: ../../ai-engineering-from-scratch/phases/14-agent-engineering/12-anthropic-workflow-patterns
[P-lg]: ../../ai-engineering-from-scratch/phases/11-llm-engineering/16-langgraph-state-machines
[P-fw]: ../../ai-engineering-from-scratch/phases/11-llm-engineering/17-agent-framework-tradeoffs
[P-prod]: ../../ai-engineering-from-scratch/phases/11-llm-engineering/13-production-app
[P-inj]: ../../ai-engineering-from-scratch/phases/14-agent-engineering/27-prompt-injection-defense
[P-eval]: ../../ai-engineering-from-scratch/phases/14-agent-engineering/30-eval-driven-agent-development

[d-home]: https://platform.claude.com/docs/en/home
[d-messages]: https://platform.claude.com/docs/en/api/messages
[d-models]: https://platform.claude.com/docs/en/models/overview
[d-choose]: https://platform.claude.com/docs/en/about-claude/models/choosing-a-model
[d-pricing]: https://platform.claude.com/docs/en/about-claude/pricing
[d-ctx]: https://platform.claude.com/docs/en/build-with-claude/context-windows
[d-tokens]: https://platform.claude.com/docs/en/build-with-claude/token-counting
[d-prompt]: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview
[d-cache]: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
[d-compact]: https://platform.claude.com/docs/en/build-with-claude/compaction
[d-think]: https://platform.claude.com/docs/en/build-with-claude/extended-thinking
[d-adaptive]: https://platform.claude.com/docs/en/build-with-claude/thinking-steering-and-cost
[d-effort]: https://platform.claude.com/docs/en/build-with-claude/effort
[d-stream]: https://platform.claude.com/docs/en/build-with-claude/streaming
[d-vision]: https://platform.claude.com/docs/en/build-with-claude/vision
[d-batch]: https://platform.claude.com/docs/en/build-with-claude/batch-processing
[d-files]: https://platform.claude.com/docs/en/build-with-claude/files
[d-structured]: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
[d-errors]: https://platform.claude.com/docs/en/api/errors
[d-rate]: https://platform.claude.com/docs/en/api/rate-limits
[d-vertex]: https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai
[d-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
[d-mcpconn]: https://platform.claude.com/docs/en/agents-and-tools/mcp-connector
[d-skills]: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
[d-sdk]: https://code.claude.com/docs/en/agent-sdk/overview
[d-tests]: https://platform.claude.com/docs/en/test-and-evaluate/develop-tests
[d-jail]: https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks

[cc-overview]: https://code.claude.com/docs/en/overview
[cc-memory]: https://code.claude.com/docs/en/memory
[cc-settings]: https://code.claude.com/docs/en/settings
[cc-hooks]: https://code.claude.com/docs/en/hooks
[cc-slash]: https://code.claude.com/docs/en/slash-commands
[cc-subagents]: https://code.claude.com/docs/en/sub-agents
[cc-skills]: https://code.claude.com/docs/en/skills
[cc-plugins]: https://code.claude.com/docs/en/plugins
[cc-headless]: https://code.claude.com/docs/en/headless
[cc-mcp]: https://code.claude.com/docs/en/mcp
[cc-iam]: https://code.claude.com/docs/en/iam

[mcp-intro]: https://modelcontextprotocol.io/docs/getting-started/intro
[mcp-build]: https://modelcontextprotocol.io/docs/develop/build-server
[mcp-spec]: https://modelcontextprotocol.io/specification/latest

[e-agents]: https://www.anthropic.com/engineering/building-effective-agents
[e-tools]: https://www.anthropic.com/engineering/writing-tools-for-agents
[e-context]: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
[cookbook]: https://github.com/anthropics/claude-cookbooks
