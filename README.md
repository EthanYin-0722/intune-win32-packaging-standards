# Intune Win32 Packaging Standards

This repo has two clear lanes:

- `skills/` contains the installable Codex skill and its supporting reference files.
- `one-pager/` contains portable, single-file guidance that can be shared with colleagues who use another AI assistant or want a human-readable summary.

## Repo Tree

```text
intune-win32-packaging-standards/
  README.md
  .gitignore
  one-pager/
    README.md
    intune-wrapper-install-one-pager.md
  skills/
    intune-wrapper-install/
      SKILL.md
      agents/
        openai.yaml
      references/
        build-when-ready.md
        detection-validation.md
        official-doc-research.md
        powershell-template.md
        pre-final-review.md
        pre-intune-validation.md
        review-skills.md
        scenario-tests.md
        uninstall-validation.md
        wrapper-standards.md
```

## What To Use

Use `skills/intune-wrapper-install/` when installing or maintaining the Codex skill.

Use `one-pager/intune-wrapper-install-one-pager.md` when sharing the packaging standard with another AI agent or colleague. It is a consolidated summary of the same standards, not the source of truth for the Codex skill.
