# /sbom-generate

Generate a human-readable Software Bill of Materials (SBOM) for this npm project.
Identifies every dependency's publisher, publish date, integrity hash, and flags
anything anomalous — without installing, executing, or modifying any file.

Built for vibe coders: you don't need to know what an SBOM is to run this.
The output tells you who wrote every piece of code running in your project,
and whether anything looks wrong.

**Version:** 1.0 — 2026-05-08 · Part of the BSQT-2026-001 developer toolkit.

---

## Rules — read first, follow always

- Read this file once per session before running
- Never install, execute, or modify any file
- Never run `npm install`, `npx`, or any package manager command
- This command reads files and fetches registry data only
- If the user asks you to install something to generate the SBOM, refuse and explain
  that this command works without installing anything

---

## Step 1 — Check prerequisites

Look for these files in the project root:
- `package.json` — required. If absent: report `No package.json found` and stop.
- `package-lock.json` — strongly preferred. If absent: report as ⚠️ WARNING and
  continue in degraded mode (registry-only, no integrity verification).
  Explain: "A lockfile pins exact versions and integrity hashes. Without it, this
  SBOM reflects what the registry says today, not what is actually installed."

If `package-lock.json` is present, parse it fully. Extract for every package entry:
- `name`, `version`, `resolved` URL, `integrity` (sha512 hash), `dev` flag

---

## Step 2 — Build the full dependency inventory

From `package.json`: collect all names from `dependencies` and `devDependencies`.
Mark each as `runtime` or `dev`.

From `package-lock.json` (if present): collect the full resolved tree including
transitive dependencies. Note: this will be a much larger list than package.json
alone — that's expected and correct.

Count totals:
- Direct runtime dependencies
- Direct dev dependencies
- Transitive dependencies (from lockfile tree)
- Total unique packages

---

## Step 3 — Fetch publisher data from npm registry

For each unique package in the inventory, fetch **concurrently**:
```
https://registry.npmjs.org/<package-name>
```
For scoped packages, URL-encode the name:
```
https://registry.npmjs.org/%40org%2Fname
```

For each package, extract from the specific installed version (from lockfile) or
latest version (if no lockfile):
- `_npmUser.name` — the account that published this version
- `time.<version>` — when this version was published
- `maintainers` — full list of maintainers
- `repository.url` — source repository
- `dist.integrity` — expected sha512 hash for this version

Batch requests in groups of 20 to avoid rate limiting. Report progress if the
project has more than 50 packages.

---

## Step 4 — Integrity verification

For each package where both lockfile `integrity` and registry `dist.integrity`
are available:

Compare them. They must match exactly.

- ✅ **Match** — the installed package matches what the registry published.
- 🚨 **MISMATCH** — the hash in the lockfile does not match the registry.
  This is a critical finding. It means either: the lockfile was tampered with,
  the registry entry was modified after the lockfile was generated, or the
  package was installed from a different source than the registry.
  Report the specific package, both hashes, and recommend immediate investigation.
- ⚠️ **No lockfile hash** — package has no integrity entry in lockfile.
  Flag for review.
- ⚠️ **Resolved URL is not registry.npmjs.org** — package came from a private
  registry, GitHub, or git source. Not necessarily wrong, but requires explicit
  acknowledgement.

---

## Step 5 — Anomaly scoring

For each direct dependency (from package.json), apply these signals:

| Signal | Flag |
|---|---|
| Publisher account created less than 6 months before this version was published | ⚠️ New publisher |
| Single maintainer with no organisation affiliation | ⚠️ Solo publisher |
| Published version is less than 2 weeks old | ⚠️ Very recent publish |
| No repository link | ⚠️ No source |
| Resolved from non-registry URL | ⚠️ Non-registry source |
| Package name is a bare variant of a known scoped package | 🚨 Namespace risk |
| Integrity hash mismatch | 🚨 Integrity failure |
| Publisher changed from a previous version in the lockfile | 🚨 Publisher change |

