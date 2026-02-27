# Research projects carried out by AI tools

Each directory in this repo is a separate research project carried out by an LLM tool - starting with Github Copilot Coding Agent. Every single line of text and code was written by an LLM.

This is a copy of the Code Research project by Simon Williams. See [Code research projects with async coding agents like Claude Code and Codex](https://simonwillison.net/2025/Nov/6/async-code-research/) for more details on how this works.

*Times shown are in UTC.*

<!--[[[cog
import os
import re
import subprocess
import pathlib
from datetime import datetime, timezone

# Model to use for generating summaries
MODEL = "github/gpt-4.1"

# Max characters to send to the model (~8k token limit at ~3 chars/token, conservative)
MAX_SUMMARY_INPUT_CHARS = 24000
# Minimum length for an extracted Executive Summary to be considered useful
MIN_EXECUTIVE_SUMMARY_CHARS = 100


def extract_summary_input(readme_text):
    """Return the Executive Summary section if present, else first MAX_SUMMARY_INPUT_CHARS chars."""
    lines = readme_text.splitlines(keepends=True)
    # Find an "## Executive Summary" heading (case-insensitive, optional numeric prefix)
    exec_summary_re = re.compile(r'^##\s+(?:\d+\.\s+)?executive\s+summary\s*$', re.IGNORECASE)
    next_h2_re = re.compile(r'^##\s+', re.IGNORECASE)
    start_idx = None
    for i, line in enumerate(lines):
        if exec_summary_re.match(line.rstrip()):
            start_idx = i
            break
    if start_idx is not None:
        # Capture from heading until next ## heading or EOF
        end_idx = len(lines)
        for i in range(start_idx + 1, len(lines)):
            if next_h2_re.match(lines[i]):
                end_idx = i
                break
        section = ''.join(lines[start_idx:end_idx]).strip()
        if len(section) >= MIN_EXECUTIVE_SUMMARY_CHARS:
            return section
    # Fallback: first N characters
    return readme_text[:MAX_SUMMARY_INPUT_CHARS]

# Get all subdirectories with their first commit dates
research_dir = pathlib.Path.cwd()
subdirs_with_dates = []

for d in research_dir.iterdir():
    if d.is_dir() and not d.name.startswith('.'):
        # Get the date of the first commit that touched this directory
        try:
            result = subprocess.run(
                ['git', 'log', '--diff-filter=A', '--follow', '--format=%aI', '--reverse', '--', d.name],
                capture_output=True,
                text=True,
                timeout=5
            )
            if result.returncode == 0 and result.stdout.strip():
                # Parse first line (oldest commit)
                date_str = result.stdout.strip().split('\n')[0]
                commit_date = datetime.fromisoformat(date_str.replace('Z', '+00:00'))
                subdirs_with_dates.append((d.name, commit_date))
            else:
                # No git history, use directory modification time
                subdirs_with_dates.append((d.name, datetime.fromtimestamp(d.stat().st_mtime, tz=timezone.utc)))
        except Exception:
            # Fallback to directory modification time
            subdirs_with_dates.append((d.name, datetime.fromtimestamp(d.stat().st_mtime, tz=timezone.utc)))

# Print the heading with count
print(f"## {len(subdirs_with_dates)} research projects\n")

# Sort by date, most recent first
subdirs_with_dates.sort(key=lambda x: x[1], reverse=True)

for dirname, commit_date in subdirs_with_dates:
    folder_path = research_dir / dirname
    readme_path = folder_path / "README.md"
    summary_path = folder_path / "_summary.md"

    date_formatted = commit_date.astimezone(timezone.utc).strftime('%Y-%m-%d %H:%M')

    # Get GitHub repo URL
    github_url = None
    try:
        result = subprocess.run(
            ['git', 'remote', 'get-url', 'origin'],
            capture_output=True,
            text=True,
            timeout=2
        )
        if result.returncode == 0 and result.stdout.strip():
            origin = result.stdout.strip()
            # Convert SSH URL to HTTPS URL for GitHub
            if origin.startswith('git@github.com:'):
                origin = origin.replace('git@github.com:', 'https://github.com/')
            if origin.endswith('.git'):
                origin = origin[:-4]
            github_url = f"{origin}/tree/main/{dirname}"
    except Exception:
        pass

    # Extract title from first H1 header in README, fallback to dirname
    title = dirname
    if readme_path.exists():
        with open(readme_path, 'r') as f:
            for readme_line in f:
                if readme_line.startswith('# '):
                    title = readme_line[2:].strip()
                    break

    if github_url:
        print(f"### [{title}]({github_url}#readme) ({date_formatted})\n")
    else:
        print(f"### {title} ({date_formatted})\n")

    # Check if summary already exists
    if summary_path.exists():
        # Use cached summary
        with open(summary_path, 'r') as f:
            description = f.read().strip()
            if description:
                print(description)
            else:
                print("*No description available.*")
    elif readme_path.exists():
        # Generate new summary using llm command
        prompt = """Summarize this research project concisely. Write just 1 paragraph (3-5 sentences) followed by an optional short bullet list if there are key findings. Vary your opening - don't start with "This report" or "This research". Include 1-2 links to key tools/projects. Be specific but brief. No emoji."""
        readme_text = readme_path.read_text()
        summary_input = extract_summary_input(readme_text)
        result = subprocess.run(
            ['llm', '-m', MODEL, '-s', prompt],
            input=summary_input,
            capture_output=True,
            text=True,
            timeout=60
        )
        if result.returncode != 0:
            error_msg = f"LLM command failed for {dirname} with return code {result.returncode}"
            if result.stderr:
                error_msg += f"\nStderr: {result.stderr}"
            raise RuntimeError(error_msg)
        if result.stdout.strip():
            description = result.stdout.strip()
            print(description)
            # Save to cache file
            with open(summary_path, 'w') as f:
                f.write(description + '\n')
        else:
            raise RuntimeError(f"LLM command returned no output for {dirname}")
    else:
        print("*No description available.*")

    print()  # Add blank line between entries

# Add AI-generated note to all project README.md files
# Note: we construct these marker strings via concatenation to avoid the HTML comment close sequence
AI_NOTE_START = "<!-- AI-GENERATED-NOTE --" + ">"
AI_NOTE_END = "<!-- /AI-GENERATED-NOTE --" + ">"
AI_NOTE_CONTENT = """> [!NOTE]
> This is an AI-generated research report. All text and code in this report was created by an LLM (Large Language Model). For more information on how these reports are created, see the [main research repository](https://github.com/simonw/research)."""

for dirname, _ in subdirs_with_dates:
    folder_path = research_dir / dirname
    readme_path = folder_path / "README.md"

    if not readme_path.exists():
        continue

    content = readme_path.read_text()

    # Check if note already exists
    if AI_NOTE_START in content:
        # Replace existing note
        pattern = re.escape(AI_NOTE_START) + r'.*?' + re.escape(AI_NOTE_END)
        new_note = f"{AI_NOTE_START}\n{AI_NOTE_CONTENT}\n{AI_NOTE_END}"
        new_content = re.sub(pattern, new_note, content, flags=re.DOTALL)
        if new_content != content:
            readme_path.write_text(new_content)
    else:
        # Add note after first heading (# ...)
        lines = content.split('\n')
        new_lines = []
        note_added = False
        for i, line in enumerate(lines):
            new_lines.append(line)
            if not note_added and line.startswith('# '):
                # Add blank line, then note, then blank line
                new_lines.append('')
                new_lines.append(AI_NOTE_START)
                new_lines.append(AI_NOTE_CONTENT)
                new_lines.append(AI_NOTE_END)
                note_added = True

        if note_added:
            readme_path.write_text('\n'.join(new_lines))

]]]-->
## 1 research projects

### [API Version-Alignment Strategy for Mobile App + Backend API](https://github.com/chrisabbotthauxwell/code-research/tree/main/2026-02-27-api-version-alignment#readme) (2026-02-27 16:00)

URL-path-based whole-API versioning, utilizing prefixes like `/v1/` or `/v2/`, ensures clear version identification and robust backwards compatibility for API clients, particularly mobile apps. The server maintains parallel support for all major versions, releasing new ones only for breaking changes, while additive updates apply to the current version. Deprecation and end-of-life are transparently signaled via HTTP headers, and strict end-of-life enforcement is implemented with `HTTP 410 Gone`. On Azure Container Apps, each major version operates as a separate container, with Cloudflare Tunnel (`cloudflared`) https://github.com/cloudflare/cloudflared handling ingress and routing, sidestepping Azure API Management while supporting traffic-splitting for staged rollouts. This approach avoids less reliable strategies like query parameter or per-endpoint-only versioning.

Key findings:
- Supports multi-versioned APIs efficiently without degrading client experience.
- Deprecation is standardized using HTTP headers (`Deprecation`, `Sunset`), improving communication to consumers.
- Container-based isolation enables safe parallel deployment and targeted rollouts via Azure Container Apps https://learn.microsoft.com/en-us/azure/container-apps/.
- Avoids pitfalls of ambiguous or fragmented versioning schemes.

<!--[[[end]]]-->

---

## Updating this README

This README uses [cogapp](https://nedbatchelder.com/code/cog/) to automatically generate project descriptions.

### Automatic updates

A GitHub Action automatically runs `cog -r -P README.md` on every push to main and commits any changes to the README or new `_summary.md` files.

### Manual updates

To update locally:

```bash
# Run cogapp to regenerate the project list
cog -r -P README.md
```

The script automatically:
- Discovers all subdirectories in this folder
- Gets the first commit date for each folder and sorts by most recent first
- For each folder, checks if a `_summary.md` file exists
- If the summary exists, it uses the cached version
- If not, it generates a new summary using `llm -m <!--[[[cog
print(MODEL, end='')
]]]-->
github/gpt-4.1
<!--[[[end]]]-->` with a prompt that creates engaging descriptions with bullets and links
- Creates markdown links to each project folder on GitHub
- New summaries are saved to `_summary.md` to avoid regenerating them on every run

To regenerate a specific project's description, delete its `_summary.md` file and run `cog -r -P README.md` again.
