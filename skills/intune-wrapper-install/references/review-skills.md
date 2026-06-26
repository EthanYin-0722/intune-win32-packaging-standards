# Review Skills

Use this workflow when the user says `review-skills`, asks to add a packaging failure to this skill, or asks the skill to self-update after validation.

## Purpose

Turn a concrete packaging failure into a reusable skill lesson so the same class of issue is less likely to happen again.

## Rules

- Update the smallest relevant skill file.
- Generalize the lesson; do not add package-specific paths, product names, or one-off log noise unless they are needed as an example.
- Prefer `references/wrapper-standards.md` for coding rules and `references/scenario-tests.md` for observed failure scenarios.
- Preserve the hard confirmation gate and validation fix gate.
- Do not rewrite unrelated skill sections.
- After editing, run lightweight validation: frontmatter exists, required fields exist, referenced files exist, and new wording can be found.

## Failure Summary Format

When adding a lesson, capture:

1. `Observed failure`: concise error or behavior.
2. `Root cause`: generalized technical cause.
3. `Prevention`: wrapper pattern to use next time.
4. `Validation signal`: how future validation would reveal the issue.

## Common Lessons

Strict-mode registry access:

- Missing uninstall registry properties can throw under `Set-StrictMode -Version Latest`.
- Read optional properties through `PSObject.Properties` before filtering.

Post-install cleanup scope:

- Cleanup after install must not remove the installed app directory or detection target.
- Split pre-install leftover cleanup from post-install shortcut/config cleanup.
- Run final validation after post-install cleanup.

Optional uninstall validation:

- Ask before testing uninstall even when install validation passed.
- Run uninstall only after explicit confirmation because it removes the app from the local test machine.
- Verify detection no longer finds the app and ask before reinstalling.
