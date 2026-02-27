# Notes: Fix README Auto-Update Token Limit

## Problem
The `cog` script in root `README.md` passes entire project `README.md` files to `llm -m github/gpt-4.1` via stdin. Long files (like `2026-02-27-api-version-alignment/README.md`) exceed the 8000-token model limit.

## Current Code (the problematic subprocess call)
```python
result = subprocess.run(
    ['llm', '-m', MODEL, '-s', prompt],
    stdin=open(readme_path),
    capture_output=True,
    text=True,
    timeout=60
)
```

## Fix Strategy
1. Add `MAX_SUMMARY_INPUT_CHARS = 24000` (conservative for ~8k tokens, ~3 chars/token)
2. Add logic to extract "Executive Summary" section (regex, case-insensitive, handles numbered variants)
3. Fall back to first N chars if no Executive Summary or extracted content is too short
4. Use `input=content` instead of `stdin=open(readme_path)`

## Executive Summary extraction rules
- Match headings: `## Executive Summary`, `## 1. Executive Summary`, `## EXECUTIVE SUMMARY`, etc.
- Regex: `r'(?i)^##\s+(?:\d+\.\s+)?executive\s+summary\s*$'`
- Capture from that heading to next `##` heading or EOF
- If extracted content < ~100 chars, fall back to first N chars

## Implementation
- Changes limited to root `README.md` cog script only
- No workflow file changes needed
