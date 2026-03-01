# Claude Code Orchestration Protocol

A slash command for Claude Code that turns your session into a **delegation-only orchestrator** with iterative quality control loops and tiered memory.

## The Problem

When you give Claude Code a complex task, it reads files, reasons, writes code, and runs out of context window before finishing. At 50-65% token usage, output quality degrades (context rot). You lose work, have to re-explain, and start over.

## The Solution

This protocol forces Claude Code to operate as a **pure orchestrator**. It never reads files or reasons about content itself. Instead, it:

1. **Plans** the work in a plan file using distillation format (full absolute paths for session recovery)
2. **Delegates** everything to focused agents (each with their own token pool)
3. **Quality controls** every agent output through an iterative loop:
   - Task Agent produces output
   - QC Agent **hunts** for problems (not just validates)
   - Fix Agent applies fixes using **additive fix** (preserve original + add improvements, never drops content)
   - Loop repeats until genuinely **zero issues** remain (all severities: CRITICAL, MAJOR, MINOR)
   - Manual content fidelity verification after every pipeline run
4. **Remembers** across sessions using a 3-tier memory architecture (recent, distilled, permanent)
5. **Hands over** cleanly when tokens run low, so a fresh session can pick up exactly where you left off

## Key Features

- **Delegation-only orchestrator**: Main session never reads source files (hard limit: 100 lines). Pre-read size check (`wc -l`) before any file read. Agents do all the heavy lifting.
- **Semantic XML structure**: All protocol sections use XML tags for precise, unambiguous parsing.
- **Iterate-to-zero QC**: Every substantive output gets Task -> QC -> Fix -> Verify. No arbitrary iteration cap. All severities must resolve. Safety valves only for genuinely stuck situations (same-issue escape after 3 attempts, token-budget escape at 50%).
- **Manual content fidelity verification**: Automated tests are necessary but not sufficient. Manual count, cross-reference, and spot-check after every pipeline run.
- **Preserve-and-improve**: Fix agents must preserve everything from the original plus add improvements. Regression check is mandatory.
- **Structured agent output**: Every agent writes a `## Summary` header that the orchestrator reads. Full output stays on disk for downstream agents.
- **5-level token budget**: GREEN (0-40%), GREEN+ (40-50%, distillation trigger), YELLOW (50-55%, finish wave), ORANGE (55-65%, stop and handover), RED (65%+, emergency handover).
- **Self-contained agent prompts**: Agents don't see conversation history, so every prompt includes full context, paths, and success criteria.
- **3-tier memory**: Recent (plan files), Distilled (topic files), Permanent (MEMORY.md + CLAUDE.md).
- **Reflect-agents**: Search agents that retrieve facts from previous sessions without loading full files into orchestrator context.
- **Distillation format**: Plan files use Context + Facts format (not free-form notes). Recursive distillation when plan files exceed 200 lines.
- **Large file strategy**: Files >1,000 lines must use Bash + Python (not the Write tool, which has a 32K token limit).
- **Model selection**: Opus recommended for all tasks. Strongest reasoning prevents rework from missed issues.
- **Anti-polling**: Zero-poll waiting for agents. Never call TaskOutput while running (each poll dumps 50-150K tokens).
- **3 cascade patterns**: Orchestrator-driven (default), self-chaining, and parallel independent.
- **Agent Teams excluded**: This protocol uses the Task tool (subagents) only. Agent Teams is a separate system.

## Files

```
SKILL.md                        # Main protocol (~356 lines, semantic XML)
orchestrate.md                  # Thin slash command (copy to ~/.claude/commands/)
references/
  anti-polling.md               # Zero-poll waiting protocol
  cascade-pipelines.md          # 3 cascade patterns for agent chains
  qc-loop.md                    # Full QC procedure, severity ratings, fix protocol
  distillation.md               # Memory compression format (Context + Facts)
  reflect-agent.md              # Search agent for past-session recall
  tiered-memory.md              # 3-tier memory architecture
```

## Installation

Copy the skill and command to your Claude Code directories:

```bash
# Create skill directory
mkdir -p ~/.claude/skills/orchestrate/references

# Copy skill files
cp SKILL.md ~/.claude/skills/orchestrate/SKILL.md
cp references/*.md ~/.claude/skills/orchestrate/references/

# Copy the slash command
cp orchestrate.md ~/.claude/commands/orchestrate.md
```

After installation, update the reference paths in `SKILL.md` to use your full absolute paths (e.g., replace `references/` with `~/.claude/skills/orchestrate/references/`).

## Usage

Instead of giving Claude Code a task directly, prefix it with `/orchestrate`:

```
/orchestrate fix the authentication bug across the login flow
```

```
/orchestrate analyze all API endpoints in the services folder and create a comparison table
```

```
/orchestrate refactor the payment module to use the new API, update tests, and document changes
```

Claude Code will then plan the work, delegate to agents, QC everything with iterate-to-zero rigor, and hand over cleanly if it runs low on tokens.

## Research

This protocol was informed by patterns from:
- Official Anthropic subagent and agent teams documentation
- Community multi-agent orchestration repos and patterns for Claude Code
- Community discussions around context window degradation, quality regression, and session recovery failures
- Open-source agent memory architectures that implement tiered memory, compression formats, and search-based recall. These ideas were adapted to work natively within Claude Code's subagent architecture.

## License

MIT
