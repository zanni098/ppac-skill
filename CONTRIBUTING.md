# Contributing to PPAC

Thanks for helping make AI-built apps safer. The most useful contributions are checks that catch real failures.

## Good contributions

- **Stack-specific verification commands** for Django, Rails, Laravel, Go, Spring, mobile backends, and others.
- **New checks** that catch a real failure you have seen in shipped code.
- **False-positive fixes** when a check flags something that is actually fine.
- **Clarity edits** to the wording in `ppac/SKILL.md`.

## Principles to keep

1. **Every check must be verifiable.** Include a concrete way to confirm it (a command, a file to read, a test to run). "Make sure it is secure" is not a check.
2. **Respect the tiers.** Tier 1 is for things that cause breaches or real cost. Do not promote scaling concerns into Tier 1.
3. **Smallest fix.** The skill closes holes. It should not rewrite apps.
4. **Keep it short.** If a check needs a paragraph to explain, it probably needs to be split or simplified.

## How to submit

1. Fork the repo and create a branch.
2. Edit `ppac/SKILL.md` (and the README checklist summary if you add or remove an item).
3. Open a pull request that explains the real failure the change catches, ideally with an example.

Not sure if an idea fits? Open an issue first using the **Checklist suggestion** template.
