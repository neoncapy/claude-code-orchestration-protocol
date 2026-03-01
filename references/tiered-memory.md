<overview>
Three-tier memory architecture for Claude Code, adapted from open-source agent memory architectures. Organizes information by recency and durability to maximize context window efficiency. Works natively with Claude Code's auto-memory and plan file system.
</overview>

<tier_1_recent>
**Tier 1: Recent (Session-Scoped Working Memory)**

Location: Project plan files (`[PROJECT]/[TASK]-PLAN.md`)
Lifecycle: Created per task, updated per action, archived on completion
Content: Active work state, current findings, in-progress analysis
Format: Distillation format (see references/distillation.md)

This is WORKING MEMORY. Full detail on recent phases, distilled older phases. Frequently updated. Lives only as long as the task is active.

Comparison: Some memory systems use a verbatim temporal store of all messages. This protocol uses plan files in distillation format instead, because Claude Code sessions cannot persist full message histories. Plan files serve the same purpose (session-scoped working memory) through a different mechanism.

After task completion: promote key learnings to Tier 2, archive the plan file.
</tier_1_recent>

<tier_2_distilled>
**Tier 2: Distilled (Cross-Session Knowledge)**

Location: Auto-memory topic files (`memory/[topic].md`)
Lifecycle: Updated when patterns emerge across 2+ sessions
Content: Project-specific knowledge, recurring solutions, workflow patterns
Format: Concise bullets organized by topic

Examples of topic files:
- `memory/project-architecture.md` — system design decisions and patterns
- `memory/project-api.md` — API design patterns and versioning choices
- `memory/debugging.md` — recurring debugging patterns and solutions
- `memory/deployment-patterns.md` — CI/CD and infrastructure learnings

Topic files stay under 100 lines. When approaching the limit, distill the oldest entries into more compact form (apply recursive distillation).

Distillation trigger: at GREEN+ (40-50% tokens) per SKILL.md token budget table. More conservative than the 60% threshold used in some memory systems because Claude Code context rot is a known problem at high token usage.

Promotion criteria (Tier 1 -> Tier 2):
- Finding useful across 2+ sessions on the same project
- Solution to a problem that's likely to recur
- Project-specific knowledge not captured in CLAUDE.md
</tier_2_distilled>

<tier_3_permanent>
**Tier 3: Permanent (Global Knowledge)**

Location: `MEMORY.md` (auto-memory index, 200-line limit) + project `CLAUDE.md` files
Lifecycle: Permanent until explicitly updated or removed
Content: Stable rules, confirmed patterns, architectural decisions, user preferences
Format: Compact entries in MEMORY.md, detailed rules in CLAUDE.md or rules/

Promotion criteria (Tier 2 -> Tier 3):
- Pattern confirmed across 3+ sessions or 2+ projects
- User explicitly requests remembering
- Architectural decision unlikely to change
- Workflow rule that applies globally
</tier_3_permanent>

<promotion_mechanism>
**How Promotion Works:**

Promotion is triggered manually by the orchestrator during handover. When writing a handover prompt, the orchestrator checks:
- Has this fact appeared in 2+ plan files? If yes, promote to Tier 2 (create/update auto-memory topic file).
- Has a Tier 2 fact been confirmed across 3+ sessions? If yes, promote to Tier 3 (add to MEMORY.md index).
- Did the user explicitly say "remember this"? Promote directly to Tier 3.

There is no automatic tracking of session counts. The orchestrator infers frequency from plan file evidence at handover time.
</promotion_mechanism>

<maintenance>
**Memory Maintenance Protocol:**

1. After completing a multi-session task: review plan file for Tier 2/3 promotions (see promotion mechanism above)
2. When MEMORY.md approaches 200 lines: distill oldest entries or move details to topic files
3. When a topic file approaches 100 lines: distill oldest entries
4. When rules change: update ALL tiers (stale knowledge in lower tiers causes bugs)
5. At session start: if orchestrating a complex task, check relevant topic files exist

**Memory never grows unbounded.** Each tier has a size limit. Exceeding it triggers distillation, not expansion.
</maintenance>
