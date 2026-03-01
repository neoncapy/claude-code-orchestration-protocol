<overview>
The reflect-agent is a specialized subagent that searches memory and plan files to retrieve specific information WITHOUT loading full files into the orchestrator's context. Adapted from reflect-tool patterns in open-source agent memory systems.
</overview>

<when_to_use>
Launch a reflect-agent when:

- You need a specific fact from a previous session (file path, decision, value)
- You need to check if something was already done or decided
- You need context from a plan file that's >100 lines
- The user references something from a previous session you don't have in context
- You need to search across multiple memory/plan files for related information

Limitation: The reflect-agent can only find information that was written to disk (plan files, memory files, handover files). Unlike reflect tools in dedicated memory systems that search full temporal message history, conversations not captured in persistent files are not searchable.

Do NOT use when:
- The information is already in your current context (MEMORY.md, loaded CLAUDE.md)
- The plan file is <100 lines (just read it directly — under the 100-line limit)
- You need the full file content for an agent (pass the path to that agent instead)
</when_to_use>

<prompt_template>
```
You are a REFLECT agent. Your job is to search memory and plan files to answer a specific question, then return a focused answer.

QUESTION: [specific question to answer]

SEARCH LOCATIONS (in order of priority):
1. Auto-memory: memory/
2. Plan files: [PROJECT]/[TASK]-PLAN.md
3. Project CLAUDE.md: [PROJECT]/CLAUDE.md
4. Handover files: [PROJECT]/*HANDOVER*.md
5. Session context files: [PROJECT]/.claude/context/

SEARCH STRATEGY:
- Use Grep to search for keywords across all locations
- Read only the relevant sections of matching files
- Do NOT read entire files unless they're <50 lines
- If the answer requires checking multiple files, search all of them

OUTPUT FORMAT:
## Summary
Answer: [direct answer to the question]
Sources: [full absolute paths where you found the information]
Confidence: [HIGH/MEDIUM/LOW]
---
[Supporting details and exact quotes if needed]

If the requested information is not found in any search location, respond with:
## Summary
Answer: NOT FOUND: [query]. Searched: [list of files checked]. The information does not exist in persistent memory.
Sources: [files checked]
Confidence: N/A
---
Do NOT guess or hallucinate an answer. NOT FOUND is a valid and expected response.
```
</prompt_template>

<model_and_cost>
**Model: Opus** (recommended). Reflect-agents perform search + extraction with light reasoning:
- Grep for keywords
- Read matching sections
- Extract and return facts
- Assess relevance and confidence

Opus provides the most reliable results for search tasks that require judgment about relevance and confidence.
</model_and_cost>

<examples>
**Example 1: Retrieving a project decision**

Orchestrator prompt: "What database engine was chosen for the backend?"

Reflect-agent prompt:
```
QUESTION: What database engine was chosen for the backend, and what was the rationale?
SEARCH LOCATIONS: memory/, my-project/*PLAN*.md, my-project/CLAUDE.md
```

Result:
```
## Summary
Answer: PostgreSQL, for JSONB support and full-text search. Evaluated against MySQL and MongoDB.
Sources: memory/MEMORY.md (line 142), my-project/BACKEND-PLAN.md (Phase 2)
Confidence: HIGH
```

**Example 2: Checking if a task was completed**

Orchestrator prompt: "Did we already migrate the user table to the new schema?"

Reflect-agent prompt:
```
QUESTION: Was the user table migrated to the new schema? If so, where is the migration file?
SEARCH LOCATIONS: memory/, my-project/*PLAN*.md, my-project/*HANDOVER*.md
```

**Example 3: Finding a preference**

Orchestrator prompt: "What format does the team prefer for API responses?"

Reflect-agent prompt:
```
QUESTION: What API response format does the team prefer?
SEARCH LOCATIONS: memory/MEMORY.md, my-project/CLAUDE.md
```
</examples>
