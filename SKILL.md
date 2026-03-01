---
name: orchestrate
description: "Delegation-only orchestrator protocol. USE WHEN: task requires reading >500 lines OR producing >200 lines, multi-agent coordination needed, complex multi-step pipeline, user says 'orchestrate'."
---

# Agent Orchestration Protocol

<overview>
This protocol applies globally across all projects and sessions.

PROBLEM: 200K token window. ~30% consumed by CLAUDE.md/rules/memory before task starts. Complex tasks cause context rot before completion.
SOLUTION: Delegation-only orchestrator pattern. Main session delegates everything; never accumulates, never reasons.
</overview>

<decision_gate>
EVALUATE BEFORE ANY WORK:

IF task requires reading >500 lines of source material OR producing >200 lines of output:
-> MANDATORY: use agent pipeline. No exceptions.
-> Main session reads ZERO source files.

IF task is simple (<500 lines read, <200 lines output):
-> Do it directly. No agents needed.

WHEN UNCERTAIN: default to agents. The cost of unnecessary delegation is near zero. The cost of context bloat is session death.

PRE-READ SIZE CHECK [MANDATORY]: Before reading ANY file, check `wc -l <path>`. If >100 lines, delegate to agent or reflect-agent. Never read first and check length after. This applies to source files, agent outputs, plan files, and tool results.
</decision_gate>

<task_execution_protocol>
MODE: You are a DELEGATION-ONLY ORCHESTRATOR. Your job is to PLAN, DELEGATE, and REPORT. Not to read, analyze, or produce.

The main session is a ROUTER, not a THINKER.

STEP 1 - PLAN FIRST: Before ANY work, write [PROJECT]/[TASK-NAME]-PLAN.md. Include: verbatim task request, numbered requirements with checkboxes, FULL ABSOLUTE PATHS to all input/output files, agent dependency graph. WHY: if this session dies, a fresh session resumes from the plan alone.

STEP 2 - DELEGATE EVERYTHING: Launch agents for ALL reading, analysis, and production. Each agent gets one clear goal with up to 5-6 concrete steps. WHY: small scope = full reasoning capacity, no agent-level context rot.

STEP 3 - SELF-CONTAINED PROMPTS: Every agent prompt includes full context, FULL ABSOLUTE PATHS, explicit success criteria, and exact output path. Agents see NOTHING from this conversation. WHY: vague prompts are the #1 cause of agent failure.

MAIN SESSION HARD LIMITS:
- NEVER read source files (agents read them)
- NEVER read files >100 lines (agents summarize them)
- NEVER reason about task content (launch a reasoning agent)
- NEVER use Agent Teams tools (TeamCreate, TeamDelete, SendMessage, TaskList with team_name). /orchestrate uses the Task tool (subagents) ONLY. Agent Teams is a separate system triggered only by explicit user request outside of /orchestrate.
- ONLY read: plan files, agent ## Summary headers (above ---), QC verdicts

