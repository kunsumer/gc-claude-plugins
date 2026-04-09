# GC Yeogibrain Guide Plugin

## Philosophy
yeogibrain is only as useful as the questions you ask it. The difference
between a helpful response and a dead end is usually the search strategy,
not the data. Always follow the document chain — a PRD without its AC and
AB results is an incomplete picture.

## Top 3 mistakes to avoid
1. Presenting researcher summaries as "what users said" (use 인터뷰보이스 for
   actual quotes)
2. Giving up after one empty search instead of trying different terms,
   categories, or full_text_search
3. Exposing document IDs instead of titles in responses

## When NOT to use this skill
- Questions about code, architecture, or technical implementation
- Questions answerable from the current repo's docs or code
- General knowledge questions not related to GC Company products

## Prerequisite
The yeogibrain MCP server must be configured in your project's MCP settings.
This plugin provides usage guidance only — it does not install the MCP server.

## Cross-plugin handoffs
- If research results reveal compliance-sensitive features (PII collection,
  cross-border data, consent flows), note that the `korea-compliance` plugin
  can review them
