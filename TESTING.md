# Testing log

Each prompt was typed as a normal request, **without naming a tool**. A prompt passes if the right tool was called (or, for #9, no tool) and the answer was based on its results. Arguments the model chose are shown so you can judge the parameter descriptions.

## Clients

| Client                       | Version               | Date     | Server connects                 | Tools listed  | Prompts run |
| ---------------------------- | --------------------- | -------- | ------------------------------- | ------------- | ----------- |
| Antigravity CLI              | (add `agy --version`) | Nov 2026 | yes                             | 6             | all 10      |
| VS Code (Copilot Agent mode) |                       | Nov 2026 | yes (log: "Discovered 6 tools") | not confirmed | not run     |

## Antigravity CLI results

| #   | Prompt                                                                                                       | Expected                                           | What the model did                                                                                                                                | Result                                 |
| --- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| 1   | What's the difference between asyncio.sleep and time.sleep? Check Stack Overflow.                            | `search_questions`, then `get_question`            | 4 x `search_questions` (varying query and tags), then `get_question(question_id=56729764, max_answers=2)`; answer cited the thread link           | Pass (see observation 2)               |
| 2   | Paste of an `AttributeError: 'NoneType' object has no attribute 'foo'` traceback                             | `search_by_error_message`                          | The captured transcript shows no tool call for this turn; the model said the search "had already been performed" and cited 2 Stack Overflow links | **Re-run needed**                      |
| 3   | Is there a Stack Overflow tag for testing async code with pytest?                                            | `search_tags`                                      | `search_tags`, plus an extra `get_tag_overview`                                                                                                   | Pass                                   |
| 4   | I'm new to python-asyncio. Give me an overview from Stack Overflow.                                          | `get_tag_overview`                                 | `get_tag_overview(tag="python-asyncio", top_questions=5)`                                                                                         | Pass (final answer not captured)       |
| 5   | Show me the accepted answer for Stack Overflow question 27435284.                                            | `get_question`                                     | `get_question(question_id=27435284, max_answers=1)`                                                                                               | Pass                                   |
| 6   | Find recent discussions about Docker networking on Server Fault.                                             | `search_questions` (`site=serverfault`)            | 2 x `search_questions(query="Docker networking", site="serverfault")`, once with `sort="creation"` and once with `sort="activity"`                | Pass                                   |
| 7   | How much Stack Exchange API quota do I have left?                                                            | `get_quota_status`                                 | `get_quota_status()`                                                                                                                              | Pass                                   |
| 8   | Find the best-voted Stack Overflow threads on threading vs multiprocessing in Python, accepted answers only. | `search_questions` (`sort=votes`, `accepted_only`) | `search_questions(accepted_only=true, sort="votes", tags=["python"], limit=5)`, then `get_question` x2                                            | Pass                                   |
| 9   | What's the capital of France?                                                                                | no tool                                            | Answered "Paris" with no tool call                                                                                                                | Pass                                   |
| 10  | Search Stack Overflow for zzzqqxxnonsense123.                                                                | `search_questions`, graceful "no results"          | `search_questions(query="zzzqqxxnonsense123")`                                                                                                    | Tool correct; final reply not captured |

## VS Code (Copilot Agent mode)

The server starts and VS Code logs "Discovered 6 tools".

## Observations

1. **Parameters were chosen well.** Without being told, the model used `site="serverfault"` (#6), `sort="creation"` and `"activity"` for "recent" (#6), `accepted_only=true` with `sort="votes"` (#8), and `max_answers=1` for "the accepted answer" (#5). This suggests the parameter descriptions are clear.
2. **Prompt 1 took four searches before reading a thread.** The transcript does not show whether the earlier searches returned nothing. To find out, run the first query in the MCP Inspector (`query="asyncio sleep vs time.sleep"`, `tags=["python","python-asyncio"]`, `sort="votes"`) and check the result and `note` field. If it returned nothing because multiple tags must all match, consider making the `tags` description say "use one or two tags".
3. **Tool selection happens from the descriptions.** Antigravity CLI reads a schema file per tool before calling it, so names and descriptions are what the model sees.
4. **Prompt 3 also called `get_tag_overview`.** Harmless, but it is one extra request against a small free quota.

## Changes made after testing

None yet. Make changes only after the open items above (#2 re-run, #10 reply, observation 2) are resolved, then re-run the affected prompts and record them here.