For transitive dependencies: apply integrity check and publisher-change check only.
Full scoring for all transitive dependencies would produce too much noise.

---

## Step 6 — Output

Present results in three sections: summary, anomalies (action required), full SBOM table.

```
SOFTWARE BILL OF MATERIALS — <project name>
Generated: <date>
Lockfile: present (integrity verification active) / absent (registry-only mode)

━━━ SUMMARY ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Direct runtime deps:      X
  Direct dev deps:          X
  Transitive deps (total):  X
  Integrity verified:       X / X (where lockfile hash available)
  Anomalies found:          X 🚨 critical  X ⚠️ warnings

━━━ 🚨 CRITICAL FINDINGS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  (only shown if anomalies exist)

  Package: some-pkg@1.2.3
  Issue:   Integrity hash MISMATCH
  Lockfile: sha512-AAAA...
  Registry: sha512-BBBB...
  Action:  Investigate immediately. Do not deploy.

  Package: suspicious-pkg@0.0.4
  Issue:   Namespace risk — bare variant of @org/package
  Publisher: unknown-account (joined 2026-03-01)
  Action:  Verify this package is intentional. Consider replacing with @org/package.

━━━ ⚠️ WARNINGS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  some-other-pkg@2.1.0   Solo publisher | No repository | Published 8 days ago
  git-dep@main           Non-registry source (github.com/user/repo)

━━━ FULL SBOM — DIRECT DEPENDENCIES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Package                  Ver      Type     Publisher          Published    Integrity  Flags
  ─────────────────────────────────────────────────────────────────────────────────────────
  react                    19.0.0   runtime  react-bot (Meta)   2024-12-05   ✅ ok      —
  vite                     6.3.2    dev      patak-dev           2025-01-10   ✅ ok      —
  @lhci/cli                0.14.0   dev      patrickhulce        2024-09-01   ✅ ok      —
  some-pkg                 1.2.3    runtime  unknown-acct        2026-04-28   🚨 FAIL    New publisher

━━━ TRANSITIVE DEPENDENCY INTEGRITY ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  X transitive packages checked
  X ✅ integrity verified  X ⚠️ no hash  X 🚨 mismatch

━━━ RECOMMENDATIONS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  (generated based on findings)

  1. Investigate integrity mismatches before any deployment.
  2. Review packages flagged as namespace risks — check npmjs.com/package/<name>
     to verify the publisher matches the tool's known maintainer.
  3. Commit package-lock.json if not already tracked in git.
  4. For CI pipelines, use `npm ci` instead of `npm install` — it enforces the
     lockfile and fails on any mismatch.
  5. Run /npm-trust-check for a deeper audit of binary collision and CI script risks.
```

---

## Step 7 — Optional CycloneDX export

If the user asks to save the SBOM as a file, or mentions CycloneDX, SPDX, or
"machine-readable format":

Generate a CycloneDX 1.4 JSON file with these fields per component:
- `type`: "library"
- `name`, `version`
- `purl`: `pkg:npm/<name>@<version>` (use `pkg:npm/%40org%2Fname@version` for scoped)
- `publisher`: from `_npmUser.name`
- `hashes`: `[{ "alg": "SHA-512", "content": "<hash without sha512- prefix>" }]`
- `externalReferences`: repository URL if available

Save as `sbom.cdx.json` in the project root. Report the file path.

Explain to the user: "This file can be submitted to security scanning tools,
used in compliance audits, or stored as a point-in-time snapshot of your
dependency supply chain. Run this command again to detect any changes."

---

## What this command does not do

- Does not install any packages
- Does not run `npm audit` (that checks for known CVEs — a different check)
- Does not check for outdated packages (use `npm outdated` for that)
- Does not replace a full SCA (Software Composition Analysis) tool for production
  security programmes — it's a fast first-pass audit designed for developers
  who haven't thought about supply chain risk before

For teams that need continuous SBOM monitoring, consider: Anchore Syft,
CycloneDX CLI (`cyclonedx-npm`), or GitHub's dependency graph + Dependabot.
