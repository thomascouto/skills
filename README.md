# Skills

A collection of reusable agent skills for software development workflows.

---

## Catalog

### `check-deps`

**Path:** `check-deps/SKILL.md`

Ensures frontend dependencies are resolved dynamically via the npm registry — never from model memory. Use when creating a project from scratch or adding dependencies to an existing one.

**Triggers:** "create a project", "scaffold a FE project", "add dependencies", "set up the project", any generation of `package.json`.

**Phases:**
1. **Repository analysis** — reads `package.json`, `.npmrc`, `renovate.json`, `dependabot.yml`, and `ARCH_PREFERENCES.md` to extract release policies and architectural preferences.
2. **Version resolution** — every version is fetched via `npm info` and validated against the repo's `minimumReleaseAge` policy before being written.
3. **Minimal scaffold** — generates only the core files (`package.json`, `tsconfig.json`, build tool config, linter config, `.gitignore`). No example files, no pre-created `src/` folder.
4. **Validation checklist** — verifies versions, policy compliance, ARCH_PREFERENCES application, and script correctness before delivering output.

**Companion file:** `check-deps/ARCH_PREFERENCES.md` — fill in your architectural decisions (linter, test runner, build tool, package manager, TypeScript strictness, explicit bans) to override skill defaults.

**Requires:** `bash_tool`

---

## Adding a Skill

Each skill lives in its own directory:

```
<skill-name>/
  SKILL.md           # skill definition with frontmatter (name, description, requires)
  ARCH_PREFERENCES.md  # optional companion for user-configurable preferences
```

The `SKILL.md` frontmatter must include at minimum:

```yaml
---
name: <skill-name>
description: >
  When to use this skill (triggers, conditions).
---
```
