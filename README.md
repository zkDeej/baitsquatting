# Baitsquatting: How AI Agents Became the Distribution Layer for a New npm Attack Class

*Dhananjayan Maniraj — published 2026-05-08*  
*Disclosure reference: BSQT-2026-001*

> **Live tracker:** [zkDeej.github.io/baitsquatting](https://zkDeej.github.io/baitsquatting) · **Visual summary:** [diagrams & attack flows](https://zkDeej.github.io/baitsquatting/diagrams.html) · **FAQ:** [baitsquatting FAQ](https://zkDeej.github.io/baitsquatting/FAQ.html) · **Developer tools:** [npm-trust-check](https://gist.github.com/zkDeej/ebc290ca7fc5240e6fd4032ede24f1eb) · [sbom-generate](https://gist.github.com/zkDeej/f4dc6bc45011eddbcc3e268032ad7edf)

> *This is a living document. Vendor response tracking is ongoing — case status and namespace claims are updated as they resolve.*

> **Disclaimer:** This is independent security research, not an official advisory from npm, Anthropic, or any other vendor named herein. Tool names and package references are used for factual identification purposes only. Developer tools in this repo are provided under the [MIT License](https://opensource.org/licenses/MIT); research documentation is provided under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

---

I have 20+ years in IT and software development, the last fifteen of them at Barclays in London — inside the safe confines of infrastructure heavily-guarded by a security culture that walks the tight line on bureaucracy, and rightly so. I left that and am currently with my new baby on a well-earned break. I spent the past year building on cloud platforms with AI vibe coding tools — not as an auditor, not with a research budget, not inside a security team. On diaper duty, just building things in the vibe.

When a stranger's name greeted me in my terminal, executing in a shell with live credentials, briefly I froze. Then jumped out of my chair and looked closer. In that moment I became very aware: I was no longer inside the perimeter. I recognised what it was and started digging in.

What followed was three weeks of the most consequential technical work I've ever done outside an institution — and most of it would not have been possible without the same AI tools that created the problem. Was it the AI's fault there's arguably a new vulnerability class? How did I end up in the Lhci 3200 Bait Club?  That's what this writeup is about.

---

## The incident

I was using Claude Code in agentic mode, wiring up CI for a project. Claude proposed:

```bash
npx lhci autorun
```

I saw the permission prompt. I approved it — I trusted the AI's judgment on tooling. npm downloaded `lhci@4.1.1` and ran it. The terminal printed:

```
Hello, this is AnupamAS01!
(timeout 30s)
```

That's not Lighthouse CI. That's a stranger's package, running with full access to my shell environment. My `.env.local` was loaded — Supabase service role key, Sentry DSN, PostHog key. I rotated everything.

AnupamAS01's package announced itself rather than exfiltrating silently — that's researcher behaviour, not attacker behaviour. But I had no way to know that at the moment of execution. Anyone who approved that same AI-proposed command over the past two years had identical exposure.

---

## The gap

The real Lighthouse CI tool is `@lhci/cli`, maintained by GoogleChrome, with ~2 million monthly downloads. After install, the binary is called `lhci`. The official docs say `lhci autorun`. That's what every tutorial, README, and CI config example teaches.

The bare npm package `lhci` is not owned by GoogleChrome. It recorded 12,947 downloads in April 2026 (~3,200/week, npm download API) — that's 3,200 developers a week running a package that isn't what they think it is. I was one of them. The package is held by AnupamAS01 — the researcher who originally published it as a PoC (`0e.vcmaa` is his alternate account; confirmed by email; he published v4.1.2 on 2026-05-06 updating the description to identify himself). The account, if ever compromised or transferred, inherits those 3,200 weekly executions with full shell credential access.

| Context | Command | How `lhci` resolves | Result |
|---|---|---|---|
| After `npm install -g @lhci/cli` | `lhci autorun` | Binary on PATH from `@lhci/cli` | ✅ GoogleChrome's tool |
| Fresh shell / CI / agentic (`npx`) | `npx lhci autorun` | Package lookup — npm registry | ❌ AnupamAS01's package |

The permission prompt showed the exact command. It didn't surface which resolution path executes. That distinction is second nature to an experienced npm developer. In an agentic context where the AI proposed the command and the user's role is approve/deny, it's invisible.

---

## What this is — and what it isn't

You may have heard of **slopsquatting** — AI models hallucinate package names that don't exist, attackers register them. This is categorically different.

| Attack class | How the wrong name enters | AI role | Existing mitigation |
|---|---|---|---|
| Typosquatting | Human typing error | None | Similarity checks |
| Slopsquatting | AI hallucinates a non-existent name | Primary cause | Hallucination reduction |
| **Baitsquatting** | AI recommends the name the tool's own docs teach | Amplifier at scale | **None** |

Claude Code didn't hallucinate anything. It recommended `npx lhci` because Lighthouse CI's own documentation teaches `lhci` as the canonical invocation. The model followed the docs exactly. The attacker doesn't need to guess what the AI will say. They register the obvious name and let the training data do the distribution.

*A note on naming: the "bait" here is not set by the attacker — it's set by the legitimate maintainer's own naming convention. The attacker is passive; the docs and the model do the work. The name reflects the trap that's already in place, not the attacker's agency. If a cleaner name emerges through community use, I'll adopt it. For now this is the most precise shorthand I have for a mechanism that is genuinely distinct from the two classes it neighbours.*

---

## The wider surface

After surveying more packages, the pattern is specific but recurring. The gap requires four conditions:

1. Tool adopted scoped packaging (`@org/cli`)
2. CLI binary kept a short, bare name
3. Official docs teach that bare name as the primary invocation
4. The bare npm namespace is unclaimed or held by an unrelated party

**Flow B — execution-time (highest severity):** `npx <bare-name>` downloads and executes directly. Full shell environment exposed. No audit trail. These are the confirmed baitsquatting cases — the tool's own documentation teaches the bare invocation, so the model follows the docs exactly. No model error required; the attack surface is fully enumerable from registry data and documentation alone.

| Real package | Bare alias | Namespace status | Scenario |
|---|---|---|---|
| `@lhci/cli` | `lhci` | AnupamAS01 (~3,200/wk — 12,947 in April 2026) | **Live incident** — CI env, service role key, full secrets |
| `@changesets/cli` | `changeset` | eugeneware (last published ~5 years ago, unrelated package) | CI pipelines — npm publish tokens, GitHub tokens |
| `dependency-cruiser` | `depcruise` | Claimed as defensive placeholder (July 2025)* | Dev environments |
| `@biomejs/biome` | `biome` | 1egoman (last published 2016, unrelated package) — Biome team notified; acknowledged as known issue, actively seeking npm transfer | Millions of weekly downloads on scoped package |
| `@moonrepo/cli` | `moon` | kabir (last published 2020, unrelated package) — moonrepo notified 2026-04-29 | Monorepo CI pipelines |

Additional confirmed cases with active vendor disclosure windows are not listed here — they will be added once those windows close.

\* `depcruise` was claimed as a defensive placeholder in July 2025 by Aikido Security (account: `debugducky`), discovered via a web search during this investigation.

"Abandoned" here means the bare namespace is held by a package unrelated to the tool in question, with no recent publish activity — verified via `npm view <package>` publish dates and maintainer identity. The risk is that any actor could approach the current holder, or that the holder publishes a new version. It is not the same as unclaimed, but it is not controlled by the tool's maintainer either.

**Adjacent cases — install-time, model error unconfirmed:** `npm install <bare-name>` adds to dependencies. Exposure limited to install scripts. These cases differ from Flow B in a taxonomically important way: the tools' documentation teaches the fully scoped name throughout — no bare invocation is taught. For an attacker to exploit these, a model would need to drop the scope prefix (e.g. generate `npm install stripe-js` instead of `npm install @stripe/stripe-js`), which would be a model output error rather than doc-following behaviour. No live AI output generating these bare install names has been observed. This places them closer to slopsquatting than baitsquatting — consistent with the finding that all five names (`aws-sdk-client-s3`, `stripe-js`, `prisma-client`, `supabase-js`, `clerk-nextjs`) independently appear in the USENIX Security 2025 hallucinated-package dataset. The namespace gaps are real and the blast radius is high — all teams were notified under BSQT-2026-001.

All affected teams have been contacted under coordinated disclosure BSQT-2026-001.

---

---

## AI agent behaviour — cross-tool verification

The behaviour is not specific to one model or one platform.

| Platform | Project scaffolding | Fresh-shell npx | Notes |
|---|---|---|---|
| Claude Code (2026-04-18) | `npx lhci` — bare, executed live | N/A — live incident | Pattern-matched from training data |
| Microsoft Copilot (2026-05-01) | `npx lhci autorun` — bare, prompt: "I'm setting up Lighthouse CI for a Node.js project. What's the npx command to run it?" — 6 bare command variants, zero mention of `@lhci/cli` | Not tested | Self-acknowledged: *"the natural solution is `npx lhci autorun`"* |
| ChatGPT (2026-05-01) | `npx lhci autorun` — bare, same direct prompt as Copilot | Not tested | Same output as Copilot — model-independent |
| Replit Agent (2026-04-22) | `@lhci/cli` installed, bare `lhci` in all scripts | Not tested | Project context masked the gap |
| Emergent / Claude Opus (2026-04-29) | `@lhci/cli` installed, bare `lhci` in scripts | `npx -y @lhci/cli@0.15.1` ✅ | Scoped in fresh-shell |
| Emergent / ChatGPT (2026-04-29) | `@lhci/cli` installed, bare `lhci` in scripts | `npx -y @lhci/cli@latest` ✅ | Model-independent on this platform |
| Gemini 3 Flash / AI Studio (2026-04-29) | `npm install -g @lhci/cli@0.13.x` + bare `lhci` ⚠️ | `npx @lhci/cli autorun` ✅ | Active reasoning visible in trace |

The persistent gap: models have improved conversational recommendations (Claude has since), but **generated file output** — CI yaml, npm scripts, shell scripts written into project files — still carries bare binary invocations across every platform tested. That's the surface that matters in practice.

---

## Industry corroboration

**Prof. Murtuza Jadliwala and Joe Spracklen** — two of the six authors of "We Have a Package for You!" (Spracklen et al., USENIX Security 2025, **Distinguished Paper Award**) on hallucination-driven package attacks — were the first external researchers contacted, on 2026-04-21. Their views on taxonomy differ, and that is worth putting on record. Jadliwala: *"Baitsquatting, as you refer it, is indeed very unique from hallucination-dependent slopsquatting which we studied in our USENIX paper."* Spracklen: *"I think baitsquatting is more of a subset of slopsquatting/package hallucination than a completely distinct class of attack."* My view: the mechanism is distinct because no hallucination occurs — the model recommends a real, predictable name that the tool's own documentation teaches. Whether that warrants a new class or a subclass sits in a grey area — Spracklen's concession that the discovery method is the key differentiator is the strongest version of the argument for distinguishing them, and the community will settle the taxonomy. What's black and white is the attack surface. Cross-checking their hallucinated package database against our bare-alias inventory revealed five names appearing in both: `stripe-js`, `aws-sdk-client-s3`, `supabase-js`, `clerk-nextjs`, `aws-sns`. The names models most frequently hallucinate are the same names maintainers failed to register — the confusion is real, systematic, and bidirectional.

**Anupam Singh** — the researcher behind the original `lhci` placeholder — was contacted on 2026-04-27. After our disclosure conversation he ran a systematic scan of ~170 GitHub organisations and 7,000+ repositories, identifying the same bare-alias collision pattern via README enumeration. Five new confirmed collisions: `dynwinrt-codegen` and `snapfeed-server` (Microsoft), `azure-functions-templates-mcp-server` (Azure), `clerk-cli` (Clerk), `create-software-factory` (Continue). All unclaimed as of 2026-04-27.

An independent security firm had already identified the same vulnerable namespaces through a full npm registry scan and claimed `depcruise` as a defensive placeholder in July 2025 — discovered during a web search in the course of this investigation, and confirmed via npm registry metadata. They did not identify other overlapping namespaces from this inventory, which is consistent with a different scanning methodology (registry-wide enumeration rather than AI-agent probing or README analysis).

Three independent research efforts — AI-agent probing, systematic README enumeration, and full registry scanning — have converged on the same collision class.

Several tool maintainers acted on notification during the disclosure window. NestJS responded within 24 hours and initiated a namespace claim. Clerk claimed `clerk-nextjs` silently and published a placeholder redirecting to `@clerk/nextjs` — no public issue required, no fanfare, just a closed attack surface. These responses set the bar for what coordinated disclosure is supposed to look like.

---

## Beyond npm

The same risk exists wherever AI tools recommend short bare commands against registries that don't enforce scoped naming.

| Ecosystem | AI execution command | Risk level | Notes |
|---|---|---|---|
| npm / npx | `npx <name>` | **High** | Confirmed live incident |
| Python — pip / uvx | `uvx <name>` · `pip install <name>` | **High** | `huggingface-cli` — a slopsquatting experiment by Bar Lanyado (Lasso Security) — received 30,000+ downloads on PyPI within 3 months after AI tools repeatedly hallucinated the name, demonstrating AI-amplified distribution applies cross-ecosystem |
| Docker Hub | `docker pull <name>` | **High** | Structurally identical namespace dynamics |
| Homebrew | `brew install <name>` | **Medium** | Maintainer-controlled naming limits surface |
| Cargo | `cargo install <name>` | **Medium** | Collisions exist; none weaponised |

---

## What Anthropic said

HackerOne #3682326 — filed as Critical (CVSS 9.0; vector: AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:H, calculated by HackerOne's Report Assistant). Closed as Informative: the permission prompt showed the exact command; the control worked as designed. Their framing, on record:

> *"A model proposing an unscoped package name where a scoped one exists is a model-output quality issue, and the underlying typosquatted npm package is third-party dependency hijacking — neither is a vulnerability in Claude Code's security controls."*

The UI:Required in that vector reflects the permission prompt — the user approved the command. The mitigating effect of that approval step is precisely what this research argues is insufficient for non-developer users delegating full trust to an AI agent. A developer seeing `npx lhci --version` in a prompt has the context to evaluate it; a non-developer in agentic mode does not.

The security team considered the permission prompt the relevant control — and by that framing, it functioned correctly. They also noted that this is a topic of active interest to Anthropic's model safety team, drawing a clear line between the security boundary (working as designed) and the model behaviour question (a separate concern, tracked separately). That distinction is precise and, if acted on, points at the right fix.

---

## What Replit said

Replit security responded within 6 hours — acknowledged baitsquatting, invited submission to their private HackerOne programme. Before filing, I asked the Replit agent to self-analyse the vulnerability. It independently diagnosed the attack: `npx <bare-name>` resolves against the npm registry, not the installed binary — and proposed a tool-layer registry check before suggesting bare package commands. That is the exact fix we had asked Anthropic for, produced unprompted by Replit's own model.

The finding was not accepted on process grounds: Replit's private HackerOne programme requires a minimum Signal score, and a first-time reporter has a score of zero. The finding was acknowledged in writing as thorough and the disclosure as professional. The model identified the problem and proposed the fix; the process couldn't receive it. There's an irony in a Signal score filtering the signal.

---

## What needs to change

**Tool maintainers** — if your CLI binary has a bare name, claim the npm namespace. Publish a placeholder. It takes 10 minutes and you've already trained the ecosystem on that name. NestJS and Remix/React Router both acted within 24 hours of notification — React Router published a placeholder and committed to updating their docs to guide agents to `@react-router`. That's the bar.

**AI platform developers** — pre-execution registry verification for bare `npx` invocations. Before an agent proposes `npx X`, resolve X in the registry and check whether the publisher is consistent with the tool's known maintainer, or whether a scoped equivalent exists with substantially higher downloads. This is enforceable at the tool-use layer — not a training problem, not a prompt engineering problem.

**AI agent definition authors** — always use the fully scoped package name in `runCommands` instructions. Use `npx @scope/package` not the bare binary alias. An agent with shell execution access and an unclaimed bare namespace in its instructions is an unregistered attack surface.

**npm** — proactive namespace gap detection is feasible: if a scoped package exposes a binary name with no corresponding bare namespace claim, flag it to the maintainer automatically.

**Developers** — four practices meaningfully reduce exposure regardless of IDE, tooling, or whether you use AI assistants at all. Commit your lockfile and use `npm ci` in CI — every install is then hash-verified against a pinned state, and any deviation fails the build. Pin exact versions (`--save-exact`) so dependency changes are always explicit and reviewable. Prefer the fully scoped install form (`npx @org/package@version`) in scripts, CI, and project documentation — it removes the bare-name ambiguity entirely. Generate a Software Bill of Materials (SBOM) as a CI step using `cyclonedx-npm` or `syft` — a publisher change between builds becomes immediately auditable rather than invisible. None of these require waiting for npm or AI platforms to act.

**Claude / AI coding agent users (vibe coders)** — two slash commands give you direct protection without needing to understand the underlying mechanics. [/npm-trust-check](https://gist.github.com/zkDeej/ebc290ca7fc5240e6fd4032ede24f1eb) audits your project for binary collisions, ephemeral `npx` calls with no lockfile protection, and AI-hallucinated package names drawn from the Spracklen et al slopsquatting dataset. [/sbom-generate](https://gist.github.com/zkDeej/f4dc6bc45011eddbcc3e268032ad7edf) produces a publisher integrity report for every dependency — who published it, when, and whether the hash matches the registry — with optional CycloneDX export. Both are read-only: they never install or execute anything. Improvements welcome.

---

## On AI as both the threat vector and the research tool

The attack is live. The namespaces are open. The AI agents are running. The fix at every layer is small and well-defined. What this three-week investigation exposed to me is not just a vulnerability class — it exposed something about how the ecosystem responds to one.

There is no vibe coder village police. Responsible disclosure is a polite fiction held together by goodwill, process, and the patience of whoever found the problem. Bug bounty programmes gate access on prior reputation. Security teams close reports as out-of-scope and route you to a different inbox. Vendor portals acknowledge receipt and go quiet. Researchers argue taxonomy. The village has a lot of committees and not many constables.

And yet — something worked. NestJS claimed a namespace within 24 hours. Clerk silently published a placeholder with no fanfare and no public issue required. Remix/React Router responded the same day with a fix and a doc update commitment. An independent researcher ran a 7,000-repo scan overnight. A Distinguished Paper Award team cross-referenced their dataset against this inventory within days of being contacted.

The more striking observation is this: the models moved faster than the processes. When Replit Agent was asked to diagnose the vulnerability it had just demonstrated, it described the attack mechanism precisely and proposed the tool-layer fix — unprompted, in the same session. When Gemini reasoned through a bare `npx` command in real time, it caught itself and pivoted to the scoped form. The AI systems are not waiting for a CVE to be filed or a patch to be reviewed. They are updating.

The bureaucracy is slow because it was built for a different age. The models are fast because they were built for this one. That asymmetry is the most pragmatic reason for optimism: the defenders who will matter most in this environment are not the ones with the largest security team — they are the ones who learn to collaborate and use the same tools the problem arrived with.

The surface is larger than this writeup covers. The methodology is reproducible. If you're a maintainer, check your binary name against the npm registry right now — it takes thirty seconds. If you're building AI agents, use scoped package names in every instruction. If you're an AI platform, a pre-execution registry check is one pull request away from closing the most exploitable part of this surface entirely.

The village doesn't need more committees. It needs more people who noticed something odd and didn't close the tab. And that's more vibe coders and a little curiosity. Skip the Lhci 3200 Bait Club.

---

## Scope and limitations

Initial discovery, taxonomy development, cross-tool verification, 45-package survey, coordinated disclosure to 18+ vendors and researchers: single researcher, unfunded, ~three weeks alongside regular work. The 45-package survey covers JavaScript/npm via AI-agent probing. Packages were selected based on scoped-to-bare naming pattern and credential exposure risk at execution time — not random sampling. Cross-ecosystem observations are directional, not exhaustive. Since disclosure, scope has expanded via Anupam's systematic scan, an independent registry-wide scan, and UTSA's academic cross-referencing. The full surface across npm, PyPI, Docker Hub, and Cargo warrants structured follow-up.

---

**Credits**

[Anupam Singh](https://github.com/AnupamAS01) — Phase 2 co-researcher. Systematic scan of ~170 GitHub organisations and 7,000+ repositories identifying new bare-alias collisions via README enumeration, and primary evidence collection for the Microsoft Copilot cross-tool results.

[Prof. Murtuza Jadliwala](https://jadliwala.github.io/) and [Joe Spracklen](https://github.com/jspracklen) — academic validation and dataset cross-referencing. Two of the six authors of "We Have a Package for You!" (Spracklen et al., USENIX Security 2025, Distinguished Paper Award).
