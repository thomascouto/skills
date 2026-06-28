---
name: check-deps
description: >
  Use this skill whenever creating a frontend project from scratch or adding
  dependencies to an existing one. It ensures all versions are resolved
  dynamically via the npm registry (never from memory), that repo release
  policies are respected, and that the generated scaffold is minimal,
  extensible, and framework-agnostic.
  Triggers: "create a project", "scaffold a FE project", "add dependencies",
  "set up the project", any generation of package.json.
requires:
  - bash_tool (required — do not run this skill without bash available)
  - ARCH_PREFERENCES.md (optional — read if present in the repo or skill folder)
compatibility: "Claude Desktop, Claude Code, Cowork — any environment with bash_tool"
---

# check-deps — Frontend dependencies and scaffold with current versions

## Why this skill exists

Language models have a training cutoff. When they generate a `package.json`
from memory, they use versions that may be 1–2 years out of date. This skill
eliminates that problem: **no version is written without first being resolved
via the npm registry**.

---

## Phase 1 — Repository analysis

Before writing any file, inspect the repository for release policies and
already-declared preferences. Run the commands below **at the project root**
(or the target directory if creating from scratch).

### 1.1 Files to inspect (in priority order)

```bash
# 1. Declared package manager and engines
cat package.json 2>/dev/null | grep -E '"packageManager"|"engines"' || echo "not found"

# 2. npm/pnpm config — minimumReleaseAge and similar
cat .npmrc 2>/dev/null || echo "not found"
cat .pnpmfile.cjs 2>/dev/null || echo "not found"

# 3. Renovate — automated update policy
cat renovate.json 2>/dev/null || cat .renovaterc 2>/dev/null || cat .renovaterc.json 2>/dev/null || echo "not found"

# 4. Dependabot
cat .github/dependabot.yml 2>/dev/null || echo "not found"

# 5. Textual mentions in docs
grep -ri "minimumReleaseAge\|release.*age\|minimum.*age\|stability.*days\|wait.*days" \
  docs/ CONTRIBUTING* ARCHITECTURE* README* 2>/dev/null | head -20 || echo "not found"
```

### 1.2 Policy extraction

After inspection, determine:

| Field | Extracted value | Fallback |
|-------|----------------|---------|
| `minimumReleaseAge` | value found (e.g. `"3 days"`) | `0` (no restriction) |
| `packageManager` | e.g. `pnpm@9`, `npm` | detect via `which pnpm npm yarn` |
| `nodeVersion` | value from `engines.node` | current LTS |

Store these values internally — they drive Phase 2.

### 1.3 Reading ARCH_PREFERENCES.md

```bash
# Search in the repo and in the skill folder
cat ARCH_PREFERENCES.md 2>/dev/null || \
cat "$(dirname "$0")/ARCH_PREFERENCES.md" 2>/dev/null || \
echo "ARCH_PREFERENCES not found — using defaults"
```

If found, any filled-in decisions in `ARCH_PREFERENCES.md` **override
any default from this skill**. Commented or empty entries are ignored.

---

## Phase 2 — Version resolution

**Absolute rule: never write a version from memory.**
Run the following flow for every dependency being added:

### 2.1 Resolution algorithm

```bash
# Step 1: get the latest stable version
LATEST=$(npm info <package> version 2>/dev/null)

# Step 2: get the publish date of this version
PUB_DATE=$(npm info <package> time.$LATEST 2>/dev/null)

# Step 3: calculate age in days
PUB_EPOCH=$(date -d "$PUB_DATE" +%s 2>/dev/null || date -j -f "%Y-%m-%dT%H:%M:%S" "$PUB_DATE" +%s)
NOW_EPOCH=$(date +%s)
AGE_DAYS=$(( (NOW_EPOCH - PUB_EPOCH) / 86400 ))

# Step 4: apply policy
if [ "$AGE_DAYS" -ge "$MIN_RELEASE_AGE" ]; then
  VERSION=$LATEST
else
  # Fallback: previous stable version
  VERSION=$(npm info <package> versions --json 2>/dev/null | \
    node -e "const v=require('fs').readFileSync('/dev/stdin','utf8');
             const list=JSON.parse(v);
             console.log(list[list.length-2])")
fi

echo "$VERSION"
```

### 2.2 Batch resolution helper

To resolve multiple dependencies at once:

```bash
resolve_version() {
  local pkg=$1
  local min_age=${2:-0}  # days, default = 0 (no restriction)

  local latest
  latest=$(npm info "$pkg" version 2>/dev/null)
  [ -z "$latest" ] && { echo "ERROR: package $pkg not found"; return 1; }

  if [ "$min_age" -eq 0 ]; then
    echo "$latest"
    return
  fi

  local pub_date age_days
  pub_date=$(npm info "$pkg" "time.$latest" 2>/dev/null | tr -d '"')
  age_days=$(( ( $(date +%s) - $(date -d "$pub_date" +%s 2>/dev/null || echo 0) ) / 86400 ))

  if [ "$age_days" -ge "$min_age" ]; then
    echo "$latest"
  else
    # Previous stable version
    npm info "$pkg" versions --json 2>/dev/null | \
      node -e "
        let v=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
        let stable=v.filter(x=>!x.includes('-'));
        console.log(stable[stable.length-2]||stable[stable.length-1]);
      "
  fi
}

# Usage example:
# MIN_AGE=3  # days, read from repo policy
# react_v=$(resolve_version react $MIN_AGE)
# typescript_v=$(resolve_version typescript $MIN_AGE)
```

