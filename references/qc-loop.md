# Quality Control Loop — Detailed Reference

<overview>
This file contains the full QC procedure referenced by the orchestrate skill SKILL.md.
</overview>

<table_of_contents>
- `<loop_structure>` — Flow diagram and safety valves
- `<qc_agent_rules>` — What QC agents must do
- `<qc_procedure>` — Mandatory 4-step procedure
- `<preserve_and_improve>` — Fix agent: preserve original + add improvements
- `<agent_failure_protocol>` — Handling failed agents
</table_of_contents>

<loop_structure>
```
TASK AGENT -> output to disk
    |
QC AGENT -> hunts for problems
    | issues found?
FIX AGENT -> all fixes via ADDITIVE FIX (preserve original + add improvements)
    |
VERIFY AGENT -> confirms fixes without regression
    | still issues? -> loop to FIX (until genuinely ZERO issues, all severities)
```

NO ARBITRARY ITERATION CAP. Loop runs until ZERO issues remain. This is the most important rule in the orchestration protocol.

ALL SEVERITIES MUST BE RESOLVED — CRITICAL, MAJOR, AND MINOR. MINOR issues are NOT optional, NOT deprioritized, NOT "good enough to ship." The loop does not stop until the count is literally zero across all severities.

WHY THIS IS NON-NEGOTIABLE: The user may be building high-stakes work across multiple domains. A MINOR issue left unfixed becomes a quality gap that compounds. "Good enough" is never acceptable.

SAFETY VALVES (the ONLY acceptable reasons to stop before zero):
1. Same-issue escape: if the SAME issue persists after 3 fix attempts on that specific issue, it is structural. Escalate to the user. Do not keep retrying.
2. Token-budget escape: if at YELLOW threshold (50% tokens), report remaining issues to the user instead of continuing. Token protocol takes priority over QC completion.

New issues found on iteration 3, 4, 5 are real and must be fixed. The safety valves catch genuinely stuck situations without imposing arbitrary limits.
</loop_structure>

<qc_agent_rules>
- FIND PROBLEMS, not confirm work looks good. "Looks good" catches nothing.
- Hunt: what was missed? Partially implemented? Lost? Contradicts requirements?
- Compare output against requirements AND source material.
- WHY: one-pass work always has undetected issues.
</qc_agent_rules>

<qc_procedure>
QC Procedure (mandatory, in order):

1. List every requirement. For each, cite WHERE in output it is met. Uncitable = MISSING.
2. Check for content in original NOT in output (loss detection).
3. Check factual consistency. Flag unverifiable claims.
4. Rate: CRITICAL (requirement unmet), MAJOR (quality gap), MINOR (polish).
</qc_procedure>

<preserve_and_improve>
**[CRITICAL]**

Fix agent output MUST contain everything from the original PLUS improvements. Missing anything = FAILED.
- WHY: Claude's worst failure mode is dropping details while "improving."
- FIX AGENTS: before writing, list every section heading and requirement reference from original. After writing, verify each present. Append to ## Summary: "Regression check: [N/N preserved]". Any 0 = add back before done. WHY: gives verify agent an auditable checklist.
</preserve_and_improve>

<fix_agent_input>
FIX AGENT INPUT [MANDATORY]:
Every fix agent prompt MUST include:
1. Full absolute path to the output file to fix
2. Full absolute path to the QC report
3. Explicit instruction: "Read the QC report FIRST, then fix ONLY the issues listed"
4. Original requirements that generated the output
5. If this is attempt 2+: the ## Stuck Issues section from the plan file
</fix_agent_input>

<manual_content_fidelity_verification>
MANDATORY MANUAL VERIFICATION [NON-NEGOTIABLE]

Automated tests (exit code 0, file exists, no tracebacks) are NECESSARY but NOT SUFFICIENT. They only confirm what you expected. They NEVER catch what you didn't expect.

AFTER every pipeline run, E2E test, or substantive output:
1. MANUALLY count items (images, tables, sections) in the actual output directory
2. CROSS-REFERENCE against the manifest/index/metadata
3. CROSS-REFERENCE against the source file (does the source have N items? Does the output?)
4. CHECK classifications for correctness — not just that they exist
5. LOOK for orphans (files not in index), ghosts (index entries not on disk), and mismatches

WHY: Running a script and checking "expected result" is a self-validating prophecy. The script says it found 5 items, you expected ~5, you say PASS. But you never checked if the source actually had 8 items and 3 were missed. Manual verification catches the gap automated tests cannot see.

This applies to ALL domains: code generation, document generation, data extraction, any output where counts or classifications matter.

PROCEDURE FOR OUTPUT VERIFICATION:
- `ls` the output directory → count actual files
- Read the manifest → count entries → compare to `ls`
- Read the index → count by category → compare to manifest
- Spot-check: open 2-3 items, verify classification is correct
- Check source: does the source have items the pipeline missed?

NEVER report a test as PASS without this manual step. "Script exited 0" is not verification. "I counted N files, manifest shows N, index shows N, and I spot-checked 3 classifications" IS verification.
</manual_content_fidelity_verification>

<agent_failure_protocol>
If agent produces no or unusable output, do NOT re-read or debug. Log in plan file (agent, task, failure). Re-launch with refined prompt. Max 1 retry, then report to the user. WHY: debugging wastes orchestrator tokens on content it should never read.
</agent_failure_protocol>
