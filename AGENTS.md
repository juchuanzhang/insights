# AGENTS.md

## Repo overview

Personal knowledge repo — Chinese-language markdown documents on microarchitecture security topics. No code, no build system, no tests, no CI.

## Structure

- `microarchitecture-attack-defense/` — v1.0 (superseded by v2.0)
- `microarchitecture-attack-defense-new/` — v2.0 (active, contains main document + `SESSION_CONTEXT.md`)
- `sp2026-cross-domain-test/` — Enter-Exit-SP26 paper analysis: 4 files (markdown notes, analysis report md, PPT html, PDF)
- `spectre-v4/` — tracked but empty (no committed content)

## Conventions

- All content is in **Chinese (简体中文)**. Preserve this when editing.
- Commit messages are also in Chinese, prefixed with `新增:`, `勘误:`, `同步勘误:`, etc. Follow this pattern.
- Only branch: `main`. No PR workflow — direct commits + push.
- `SESSION_CONTEXT.md` in `microarchitecture-attack-defense-new/` serves as a cross-device session restore file; update it when making significant changes to that directory.
- When working on `microarchitecture-attack-defense-new/`, operate only within that directory — per user's documented preference in `SESSION_CONTEXT.md`.

## Workflow

```bash
git add <changed-dir>
git commit -m "新增/勘误: 描述"
git push origin main
```

No lint, typecheck, or test steps needed.

## Gotchas

- v1.0 in `microarchitecture-attack-defense/` is outdated; always use v2.0 in `microarchitecture-attack-defense-new/` as the working copy.
- File names use Chinese characters — be careful with encoding in git operations on Windows.
