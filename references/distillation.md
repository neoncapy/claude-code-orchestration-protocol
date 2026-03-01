<overview>
Distillation is a memory compression technique adapted from open-source agent memory architectures. Unlike summarization (which loses operational details), distillation preserves the intelligence agents need to continue working. Applied to plan files, handover prompts, and memory files.
</overview>

<core_concept>
Distillation produces TWO outputs per work segment:

1. **Operational Context** (1-3 sentences): What happened, what state we're in
2. **Retained Facts** (bullet list): Actionable details that must survive compression

This dual-output format preserves working intelligence while dropping narrative noise.
</core_concept>

<what_to_retain>
RETAIN (operational intelligence that agents need):

- Full absolute file paths and what each file contains
- Decisions made and their rationale (WHY, not just WHAT)
- User corrections and preferences expressed during work
- Specific values: URLs, config settings, version numbers, commands
- Errors encountered and how they were resolved
- Dependencies between tasks (what blocks what)
- Requirements met (with citations) and requirements still pending
- Agent verdicts (PASS/FAIL) and their key findings
</what_to_retain>

<what_to_excise>
EXCISE (noise that wastes context):

- Debugging back-and-forth that led to dead ends (keep only final approach)
- Verbose tool outputs (keep only relevant findings)
- Narrative filler ("Let me check...", "I'll now look at...")
- Casual acknowledgments and pleasantries
- Missteps and corrections (keep only the correct final approach)
- Repeated information already captured in retained facts
- Full file contents (reference by path, don't embed)
</what_to_excise>

<plan_file_format>
Apply distillation to plan file updates. Each completed phase:

```
## Phase N: [Name] [checkmark]

**Context:** [1-3 sentences: what this phase accomplished and current state]

**Facts:**
- Output: /absolute/path/to/output.md (N lines, [brief description])
- Decision: [what was decided] because [rationale]
- Agent: [agent type] verdict [PASS/FAIL], [1-line finding]
- Blocker resolved: [what was blocking] -> [how resolved]
- Value: [specific value discovered/confirmed]
```

Uncompleted phases retain their full task breakdown (no distillation yet — you can't distill what hasn't happened).
</plan_file_format>

<handover_distillation>
Handover prompts use the same distillation format:

```
## Handover: [Task Name]

**Context:** [2-3 sentences: overall state, what's done, what's next]

**Facts:**
- Plan file: /absolute/path/PLAN.md
- Project CLAUDE.md: /absolute/path/CLAUDE.md
- Files created: [list with full paths]
- Files modified: [list with full paths]
- Current phase: [N] of [total]
- Next action: [specific next step]
- Running agents: [list — check output before re-launching]

**Resume instruction:** Read plan file first, then project CLAUDE.md, then continue from Phase [N].
```
</handover_distillation>

<recursive_distillation>
For long-running tasks spanning multiple sessions:

Session plan files accumulate distilled phases. When a plan file exceeds 200 lines, apply recursive distillation: compress the oldest completed phases into a single higher-level meta-distillation.

```
Session 1: Phases 1-3 completed (detailed distillations)
Session 2: Phases 1-3 -> meta-distillation (5 lines), Phases 4-6 (detailed)
Session 3: Phases 1-6 -> meta-distillation (5 lines), Phases 7-9 (detailed)
```

This creates a fractal pattern: recent work stays detailed, older work becomes progressively compressed. Same principle as recursive compression in tiered memory systems, applied manually via plan files.

At 40-50% token usage (per SKILL.md token budget GREEN+ threshold): proactively check if the plan file needs compression. If completed phases are verbose, launch a distillation agent to compress them before continuing work.
</recursive_distillation>

<known_limitations>
Artifact tracking (which files were read/modified during a session) remains the hardest unsolved problem in agent memory systems, with low scores across all evaluated systems in recent benchmarks. The RETAIN list above includes file paths as a partial mitigation, but systematic artifact tracking is not yet implemented. Over recursive distillation cycles, artifact tracking fidelity will degrade. This is a known gap shared by all current systems.
</known_limitations>
