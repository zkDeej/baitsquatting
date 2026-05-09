# /npm-trust-check

Audit all npm dependencies in this project for supply chain trust signals,
including the install=scoped / run=bare binary name split, ephemeral npx
execution without lockfile protection, and lockfile integrity verification.

**Version:** 2.0 — updated 2026-05-08 following BSQT-2026-001 baitsquatting disclosure.

## Rules — read first, follow always

- Read this file once per session before running the command
- Do not skip steps or abbreviate the output
- This is a supply-chain audit: completeness > speed
- Never execute, install, or modify any file — report findings only
- If no `package.json` exists, report: `No package.json found — this does not appear to be an npm project` and stop
- For monorepos with workspaces, check all `packages/*/package.json` and `apps/*/package.json` in addition to the root
- Collect every entry from `dependencies` and `devDependencies`
- Separate into two lists: **scoped** (`@org/name`) and **unscoped** (`name`)

---

## Step 1 — Read lockfile

Check for `package-lock.json` or `yarn.lock` in the project root.

If **no lockfile exists**: report immediately as 🚨 CRITICAL — no lockfile found.
A missing lockfile means every `npm install` and every `npx` call resolves
packages fresh from the registry with no pinned integrity guarantee. This is
the highest-risk configuration for baitsquatting exposure. Recommend: run
`npm install` to generate `package-lock.json` and commit it before proceeding.

If lockfile exists: parse it. For each resolved package entry, extract:
- `name`, `version`, `resolved` URL, `integrity` hash (sha512)
- Note any entry whose `resolved` URL is not `https://registry.npmjs.org/` —
  these are private registry, GitHub, or git installs and need explicit review.

Store this as the **lockfile map** for use in Step 4b.

---

## Step 2 — Fetch scoped package registry data + build binary map

Scoped packages are not automatically safe. They may expose bare binary names that
can be squatted separately on npm.

For each scoped package, fetch **concurrently**:
```
https://registry.npmjs.org/<encoded-scoped-name>
```
(e.g. `https://registry.npmjs.org/%40lhci%2Fcli` for `@lhci/cli`)

From the `versions[dist-tags.latest]` object, extract:
- `bin` field — a map of `{ "binary-name": "path" }`. If absent, no binary exposed.

Build a **binary map**: `{ "lhci": "@lhci/cli", "sentry-cli": "@sentry/cli", ... }`
covering every binary name exposed by every scoped package in this project.

---

## Step 3 — Binary collision check (scoped packages)

For each entry in the binary map, check whether that bare name exists as a separate
npm package. Fetch **concurrently**:
```
https://registry.npmjs.org/<binary-name>
https://api.npmjs.org/downloads/point/last-week/<binary-name>
```

For each binary name, report one of:
- **Collision** — a package with this name exists on npm, published by a different
  entity than the scoped package owner. `npx <binary-name>` resolves to the
  collision package, NOT to the scoped package's binary.
- **No collision** — name does not exist as a standalone npm package. Still prefer
  `npx @scope/pkg@version` form in CI for determinism.
- **Unverified** — registry unreachable.

Flag any collision as 🚨 HIGH regardless of download counts — intent does not
matter, the resolution path is wrong.

For each collision, output the safe alternative run command:
```
Unsafe:  npx lhci autorun
Safe:    npx @lhci/cli@<version> autorun
```

---

## Step 4a — Fetch unscoped package registry data

For each unscoped package, make **one** HTTP GET request **concurrently**:
```
https://registry.npmjs.org/<package-name>
```

Extract:
- `maintainers`
- `time.created`
- `repository.url` / `homepage`
- `dist-tags.latest`
- `bin` field of the latest version (note any binary names it exposes)
- `_npmUser` of the latest version (the publisher account)

Then fetch download stats (also in parallel):
```
https://api.npmjs.org/downloads/point/last-week/<package-name>
```

Also check: does this unscoped package's name appear in the binary map from Step 2?
If yes, flag as a **known binary collision** (already reported in Step 3).

---

## Step 4b — Lockfile integrity verification

For each package in the **lockfile map** from Step 1:

**Check A — Integrity hash present:**
If the `integrity` field is absent from a lockfile entry, flag as ⚠️ UNVERIFIED —
this package was added without hash verification and could be tampered.

**Check B — Resolved URL:**
If `resolved` URL is not `https://registry.npmjs.org/`, flag as ⚠️ NON-REGISTRY SOURCE —
requires explicit review. GitHub URLs, private registries, and git references
bypass standard npm security checks.

**Check C — Publisher age cross-check:**
For any package where the lockfile `version` matches a release published less than
6 months ago, cross-reference the publisher from Step 4a. A recently published
version from a new or single-maintainer account is elevated risk.

Report all flagged entries in the output under `LOCKFILE INTEGRITY`.

---

## Step 5 — Score each unscoped package

Apply signals independently. Use **exclusive ranges** for download counts (add only one).