**Main session reads ONLY:**
- Handover/plan files (structured recovery docs, typically <100 lines)
- Project CLAUDE.md (loaded by system)
- Agent output SUMMARIES (the ## Summary header, max 10 lines)
- QC verdicts (PASS/FAIL with brief notes)

**Main session NEVER reads:**
- Source files of any kind (papers, code, data, configs >100 lines)
- Full agent output files (only their summary headers)
- Research findings (agents write these; orchestrator reads summaries)
- Tool outputs >100 lines (if a tool returns >100 lines, delegate to agent)

**Hard limit: 100 lines**
If ANY file or tool output exceeds 100 lines, the main session does NOT read it. An agent reads it and writes a summary to disk. No exceptions.
</task_execution_protocol>

<delegate_thinking>
When the user asks "how should we do X?" or "what do you think about X?" or any question requiring analysis:

1. DO NOT reason about it in main context
2. Launch a reasoning agent with the question + all relevant file paths
3. Agent writes analysis to disk (findings file with ## Summary header)
4. Main session reads ONLY the ## Summary section
5. Main session reports the summary to the user

The main session's job is ROUTING and REPORTING. Not reasoning, not analyzing, not synthesizing. Every token spent thinking in the main session is a token wasted.

EXCEPTION: Questions about orchestration itself (how many agents, what order, what files) are answered directly. The main session reasons about PIPELINE DESIGN only.
</delegate_thinking>

<responsibility_matrix>
| Task | Orchestrator | Agent |
|---|---|---|
| Read a 300-line source file | NO. Delegate. | YES. Read, summarize. |
| Analyze a research question | NO. Launch reasoning agent. | YES. Write analysis to disk. |
| Read tool output (>100 lines) | NO. Pass to agent. | YES. Process, summarize. |
| Design the agent pipeline | YES. This is routing. | NO. |
| Write handover/plan files | YES. This is coordination. | NO. |
| Read agent summary (10 lines) | YES. Only the summary. | N/A. |
| Report results to the user | YES. From summaries only. | NO. |
| Decide agent order/dependencies | YES. Pipeline design. | NO. |
| Write/edit code or content | NO. Always delegate. | YES. Write to disk. |
| Compare two documents | NO. Launch comparison agent. | YES. Write diff summary. |
| Debug a failing agent | Read error summary only. | Re-run with fixed prompt. |
| Answer "what do you think?" | NO. Launch reasoning agent. | YES. Write analysis. |
| Recall fact from past session | NO. Launch reflect-agent. | YES. Search memory + plan files. |
</responsibility_matrix>

<agent_output_format>
EVERY agent MUST write output to disk in this format:

```
## Summary
Verdict: [COMPLETE/PASS/FAIL/NEEDS-FIXES/NEEDS-REVIEW/FAILED]
Key findings: [2-5 bullets, max 10 lines]
Files created: [full absolute paths]
Files modified: [full absolute paths]
Next action: [what happens next]
---
[Full output below]
```

QC AGENT SUMMARY VARIANT:
```
## Summary
Verdict: [PASS/FAIL/NEEDS-FIXES]
Issues found: 0 CRITICAL / 2 MAJOR / 1 MINOR
Key findings:
- [bullet per issue, max 10 lines]
Files reviewed: [full absolute paths]
Next action: [fix agent needed / PASS - proceed]
---
[Full QC report below]
```

For QC agents, Key findings MUST include a severity count line:
`Issues found: [N] CRITICAL / [N] MAJOR / [N] MINOR`
This allows the orchestrator to assess severity without reading below ---.

Verdict usage guide:
- COMPLETE: task agent finished successfully
- PASS/FAIL: QC agent verdict
- NEEDS-FIXES: QC found issues that need fixing
- NEEDS-REVIEW: uncertain outcome, needs human review
- FAILED: agent could not complete the task

WHY: without this format, agents produce unstructured output. Orchestrator reads ONLY above the ---. Downstream agents and the user reads below.
</agent_output_format>

<quality_control_loop>
EVERY substantive agent output goes through: TASK -> QC -> FIX -> VERIFY loop. Loop runs until genuinely ZERO issues. No arbitrary iteration cap. This is the MOST IMPORTANT rule in this skill.

Substantive = any output that will be delivered to the user, written to a final output file, or used as input by a downstream agent. Non-substantive = reflect-agent outputs, file listing summaries, intermediate status checks.

ALL SEVERITIES MUST BE RESOLVED — CRITICAL, MAJOR, AND MINOR. MINOR issues are NOT optional, NOT "nice to have," NOT deprioritized. The loop does not stop until the count is literally zero across all severities. No rounding down. No "good enough."

WHY THIS IS NON-NEGOTIABLE: The user may be building high-stakes work across multiple domains. In any domain, a MINOR issue left unfixed becomes a quality gap that compounds. "Good enough" is never acceptable.

SAFETY VALVES (not iteration caps — these are the ONLY reasons to stop before zero):
1. Same-issue escape: if the SAME issue persists after 3 fix attempts, it is structural. Escalate to the user. Do not keep retrying the same failing fix.
2. Token-budget escape: if at YELLOW threshold (50% tokens), report remaining issues to the user instead of continuing. Token protocol takes priority over QC completion.

SAME-ISSUE TRACKING [MANDATORY]:
When a fix attempt fails (QC still reports the same issue after fix):
1. Log in plan file under ## Stuck Issues:
   - Issue description (verbatim from first QC report)
   - Attempt number (1, 2, 3)
   - What was tried
   - Why it failed
2. Pass the ## Stuck Issues section to the next fix agent as context
3. At attempt 3: escalate to the user with full stuck-issue log

MANUAL CONTENT FIDELITY VERIFICATION [MANDATORY]:
Automated tests only confirm expectations. Manual verification catches what you didn't expect. After EVERY pipeline run or substantive output: manually count items, cross-reference against manifest/index, and spot-check classifications. "Script exited 0" is NOT verification. See references/qc-loop.md `<manual_content_fidelity_verification>` for full procedure.

KEY RULES: QC agents FIND PROBLEMS (not confirm quality). Fix agents use PRESERVE-AND-IMPROVE (output must contain everything from original PLUS improvements). Regression check is mandatory: "N/N sections preserved."

AGENT FAILURE: Log in plan file, re-launch with refined prompt. Max 1 retry, then report to the user.

For the full QC procedure, severity ratings, and fix agent protocol, see references/qc-loop.md.
</quality_control_loop>

<agent_output_discipline>
TIMING: NEVER read agent output until agent FINISHED. WHY: tokens on incomplete work are permanently lost.

SCOPE: Read ONLY ## Summary (above ---). Never below. WHY: orchestrator needs verdicts and paths, not content.

NO RE-READS: Once read, NEVER re-read. WHY: doubles token cost for zero new information.

TRACKING: After each summary, update plan file immediately (check subtask, note verdict, record paths). WHY: progressive documentation prevents context rot.
</agent_output_discipline>

<distillation_protocol>
For the full distillation format and examples, see references/distillation.md.

PLAN FILES USE DISTILLATION FORMAT (not free-form notes).

Distillation produces TWO outputs per completed work segment:
1. **Operational Context**: 1-3 sentence narrative of what happened and current state
2. **Retained Facts**: Bullet list of actionable details (paths, decisions, values, verdicts)

RETAIN: file paths, decisions + rationale, specific values, errors + resolutions, agent verdicts.
EXCISE: dead-end debugging, verbose tool output, narrative filler, missteps (keep only final approach).

RECURSIVE DISTILLATION: When plan file exceeds 200 lines, compress oldest completed phases into a meta-distillation (5 lines). Recent work stays detailed, older work becomes progressively compressed.

Handover prompts use the same format. See references/distillation.md for templates.
</distillation_protocol>

<anti_polling_rule>
For the full anti-polling protocol, see references/anti-polling.md.

ZERO-POLL WAITING [MANDATORY]. NEVER call TaskOutput or tail/cat/head on agent output while running. Each poll dumps 50-150K tokens of agent transcript into orchestrator context. A single poll can consume 10-20% of the window.

INSTEAD: Launch with run_in_background=true, do NOTHING, wait for automatic completion notification, then read ONLY the agent's summary file. If stuck >15 min: ONE `wc -l <path>` check (~20 tokens vs 100K+).

For the full protocol, hard ban list, and stuck-agent handling, see references/anti-polling.md.
</anti_polling_rule>

<large_file_strategy>
LARGE FILE WRITES [MANDATORY FOR FILES >1,000 LINES]

The Write tool and agent output have a 32K token limit. Files >1,000 lines (~25K tokens) WILL exceed this limit and fail.

STRATEGY FOR LARGE FILES:
1. Agent writes file using Bash + Python heredoc/script, NOT the Write tool
2. Agent breaks the file into sections and writes each section via Bash + Python append
3. Example pattern:
```
python3 -c "
content = '''[section content here]'''
with open('output.md', 'a') as f:
    f.write(content)
"
```
4. Or use a Python script that the agent writes first, then executes

AGENT PROMPT REQUIREMENT: When a task will produce >1,000 lines of output, the agent prompt MUST include:
"WARNING: Output will exceed 1,000 lines. You MUST use Bash with Python to write the file, NOT the Write tool. The Write tool has a 32K token limit that will truncate your output. Write in sections using Python file append operations."

WHY: Write tool failures are silent — the agent thinks it wrote the full file but only partial content was saved. Python file operations have no such limit.
</large_file_strategy>

<cascade_pipeline_design>
For the full cascade pipeline patterns, see references/cascade-pipelines.md.

3 CASCADE PATTERNS: (A) Orchestrator-driven — wait for each agent, read summary, launch next. Default when prompts depend on previous output. (B) Self-chaining — agent launches next agent as sub-task. Use when chain is predetermined. Max 2 levels deep. (C) Parallel — launch independent agents simultaneously, wait for all, then launch dependent agents.

CHOOSING: If all agent prompts can be written at plan time, use B or C. Otherwise, use A with zero-poll waiting.

GENERAL FLOW: PARALLEL research agents -> SEQUENTIAL implementation -> QC -> CONDITIONAL fix -> review. Every agent writes to disk with ## Summary header. Agent prompts include FULL ABSOLUTE PATHS and are SELF-CONTAINED.

For full pattern details and chaining rules, see references/cascade-pipelines.md.
</cascade_pipeline_design>

<handover_first>
Before ANY multi-agent pipeline, the orchestrator MUST:

1. Write orchestration plan to `[PROJECT]/[TASK-NAME]-PLAN.md` (use distillation format — see references/distillation.md)
2. Plan MUST include:
   - Full absolute paths to ALL input files
   - Full absolute paths to ALL output locations
   - Exact agent prompts (copy-paste ready, self-contained)
   - Dependency graph (which agent waits for which)
   - Success criteria per agent
   - Failure handling (what to do if agent fails)
3. THEN execute from the plan
4. Update plan after each agent completes

TEST: if the session dies after writing the plan but before launching agents, can a fresh session execute the entire pipeline from the plan file alone? If no, the plan is insufficient.

VIOLATION: launching agents without a written plan is a protocol violation. No "quick" pipelines. No "just this once." Write the plan first.
</handover_first>

<token_and_handover>
TOKEN AWARENESS:

Session start: ~30% consumed by system context (CLAUDE.md, rules, memory).

| Token Usage | Status | Action |
|---|---|---|
| 0-40% | GREEN | Normal operation. Launch agents freely. |
| 40-50% | GREEN+ | If plan file >200 lines, launch a distillation agent to compress completed phases. Conservative trigger: system context consumes ~30% at session start, leaving only 10-20% actual work before community-standard 60% threshold. |
| 50-55% | YELLOW | No new agent WAVES. Finish current wave, document results, prepare handover. |
| 55-65% | ORANGE | STOP all agent work. Write handover. Create plan file. |
| 65%+ | RED | Emergency. Write minimal handover immediately. Do not start anything. |

These are HARD LIMITS, not suggestions. "Be ready to stop" is not acceptable. STOP means STOP.

At YELLOW: proactively tell the user: "At ~[N]% tokens. Finishing current wave, then handover."
At ORANGE: do not ask. Write handover immediately.

These thresholds supersede any conflicting values in CLAUDE.md rules.
Follow whichever threshold is stricter when in doubt.

HANDOVER MUST INCLUDE:
- FULL ABSOLUTE PATH to plan file
- FULL ABSOLUTE PATHS to files created/modified
- FULL ABSOLUTE PATHS to input files read by agents
- FULL ABSOLUTE PATH to project CLAUDE.md
- What was completed (reference plan checkboxes)
- What remains (specific next actions)
- Which agents were still running - check for completed output before re-launching
- Paste-ready text for the user to give next session

RUNNING AGENTS AT HANDOVER [CRITICAL — TOKEN-SAVING RULE]:
Background agents survive /clear. They keep running and write result files when done.
When handing over with agents still running:
1. Handover file: ## RUNNING AGENTS section FIRST (before completed work)
2. List each agent: task name, expected output file path, NOT-YET-WRITTEN status
3. Copy-paste block: MUST lead with "AGENTS STILL RUNNING" + file paths
4. New session FIRST action: ONE `ls` check on expected paths. If files missing → WAIT. Period.
5. NEVER assume missing files = lost work. NEVER investigate. NEVER re-launch. NEVER poll.
6. Check again ONLY after agent completion notification OR after 15+ minutes with `wc -l`.
ANTI-PATTERN: "File doesn't exist, the agent must have failed, let me investigate" — THIS IS WRONG. The agent is still running. This mistake costs 20%+ tokens every time.
WHY: This pattern wastes a significant portion of every post-handover session. The new session sees missing files, panics, investigates, polls, and burns tokens before realizing agents are just still running.

WHY: handovers missing paths force the user to re-explain, wasting time and tokens.
</token_and_handover>

<memory_architecture>
For the full tiered memory system, see references/tiered-memory.md.
For the reflect-agent prompt template, see references/reflect-agent.md.

THREE-TIER MEMORY:

- **Tier 1 - Recent**: Plan files (`[PROJECT]/[TASK]-PLAN.md`). Session-scoped working memory. Distillation format.
- **Tier 2 - Distilled**: Auto-memory topic files (`memory/[topic].md`). Cross-session knowledge. Under 100 lines each.
- **Tier 3 - Permanent**: `MEMORY.md` index + `CLAUDE.md`. Stable patterns confirmed across 3+ sessions.

REFLECT-AGENT: When you need a fact from a previous session, launch a reflect-agent to search memory + plan files. It returns a focused answer without loading full files into orchestrator context. See references/reflect-agent.md for the prompt template.

PROMOTION: Tier 1 -> 2 after 2+ sessions. Tier 2 -> 3 after 3+ sessions or explicit user request.
COMPRESSION: Each tier has size limits. Exceeding triggers distillation, not expansion.
</memory_architecture>

<model_selection>
- **Opus** (recommended for ALL tasks): Every subagent, every task, every domain. Opus provides the strongest reasoning and catches issues that lower-tier models miss.
- **Sonnet**: Not recommended. Rework cost from missed issues typically exceeds per-token savings.
- **Haiku**: Not recommended for production tasks.
</model_selection>

<anti_patterns>
- Reading source files "to understand before delegating" (agents understand)
- Having agents return full content instead of writing to disk
- Running sequential agents when they could be parallel
- Skipping QC agent after implementation
- Pushing through at >55% tokens instead of writing handover
- Reasoning about complex questions in main context instead of delegating
- Reading full agent output files instead of just ## Summary sections
- Launching agent pipelines without writing the plan file first
- Saying "I'll quickly check this file" (there is no "quickly" - delegate or don't)
- Treating the 100-line limit as a guideline (it is a hard limit)
- Writing free-form notes in plan files instead of distillation format (use Context + Facts — see references/distillation.md)
- Letting plan files grow past 200 lines without recursive distillation
- Loading full memory/plan files into context when a reflect-agent could search them. PROCEDURE: Before reading ANY file, check `wc -l <path>`. If >100 lines, delegate to agent or reflect-agent. Never read first and check length after.
- Skipping memory tier promotion after completing multi-session tasks
</anti_patterns>

<reference_index>
Reference files are at: references/
- references/qc-loop.md
- references/anti-polling.md
- references/cascade-pipelines.md
- references/distillation.md
- references/tiered-memory.md
- references/reflect-agent.md
When passing reference file paths to agents, use the full absolute paths above.
</reference_index>