### 2.3 Packages that must always be resolved dynamically

Regardless of framework, always resolve:

**Runtime / core**
- `typescript`
- `vite` or the project's build tool
- `@types/node`

**Code quality** (per `ARCH_PREFERENCES.md` or default)
- linter: `eslint` + relevant plugins, or `@biomejs/biome`
- formatter: `prettier`, or included in biome

**Testing** (per `ARCH_PREFERENCES.md` or default)
- `vitest` (default) or `jest`
- `@testing-library/*` if there is UI

**Environment types**
- `vite/client` or the build tool equivalent

> For framework packages (e.g. `react`, `vue`, `svelte`), resolve the same
> way — never assume the major version.

---

## Phase 3 — Minimal and extensible scaffold

Generate only the core files listed below. No example files,
no pre-created `src/` folder, no configuration beyond the essential.

### 3.1 `package.json`

```json
{
  "name": "<project-name>",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "scripts": {
    "dev":   "<build tool dev command>",
    "build": "<build command>",
    "check": "<type-check command>",
    "test":  "<test runner command>",
    "lint":  "<linter command>"
  },
  "dependencies": {
    // runtime only — resolved via Phase 2
  },
  "devDependencies": {
    // typescript, build tool, linter, types — resolved via Phase 2
  },
  "engines": {
    "node": ">=<detected LTS version>"
  }
}
```

**Rules:**
- Exact versions — no `^` or `~` prefix.
- No auto-generated `peerDependencies`.
- No redundant scripts (`prebuild`, unnecessary `postinstall`).

### 3.2 `tsconfig.json`

```jsonc
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,              // adjustable via ARCH_PREFERENCES
    "skipLibCheck": true,
    "isolatedModules": true,
    "noEmit": true               // build via bundler, not tsc
  },
  "include": ["src"]
}
```

Add `paths` only if `ARCH_PREFERENCES.md` declares aliases.
Do not add commented-out options — genuinely minimal.

### 3.3 Build tool config file

**Vite (default when not specified):**

```ts
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  // plugins added per framework (e.g. react(), vue())
})
```

**If another build tool is chosen** (webpack, turbopack, rspack, etc.),
generate the minimal equivalent file following the same philosophy.

### 3.4 Linter / formatter

**If `ARCH_PREFERENCES.md` specifies `biome`:**

```jsonc
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/<resolved-version>/schema.json",
  "formatter": { "enabled": true, "indentStyle": "space" },
  "linter":    { "enabled": true, "rules": { "recommended": true } },
  "javascript": { "formatter": { "quoteStyle": "single" } }
}
```

**If it specifies `eslint` (or no preference — eslint is the default):**

```js
// eslint.config.js  (flat config — required for ESLint >=9)
import js from '@eslint/js'
import tseslint from 'typescript-eslint'

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended,
)
```

Do not generate `.eslintrc.*` — the legacy format is deprecated.

### 3.5 `.gitignore`

```
node_modules/
dist/
.env
.env.*
!.env.example
```

---

## Phase 4 — Validation checklist

Before delivering the scaffold to the user, run and report this checklist.
**Each item must have an explicit status: ✅ or ❌ + reason.**

```
VALIDATION CHECKLIST — check-deps
─────────────────────────────────────────────────────────────
[ ] 1. VERSIONS RESOLVED DYNAMICALLY
        Were all versions obtained via `npm info` in this session?
        Was no version written from memory or training data?

[ ] 2. RELEASE POLICY RESPECTED
        Was a policy search performed in the repo (Phase 1)?
        If policy found: do versions respect minimumReleaseAge?
        If no policy: was "no restriction detected" recorded?

[ ] 3. ARCH_PREFERENCES APPLIED
        Was ARCH_PREFERENCES.md read (or its absence recorded)?
        Were all filled-in decisions applied to the scaffold?
        Does no prohibition from the `never:` section appear in deps?

[ ] 4. MINIMAL SCAFFOLD
        Were no files beyond the 4 core generated without explicit request?
        Was no `src/` folder or directory structure created?
        Are there no configs with unnecessarily commented-out options?

[ ] 5. TYPESCRIPT CORRECTLY CONFIGURED
        Is `moduleResolution` compatible with the chosen build tool?
        Is `noEmit: true` present (build via bundler)?
        Is `strict: true` set, or as specified in ARCH_PREFERENCES?

[ ] 6. FUNCTIONAL SCRIPTS
        Were all package.json scripts verified?
        Are commands correct for the resolved build tool version?

[ ] 7. VERSION CONSISTENCY
        Is the `@types/node` version compatible with the detected Node version?
        Are linter plugins compatible with the resolved linter version?
─────────────────────────────────────────────────────────────
```

If any item fails, fix it before presenting the scaffold.
Report the full checklist to the user at the end.

---

## Edge cases

**No internet access in bash:** If `npm info` fails due to network restrictions,
tell the user explicitly: *"Could not resolve versions dynamically. I recommend
running manually: `npm info <package> version`"*. Do not silently fall back to
versions from memory.

**Monorepo with `workspaces`:** If detected (`workspaces` in the root
`package.json` or presence of `pnpm-workspace.yaml`), generate the child
package's `package.json` without duplicating deps already declared at the root.
Inform the user about what was omitted.

**Framework with an official CLI** (e.g. `create-react-app`, `nuxi`, `@sveltejs/kit`):
Ask the user whether they prefer to use the official CLI (which already handles
versions) before generating the scaffold manually. This skill is the fallback
when the CLI is not desired.
