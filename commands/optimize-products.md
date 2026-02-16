---
description: Optimize product listings from a CSV file
allowed-tools: Read, Write, Edit, Bash, Task, Glob, Grep, TodoWrite, AskUserQuestion
argument-hint: [csv-file-path or just drop a CSV]
---

Read the ecommerce-product-optimizer skill (SKILL.md) from this plugin and follow its instructions exactly.

The user wants to optimize ecommerce product listings from a CSV file. If they provided a file path as an argument, use that: $ARGUMENTS

If no file path was provided, check if the user uploaded a CSV in this conversation or ask them to provide one.

Follow the full workflow in SKILL.md: parse the CSV, map columns, detect processing mode from the user's instructions, spawn parallel subagents to optimize each product row, merge results back into the original CSV with new columns appended, and save the enriched CSV to the output folder.

Key behaviors:
- Default to parallel batched processing unless the user says otherwise
- If the user says "just one" or "test one row", process only 1 row
- If the user says "sequential" or "one at a time", process one row at a time with approval between each
- Don't pause for confirmations unless the user asks or something is genuinely ambiguous
- Always save the output CSV and provide a clickable link