| Signal | Points |
|---|---|
| First published less than 6 months ago | +2 |
| No homepage and no repository link | +2 |
| Name matches a binary exposed by a scoped package in this project (binary collision) | +3 |
| Name is a bare variant of a known scoped package (e.g. `lhci` where `@lhci/cli` has 100× more downloads) | +3 |
| Weekly downloads under 1,000 | +2 |
| Weekly downloads 1,000–10,000 | +1 |
| Weekly downloads over 10,000 | +0 |
| Single maintainer | +1 |
| No homepage link | +1 |
| No repository link | +1 |
| **Total** | **Score: 0–18** |

Grade:
- **0–2:** Low risk ✅
- **3–5:** Medium risk ⚠️
- **6+:** High risk 🚨

---

## Step 6 — Scan CI and scripts for bare binary calls

Search these locations:
- `.github/workflows/*.yml`
- `.gitlab-ci.yml`
- `Dockerfile` and `docker-compose.yml`
- `package.json` scripts section
- Root `Makefile`
- Any `.sh` files in project root or `scripts/`

For each line containing `npx <name>` or direct binary invocation `<name> <args>`:

**Check 1 — Binary map match:**
Cross-reference `<name>` against the binary map built in Step 2.
If matched: the bare call resolves to the npm registry package `<name>`, NOT to the
scoped package's installed binary. Flag as 🚨 regardless of whether a collision
exists today — the resolution path is fragile by design.
Output the pinned safe alternative: `npx @scope/pkg@<pinned-version> <args>`

**Check 2 — Unscoped package in dependencies:**
If `<name>` matches an unscoped package in `dependencies` or `devDependencies`,
check its trust score from Step 5. Report score inline.

**Check 3 — Unknown bare name:**
If `<name>` matches neither the binary map nor a listed dependency, flag as
⚠️ Unresolved — binary origin unknown. Could be a global install assumption,
a PATH dependency, or an unlisted package.

**Check 4 — Ephemeral execution (highest severity):**
If `<name>` is absent from both `dependencies` and `devDependencies` entirely,
flag as 🚨 EPHEMERAL EXECUTION — this package is downloaded fresh from the
registry on every run with zero lockfile protection. No integrity hash. No pinned
version. If the bare namespace is squatted today, every future CI run executes
the squatter's code.

This is the exact pattern behind the BSQT-2026-001 live incident:
`npx lhci autorun` — `lhci` was not in package.json, downloaded fresh, resolved
to a third-party account with full shell credential access.

Output for each ephemeral call:
```
🚨 EPHEMERAL EXECUTION
  File:    .github/workflows/ci.yml:42
  Command: npx lhci autorun
  Risk:    No lockfile entry. Downloaded fresh on every run. No integrity guarantee.
  Fix:     Add @lhci/cli to devDependencies, commit lockfile, use:
           npx @lhci/cli@<pinned-version> autorun
```

Do not modify any file — report findings only.

---

## Step 7 — Legacy package and AI hallucination scan

Check all package names in `dependencies` and `devDependencies` against this
known-risk table. These are packages where AI coding agents commonly suggest
the old or bare name instead of the correct current scoped name — drawn from
training data patterns and cross-referenced against the UTSA slopsquatting
dataset (Spracklen et al., USENIX Security 2025).

| Old / bare name | Current canonical name | Source |
|---|---|---|
| `react-query` | `@tanstack/react-query` | AI training data |
| `@material-ui/core` | `@mui/material` | AI training data |
| `@material-ui/icons` | `@mui/icons-material` | AI training data |
| `apollo-client` | `@apollo/client` | AI training data |
| `aws-sdk` | `@aws-sdk/client-*` (v3 modular) | AI training data |
| `clerk-nextjs` | `@clerk/nextjs` | UTSA hallucination dataset |
| `clerk-js` | `@clerk/clerk-js` | UTSA hallucination dataset |
| `clerk-remix` | `@clerk/remix` | UTSA hallucination dataset |
| `prisma-client` | `@prisma/client` | UTSA hallucination dataset |
| `stripe-js` | `@stripe/stripe-js` | UTSA hallucination dataset |
| `supabase-js` | `@supabase/supabase-js` | UTSA hallucination dataset |
| `trpc-client` | `@trpc/client` | UTSA hallucination dataset |
| `sentry-browser` | `@sentry/browser` | UTSA hallucination dataset |
| `aws-sdk-client-s3` | `@aws-sdk/client-s3` | UTSA hallucination dataset |
| `aws-sns` | `@aws-sdk/client-sns` | UTSA hallucination dataset |
| `typeorm` | Still `typeorm` — no migration needed | (false positive guard) |

If an old/bare name is found: flag as ⚠️ LIKELY AI-GENERATED — report the canonical replacement.
If a project uses both old and new names: flag as 🚨 DUAL DEPENDENCY — likely a migration in progress or an AI-generated scaffold that was partially corrected.

---

## Step 8 — Output

Structure output with