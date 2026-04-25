# ai-coding-skills

A growing collection of AI coding skills for software engineering workflows. Each skill teaches the agent a structured, repeatable process for a specific coding task — so you get consistent, disciplined results instead of ad-hoc responses.

---

## Compatibility

Use these skills with any AI coding agent that supports the SKILL.md format. To install simply run the command:
```
npx skills add ma-armenta/ai-coding-skills
```

---

## Skills

### [`codebase-cleanup`](./codebase-cleanup/)

Systematic codebase hardening across 7 phases: DRY consolidation, type deduplication, dead code removal, circular dependency resolution, type strengthening, error handling cleanup, and legacy/AI artifact removal. Language-agnostic — works with TypeScript, Python, Go, Rust, Java, and more.

**Triggers on:** *"clean up the codebase", "remove dead code", "fix circular imports", "remove `any` types", "strip placeholder comments"*, and similar.

---

> More skills coming. Each one targets a specific engineering workflow where a structured approach beats a one-off prompt.

---

## Contributing

Skill format follows the `SKILL.md` standard with YAML frontmatter:

```yaml
---
name: skill-name
description: >
  What this skill does and when the agent should use it.
---
```

PRs welcome. Open an issue first if you want to discuss a new skill before building it.

---

## License

MIT