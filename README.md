# agentlint

[![CI](https://github.com/thenayanhazra/Skill-Agent-File-Linter/actions/workflows/ci.yml/badge.svg)](https://github.com/thenayanhazra/Skill-Agent-File-Linter/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**Checks the instruction files AI agents load (skills, project rules, MCP configs) for hidden attacks before an agent ever reads them.**

Agent instruction files are this decade's macros: plain text that a machine treats as commands, copied between repositories and pulled in from public skill marketplaces. agentlint scans them for invisible Unicode, override attempts, data-exfiltration tricks and dangerous tool configurations. It runs in milliseconds, from the terminal or as a GitHub Action.

## Why it matters

- **People review these files, and attackers hide from people.** A right-to-left override can make `report.exe` display as `report.pdf`. A zero-width character can split a phrase so it slips past review. The file looks clean to a reviewer and reads differently to the model.
- **One poisoned skill compromises every agent that loads it.** Catching it at pull-request time is the cheapest control there is.
- **It fits the pipeline you already run.** SARIF output feeds GitHub code scanning, and exit codes fail the build on critical findings.

## What you get

Scanning [a sample skill](examples/poisoned-skill/SKILL.md) that presents itself as a harmless PDF summariser:

```
$ agentlint check examples/poisoned-skill/SKILL.md
examples/poisoned-skill/SKILL.md
  critical[AGT042]: Sensitive path or credential reference: '~/.aws/'
   --> examples/poisoned-skill/SKILL.md:14:26
  error[AGT002]: Zero-width character U+200B ZERO WIDTH SPACE at line 14, col 84
   --> examples/poisoned-skill/SKILL.md:14:84
  error[AGT040]: Image URL with data-carrying query string: https://telemetry.example.net/p?d=SUMMARY_AND_CONTEXT
   --> examples/poisoned-skill/SKILL.md:15:27
  critical[AGT020]: Instruction override pattern: 'ignore previous instructions'
   --> examples/poisoned-skill/SKILL.md:17:25
  critical[AGT001]: Bidirectional control character U+202E RIGHT-TO-LEFT OVERRIDE at line 19, col 20
   --> examples/poisoned-skill/SKILL.md:19:20
  critical[AGT001]: Bidirectional control character U+202C POP DIRECTIONAL FORMATTING at line 19, col 28
   --> examples/poisoned-skill/SKILL.md:19:28

Found 6 finding(s) in 1 file(s): 4 critical, 2 error, 0 warn, 0 info
```

Each finding links to a rule with an explanation (`agentlint explain AGT040`), and the Unicode findings can be repaired automatically with `agentlint fix`.

---

## Supported File Types

| File | Detection |
|------|-----------|
| `SKILL.md` | Skill files with YAML frontmatter |
| `AGENTS.md`, `CLAUDE.md` | Project memory files |
| `.cursorrules`, `.clinerules`, `.windsurfrules` | IDE agent rules |
| `.github/copilot-instructions.md` | GitHub Copilot instructions |
| `claude_desktop_config.json` + MCP configs | MCP server lists |
| `*.md` with `--treat-as agent-instructions` | Generic instruction files |

## Detection Categories

See [RULES.md](RULES.md) for the full taxonomy.

| Range | Category | Examples |
|-------|----------|---------|
| AGT001–AGT007 | Unicode / encoding | Bidi chars, zero-width chars, tag chars, confusables |
| AGT020–AGT024 | Instruction override | "Ignore previous instructions", roleplay jailbreaks, system impersonation |
| AGT040–AGT043 | Exfiltration sinks | Image URL data exfil, URL shorteners, sensitive path reads |
| AGT060–AGT064 | Dangerous MCP tool patterns | Unpinned npx, private URLs, filesystem+HTTP combos |
| AGT080–AGT082 | Skill metadata hygiene | Missing triggers, external file refs, safety-disable instructions |

## Installation

```bash
cargo install --path crates/agentlint-cli
```

## Usage

```bash
# Scan a file
agentlint check SKILL.md

# Scan a directory (respects .gitignore)
agentlint check .

# Minimum severity filter
agentlint check . --severity error

# Output formats
agentlint check SKILL.md --format json
agentlint check SKILL.md --format sarif > results.sarif

# Read from stdin
cat SKILL.md | agentlint check --stdin --filename SKILL.md

# Auto-fix fixable findings
agentlint fix SKILL.md

# Explain a specific rule
agentlint explain AGT020

# List all rules
agentlint list-rules
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | No findings at or above the threshold |
| 1 | One or more findings at or above the threshold |
| 2 | Internal error |

## GitHub Action

Add to your workflow:

```yaml
- name: Scan agent configuration files
  uses: thenayanhazra/Skill-Agent-File-Linter@main
  with:
    paths: '.'
    severity: warn
    sarif-output: agentlint.sarif

- name: Upload results to code scanning
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: agentlint.sarif
```

The action annotates pull requests with inline findings and uploads SARIF to GitHub code scanning.

## Rule Documentation

See [RULES.md](RULES.md) for full descriptions, examples, and false-positive guidance for each rule.

## License

MIT
