# Fix README Auto-Update Token Limit

## Problem

The `cog` script embedded in the root `README.md` was failing with a `tokens_limit_reached` error when generating per-project summaries for long investigation README files. The script was passing the entire content of each project's `README.md` to `llm -m github/gpt-4.1` via `stdin`, but long reports (e.g. `2026-02-27-api-version-alignment/README.md`) exceeded the model's 8,000-token request limit.

## Solution

Two targeted changes were made to the `cog` Python script in the root `README.md`:

### 1. Added `MAX_SUMMARY_INPUT_CHARS` constant

```python
MAX_SUMMARY_INPUT_CHARS = 24000
```

Conservative cap at ~24,000 characters (~8k tokens at ~3 chars/token).

### 2. Added `extract_summary_input()` helper function

A smart cap strategy:
- If the README contains an `## Executive Summary` heading (case-insensitive, handles numbered variants like `## 1. Executive Summary`), extract that section (up to the next `##` heading or EOF).
- If the extracted content is >= 100 characters, use it as the model input.
- Otherwise, fall back to the first `MAX_SUMMARY_INPUT_CHARS` characters.

### 3. Changed subprocess call

Changed from `stdin=open(readme_path)` to `input=summary_input` so the truncated string is passed directly.

## Files Changed

- `README.md` — cog script updated with the constant, helper function, and subprocess change.

## Testing

The `extract_summary_input` logic was verified with unit tests covering:
- Standard `## Executive Summary` heading
- Numbered variant `## 1. Executive Summary`
- Uppercase variant `## EXECUTIVE SUMMARY`
- Fallback when no executive summary present
- Fallback when executive summary content is too short
