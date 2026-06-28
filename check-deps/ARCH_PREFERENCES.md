# ARCH_PREFERENCES.md
# Personal architectural preferences — edit manually
#
# This file is read by the `check-deps` skill before generating any scaffold.
# Filled-in entries here OVERRIDE the skill's defaults.
# Leave a section empty or comment it with # to use the default.
#
# Entry format:
#   decision: what was decided
#   reason:   why (optional, but recommended for onboarding)
#   applies:  always | when:<condition> (e.g. when:react, when:monorepo)

---

## Tooling

# Linter / formatter
# decision: ""
# reason: ""
# applies: always
# Example values: "biome", "eslint+prettier", "eslint-only"

# Test runner
# decision: ""
# reason: ""
# applies: always
# Examples: "vitest", "jest", "playwright-only"

# Build tool
# decision: ""
# reason: ""
# applies: always
# Examples: "vite", "turbopack", "webpack5"

# Package manager
# decision: ""
# reason: ""
# applies: always
# Examples: "pnpm", "npm", "yarn"

---

## TypeScript

# Strictness
# decision: ""
# reason: ""
# applies: always
# Examples: "strict", "strict + noUncheckedIndexedAccess", "moderate"

# Path aliases
# decision: ""
# reason: ""
# applies: always
# Examples: "@/* → src/*", "none"

---

## Code conventions

# Component style (when React)
# decision: ""
# reason: ""
# applies: when:react
# Examples: "functional only", "functional + hooks", "no class components"

# Global state
# decision: ""
# reason: ""
# applies: when:react
# Examples: "zustand", "context-only", "jotai", "none"

# Fetch / data fetching
# decision: ""
# reason: ""
# applies: always
# Examples: "tanstack-query", "swr", "native fetch only"

---

## Explicit bans
# List here what must never appear in a generated scaffold.
# The skill checks this list during the validation checklist.

# never:
#   - moment.js        # use date-fns or dayjs
#   - lodash           # use native or radash
#   - class-validator  # use zod
