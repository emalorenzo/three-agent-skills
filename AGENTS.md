# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Repository Overview

A collection of agent skills for Three.js development. Skills are packaged instructions and scripts that extend AI agent capabilities for 3D web development, shader programming, and WebGL optimization.

## Creating a New Skill

### Directory Structure

```
skills/
  {skill-name}/           # kebab-case directory name
    SKILL.md              # Required: skill definition
    scripts/              # Optional: executable scripts
      {script-name}.sh    # Bash scripts (preferred)
    references/           # Optional: supporting documentation
      *.md                # Additional reference files
  {skill-name}.zip        # Required for distribution: packaged skill
```

### Naming Conventions

- **Skill directory**: `kebab-case` (e.g., `shader-optimization`, `scene-setup`)
- **SKILL.md**: Always uppercase, always this exact filename
- **Scripts**: `kebab-case.sh` (e.g., `analyze.sh`, `generate-boilerplate.sh`)
- **Zip file**: Must match directory name exactly: `{skill-name}.zip`

### SKILL.md Format

Every skill must have a `SKILL.md` file with this structure:

```markdown
---
name: {skill-name}
description: {One sentence describing when to use this skill. Include trigger phrases.}
argument-hint: <optional-args>  # Optional: hint for arguments
license: MIT
metadata:
  author: {author-name}
  version: "{semver}"
---

# {Skill Title}

{Brief description of what the skill does.}

## When to Apply

Reference these guidelines when:
- {Scenario 1}
- {Scenario 2}
- {Scenario 3}

## How It Works

{Numbered list explaining the skill's workflow}

1. Step one
2. Step two
3. Step three

## Usage

```bash
bash /mnt/skills/user/{skill-name}/scripts/{script}.sh [args]
```

**Arguments:**
- `arg1` - Description (defaults to X)

**Examples:**
{Show 2-3 common usage patterns}

## Quick Reference

{Concise summary of rules/patterns organized by category}

## Output

{Show example output users will see}

## Present Results to User

{Template for how the agent should format results when presenting to users}

## Troubleshooting

{Common issues and solutions}
```

### Best Practices for Context Efficiency

Skills are loaded on-demand. Only the skill name and description are loaded at startup. The full `SKILL.md` loads into context only when the agent decides the skill is relevant.

To minimize context usage:

- **Keep SKILL.md under 500 lines** - put detailed reference material in separate files
- **Write specific descriptions** - helps the agent know exactly when to activate the skill
- **Use progressive disclosure** - reference supporting files that get read only when needed
- **Prefer scripts over inline code** - script execution doesn't consume context (only output does)
- **File references work one level deep** - link directly from SKILL.md to supporting files

### Script Requirements

If your skill includes scripts:

```bash
#!/bin/bash
set -e  # Fail-fast behavior

# Write status messages to stderr
echo "Processing..." >&2

# Write machine-readable output (JSON) to stdout
echo '{"result": "success"}'

# Include cleanup trap for temp files
cleanup() {
    rm -rf "$TEMP_DIR"
}
trap cleanup EXIT
```

Key requirements:
- Use `#!/bin/bash` shebang
- Use `set -e` for fail-fast behavior
- Write status messages to stderr: `echo "Message" >&2`
- Write machine-readable output (JSON) to stdout
- Include a cleanup trap for temp files
- Reference the script path as `/mnt/skills/user/{skill-name}/scripts/{script}.sh`

### Creating the Zip Package

After creating or updating a skill:

```bash
cd skills
zip -r {skill-name}.zip {skill-name}/
```

### End-User Installation

**Claude Code:**
```bash
cp -r skills/{skill-name} ~/.claude/skills/
```

**claude.ai:**
Add the skill to project knowledge or paste SKILL.md contents into the conversation.

If the skill requires network access, instruct users to add required domains at `claude.ai/settings/capabilities`.

## Skill Types

### 1. Guidelines Skills (No Scripts)

Pure documentation skills that provide rules and best practices. Example: `react-best-practices`.

Structure:
```
skill-name/
  SKILL.md          # Rules and guidelines
  rules/            # Optional: individual rule files
    rule-1.md
    rule-2.md
```

### 2. Tool Skills (With Scripts)

Skills that execute actions via bash scripts. Example: `vercel-deploy`.

Structure:
```
skill-name/
  SKILL.md          # Usage instructions
  scripts/
    main-action.sh  # Primary script
    helper.sh       # Supporting scripts
```

### 3. Hybrid Skills

Combine guidelines with automation tools.

Structure:
```
skill-name/
  SKILL.md          # Instructions + quick reference
  AGENTS.md         # Full compiled guidelines
  scripts/
    analyze.sh      # Automation scripts
  rules/
    detailed-rules/
```

## Frontmatter Reference

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier (kebab-case) |
| `description` | Yes | One sentence with trigger phrases |
| `argument-hint` | No | Hint for arguments (e.g., `<file-or-pattern>`) |
| `license` | No | License type (default: MIT) |
| `metadata.author` | No | Author name |
| `metadata.version` | No | Semantic version |

## Testing Your Skill

1. **Local Testing**: Copy to `~/.claude/skills/` and test with Claude Code
2. **Description Testing**: Verify the agent activates on expected trigger phrases
3. **Script Testing**: Run scripts manually to verify behavior
4. **Context Testing**: Ensure SKILL.md stays under 500 lines

## Example Skills for Three.js

Future skills in this repository:

- `three-scene-setup` - Best practices for Three.js scene initialization
- `shader-optimization` - GLSL shader performance patterns
- `three-performance` - Rendering optimization guidelines
- `three-postprocessing` - Post-processing effects setup
- `three-physics` - Physics engine integration patterns
