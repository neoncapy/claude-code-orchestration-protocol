# Anti-Polling Rule — Detailed Reference

<overview>
This file contains the full anti-polling protocol referenced by the orchestrate skill SKILL.md.
</overview>

<zero_poll_waiting>
**ZERO-POLL WAITING [MANDATORY — HARDEST RULE IN THIS PROTOCOL]**

NEVER poll agent progress. NEVER call TaskOutput while an agent is running. NEVER use Bash to tail/read agent output files. NEVER use sleep+check loops.
</zero_poll_waiting>

<why_this_rule_exists>
Every poll of TaskOutput returns the FULL agent transcript (often 50-150K tokens) into the orchestrator's context. A single poll can consume 10-20% of the context window. Multiple polls can easily consume the entire remaining context window.
</why_this_rule_exists>

<what_to_do_instead>
1. Launch agent with run_in_background=true
2. Tell the user: "Agent launched. Will process result when done."
3. DO NOTHING. Zero tool calls. Zero bash commands. The system sends automatic notifications when agents complete.
4. When the completion notification arrives, read ONLY the agent's summary file (the one specified in the agent prompt), NOT the TaskOutput.
5. Process the summary and launch the next agent.
</what_to_do_instead>

<if_user_asks_for_status>
Say "Agent is still running. The system will notify me when it completes. No action needed from either of us."
</if_user_asks_for_status>

<if_agent_seems_stuck>
If agent seems stuck (>15 minutes):

Use ONE minimal bash command: `wc -l <output_file_path> 2>/dev/null || echo "NOT YET"`. This costs ~20 tokens vs 100K+ for TaskOutput. Never use TaskOutput for status checks.
</if_agent_seems_stuck>

<hard_ban_list>
- TaskOutput with block=false (returns full transcript)
- TaskOutput with block=true (blocks AND returns full transcript)
- Bash: tail/cat/head on agent output files
- Bash: sleep + any check pattern
- Reading agent temp files directly
</hard_ban_list>
