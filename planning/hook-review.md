# Change Review

## Findings

### [P1] Add a valid plugin manifest

File: `independent-reviewer/.claude-plugin/plugin.json:1`

The manifest is empty, so it is not valid JSON and does not identify the plugin. Claude Code cannot discover or load `independent-reviewer` as a plugin in this state. Populate the file with a valid manifest containing at least a plugin `name` (and the intended metadata).

### [P1] Make the plugin hooks file valid JSON

File: `independent-reviewer/hooks/hooks.json:1`

The file begins with `"hooks":` but has no enclosing top-level braces. JSON parsing therefore fails before Claude Code can register the Stop hook. Wrap the current content in `{ ... }` and validate the completed file as JSON.

### [P1] Use supported model values in the custom agents

Files: `.claude/agents/change-reviewer.md:4`, `.claude/agents/reviewer.md:4`, `.claude/agents/security-check.md:4`

The agents specify `GPT-4.1` and `Sonnet 5`. Claude Code agent frontmatter expects a supported Claude alias such as `sonnet`, `opus`, `haiku`, or `inherit`, or a valid full Claude model ID. These values can prevent agent creation. Replace them with supported identifiers or omit `model` to inherit the caller's model.

### [P1] Correct the `change-reviewer` tool allowlist

File: `.claude/agents/change-reviewer.md:5`

The `tools` field is an allowlist, but it contains generic lowercase names (`execute`, `read`, `search`, `web`, `agent`, and `todo`) rather than Claude Code's exact tool identifiers. The agent may consequently lack the inspection and write tools required to produce `change-review.md`. Use valid names such as `Read`, `Grep`, `Glob`, `Bash`, `Write`, and `Edit`, limited to what the reviewer actually needs.

### [P2] Give `security-check` review instructions

File: `.claude/agents/security-check.md:6`

The file ends immediately after its frontmatter. The `description` is delegation metadata, while the Markdown body supplies the subagent's operating prompt. Add a body defining what to inspect, severity criteria, and the expected output; otherwise the agent has no security-review-specific instructions.

## Summary

The README and plan updates accurately distinguish completed market-data work from the target architecture, and the root `.claude/settings.json` Stop hook is structurally valid. The independent plugin is currently unloadable, and the custom agents need corrected model/tool metadata before they can be relied on. `git diff --check` also flags the missing final newline in `.claude/settings.json`; that is cosmetic.
