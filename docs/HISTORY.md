# History

The design journey, chronologically. This doc exists so anyone joining the project later can reconstruct *why* the design looks the way it does without having to re-derive it from scratch. Each section is a step in the thinking; the conclusions roll forward.

## 1. Genesis: filter + hook as the universal primitives

The original framing was about *agent synergy*: every productive interaction between an AI coding agent and a system has two components — **filtering** (what set of actions is the agent allowed to attempt) and **hooking** (what happens to each allowed action). Most existing agent tooling collapses these together; treating them as separate composable primitives turns out to be the single most leveraged design insight.

A profile selects the filter. A filter constructs the agent's reachable universe of actions. A hook then decides, per action, whether to forward / rewrite / refuse. Two simple primitives, composable, orthogonal.

## 2. The PATH-ownership realization

The first major reframe came from noticing that **we are not a participant in command resolution — we are the resolver**. The agent's view of which commands exist is whatever resolves on its `PATH`. If a single directory containing our shims is at the front of `PATH`, every command the agent invokes hits our shim first. Tools we did not symlink into that directory don't exist from the agent's perspective.

This collapses the comprehensiveness problem. The old worry — "what if the agent finds a tool we didn't wrap?" — assumes the agent and the filter compete on a shared `PATH`. They don't. **Allowlist by presence, not by policy.** Absence is the policy for the long tail; our policy table handles the rest.

The framing crystallized as *"alpha command, top dog"*: we are the first thing the OS sees when the agent invokes a command, not the last.

## 3. The trampoline pattern

How does a single binary serve as `tap-git`, `tap-bash`, `tap-cargo`, etc.? **argv[0] dispatch** — busybox / toybox / volta have done this for decades. One binary on disk, many symlinks pointing to it. The binary inspects how it was invoked (argv[0]) and dispatches to the matching handler.

Install strategy varies by OS:
- **Unix**: symlinks (atomic, cheap, well-supported).
- **Windows**: **copies by default** (symlinks require admin or Developer Mode), with NTFS hardlink optimization when source and dest share a volume. Never `.cmd` shims (slow + argv-quoting hell).

Parse argv[0] with `Path::file_stem()` — handles `.exe` stripping on Windows and bare names on Unix uniformly. Strip the `tap-` prefix → tool name. Find the real binary via injected `TAP_REAL_PATH` env (set by orchestrator from a sanitized `PATH`); fallback walks `$PATH` skipping the directory containing `canonicalize(current_exe())`. Hard abort if the resolved real path equals our canonicalized exe — prevents fork bombs from misconfigured `PATH`.

## 4. IPC for live profile state

The orchestrator needs to be able to flip the active profile mid-session and have the next shim invocation see the new policy without restarting the agent. Env vars are inherited at spawn and frozen, so `TAP_PROFILE=foo` doesn't meet the bar.

Surveyed: file + atomic rename; UDS / Windows named pipe to a `tap-daemon`; HTTP loopback to a local daemon; shared memory / `mmap` (POSIX `shm_open`; Windows `CreateFileMapping`); env-var-pointing-to-file; signal-based; exotic options (abstract UDS, Linux kernel keyring via `keyctl`, `memfd_create` + sealed FD, FUSE-as-config-bus).

**Winner: file + atomic rename**, with `TAP_PROFILE_FILE` env var indirection. Reasons:
- Warm-cache read is ~10–50 µs; three orders of magnitude under the ~30–80 ms Windows `CreateProcess` floor. Latency is invisible in practice.
- No daemon to supervise. No daemon-crash failure mode. No lifecycle bug surface.
- `rename()` is atomic for live readers on ext4 / APFS / NTFS — never a torn read.
- One implementation across all six target platforms.

Daemons would have bought sub-microsecond reads at the cost of two backends (Windows AF_UNIX lacks SCM_RIGHTS / abstract / SEQPACKET, so code can't be shared), supervision logic, and process-death failure modes. Not worth it under the `CreateProcess` perf floor.

The Linux kernel keyring trick (`keyctl_search(@us, "user", "tap.profile", 0)`) was genuinely elegant — sub-µs reads, kernel-GC'd at logout, atomic at the key level — but Linux-only. Kept in the back pocket as a future fast-path if profiling ever justifies it.

## 5. Multi-agent isolation via per-agent state directory

A global `~/.tap/profile` only works for one agent at a time. With multiple agents in parallel (a swarm, or just two terminals), each agent needs its own profile context. The fix is one extra level of indirection: an env var (`TAP_AGENT_DIR`) points to a *per-agent directory* containing the profile pointer + audit log + other state. Each agent gets its own UUID-named directory; profile flips happen by atomic-rename *within* that directory.

```
TAP_AGENT_DIR=/run/tap/agents/<uuid>/
  ├── profile.active        (atomic-renamed pointer)
  ├── compiled/<sha>.fbs    (content-addressed compile artifacts)
  └── audit.log             (per-agent JSONL audit trail)
```

Bonus: garbage collection becomes `rm -rf $TAP_AGENT_DIR` when the agent dies.

## 6. Language choice: plain Rust, not a script-compile dance

Considered: `rust-script` / `cargo -Zscript`, `scriptisto`, C++ with a shebang-compile shim, Zig, Go, Nim / Crystal / V, `tcc -run`.

The user's intuition was that a "script that compiles itself" pattern (`#!/usr/bin/env tap-policy`) would be ergonomic. After research: it isn't, because the people authoring policy are users authoring profiles — *not* writing the binary itself. The binary is our code. Shipping an evaluator's cache-stat-on-every-invocation costs more than just running the prebuilt binary.

Winner: **plain Rust binary**, shipped as a workspace member alongside whatever else. Release profile: `opt-level="z"`, `lto=true`, `codegen-units=1`, `panic="abort"`, `strip=true`. Targets <300 KB, 2–3 ms cold start. Cross-platform via the existing toolchain. Zero new build dependencies.

Zig was the only honest alternative — single-file static binaries, best-in-class cross-compile — and remains a "revisit if a standalone distribution outside the existing wheel ever becomes the goal" option. Not now.

## 7. Prior art research

Surveyed: Claude Code's `PreToolUse`/`PostToolUse` hooks; Cursor's allowlist/denylist (4 bypasses found, deprecating denylists in 1.3); Continue.dev's `--auto`/`--readonly` mode cycle; OpenInterpreter Profiles (Python files — Turing-complete config destroys auditability); pyenv/asdf/mise/volta PATH shims; `sudo intercept` (ptrace/LD_PRELOAD on grandchild execs); Bazel sandbox `secure_path`; OPA / Cedar / Rego policy engines.

Key findings:
- **Claude Code's hooks can't see grandchildren** — `Bash(cmd="make")` spawning `git push` is invisible to `PreToolUse`. PATH-prefix interception of `bash` / `make` closes the gap by ensuring the grandchild's `git` resolves to a `tap` shim.
- **Cursor's denylist failures** — every argv-string denylist gets defeated by shell quoting, `$()`, aliases. Allowlist-by-presence is the only defensible posture.
- **volta's argv[0] hardlink model** — exactly the trampoline pattern we want.
- **sudo's `intercept` mode** is the closest production analog to grandchild execve interception, but uses ptrace/LD_PRELOAD which is OS-specific and adversarial-grade.
- **Cedar's `permit`/`forbid`/`when` shape** — the right rule grammar, sub-ms eval, embeddable. Steal the shape, skip the engine.

**Identified gap**: no existing tool combines volta-style argv[0] dispatch + Cedar-shaped policy + Claude-style structured refusals + a cooperative-agent (not adversarial) threat model + a workflow-mode profile system. Each ingredient is proven separately; nobody assembled them.

## 8. Vocabulary disambiguation

"Profile" was overloaded early — sometimes meaning a high-level workflow archetype ("brainstorm", "ship-it"), sometimes meaning a granular constraint module ("no-network"), sometimes the active runtime selection.

Settled vocabulary:
- **Rule** — one atomic `permit`/`revoke` clause with optional predicates.
- **Template** — a reusable bundle of rules expressing one coherent stance (`no-network`, `safe-git`).
- **Profile** — a named composition of templates plus inline rules; what the agent runs under.
- **Capability** — a taxonomy axis (`git`, `shell`, `network`, `filesystem`, `release`, …) used for grouping rules and for namespacing rule IDs.
- **Workflow** — a *human-readable label* for what the agent is doing (`brainstorm`, `ship-it`). Often pairs 1:1 with a profile but is a *user-facing label*, not the policy itself.

Keeping Workflow ≠ Profile as separate words lets us have multiple profiles for one workflow (e.g., `brainstorm-careful` vs `brainstorm-wild`) without polluting the human-facing vocabulary.

## 9. Five composition strategies — first bake-off

Spawned five sub-agents, each assigned a fundamentally different composition philosophy, to design the profile format:

| Strategy | Core idea |
|---|---|
| **CSS cascade** | Order + specificity + `!important`-style layers; last-match-by-specificity wins |
| **OOP inheritance** | Profile `extends` base + `includes` mixins; rules are named; subclass overrides by name; Ruby flat-include MRO |
| **Kustomize overlays** | Base + ordered overlays via Strategic Merge Patch; merge key on rule `id`; `$patch: delete` to remove |
| **Functional composition** | Templates are pure functions; profile is `compose([t1(p1), t2(), ...])`; Starlark embedded |
| **Pure-additive + revoke** | Set-union semantics; `revoke "rule-id"` is itself a rule; no precedence, no order dependence |

Convergence on low-level decisions (all five independently): mandatory eager `compile` step → content-addressed mmap-friendly artifact (3 picked FlatBuffers, 1 MessagePack, 1 CBOR); embedded provenance for `tap why`; KDL won as the surface syntax (3 of 5).

## 10. Pure-additive wins on the cardinal sin

The deciding criterion: **in a security tool the cardinal sin is silent precedence shifts**. Cascade (CSS-style), MRO (OOP-style), and overlay-stack (Kustomize-style) all use implicit ordering — and the failure mode is a permit accidentally shadows a forbid and the agent gains a capability nobody granted. That's exactly the class of bug Cursor's denylist team hit 4 times before deprecating denylists in 1.3.

**Pure-additive + revoke is the only strategy where the resolved policy is a literal set** — no order, no precedence, no specificity. The reviewer reads the union, sees every rule, knows that's the full universe of permissions.

Adopted with two borrows: lockfile + content-addressed artifact pattern (from the functional / Starlark proposal), and tombstone-style audit ladders ("disabled here, originally permitted there") for revocations (from the OOP proposal).

## 11. Posture inversion: god-mode by default

The next reframe was about *which way the default points*. Initial framing was allowlist (default-deny, permits as primary rule). The product positioning is **"YOLO by default — your AI agent has god-mode, unless you say otherwise"**, which inverts the posture: default-permit (god-mode), forbids as primary rule, revoke loosens.

The composition mechanism survived the inversion intact — set-union of forbids works the same as set-union of permits. Only the default-state semantics and the verb names changed.

## 12. Bidirectional posture stress test

The natural next question: *can we support both postures so users pick per profile?* Spawned five lenses (security, composition, UX, implementation, migration) to find issues. **50 raw issues, deduped to 29 themes.**

The cross-cutting observation that ended the question: **posture is a second composition lattice layered on top of the rule lattice, and the pure-additive set-union invariant only works for one lattice**.

Concretely:
- `revoke` becomes a polymorphic verb whose security gradient inverts with target posture (same line, opposite effect).
- Per-capability posture in templates is a privilege-escalation primitive (a community template flipping `posture(shell) = god-mode` invalidates every existing forbid).
- Set-union of postures per capability is *not* a commutative join — two templates disagreeing have no resolution.
- `forbid`/`permit` keywords carry opposite security weight under different postures (a `forbid` in god-mode is a guardrail; in allowlist it's a tautological no-op).

The most aggressive critique (C10 in the composition lens): *"Bidirectionality may be gratuitous — pick one posture and express the other as inversion at authorship time."*

## 13. Five resolution strategies — second bake-off

Spawned five sub-agents again, each given a distinct *resolution* for the bidirectional-posture conflict:

| Strategy | Philosophy |
|---|---|
| **A — Drop posture** | Allowlist-only; `permit-all <cap>` is the inversion verb; empty profile auto-synthesizes `permit-all *` (god-mode) |
| **B — Profile-scoped posture** | Both postures coexist as alternatives, declared once at profile root, never per-cap, never settable by templates |
| **C — Capability-defined defaults** | Binary owns the per-capability default (`git=permit`, `release=deny`); user can never flip (except via narrow root-only override) |
| **D — Sandbox levels** | Discrete lattice (`locked < restricted < loose < god`) per capability; pre-built rulesets in the binary; composition by monotone meet |
| **E — Two-system split** | One binary, two completely separate modes (`TAP_MODE=god\|allowlist`); chosen at agent-spawn, immutable for the process tree |

Each was asked to walk through how it would resolve the 7 Critical and 11 High issues, with concrete KDL examples.

## 14. Strategy A wins, with two borrows

**A resolves all 7 Critical issues by elimination, not mitigation.** It is the only strategy that doesn't carry an asterisk on any Critical theme. It also has the lowest implementation cost (~50% reduction vs. bidirectional), the smallest cognitive surface (3 verbs, no posture vocabulary), and the cleanest product story (the marketing line *"empty profile = god-mode"* is literally true).

**Borrows:**
- **From C** — risk-asymmetric defaults (some capabilities want default-deny: `release`, `package-publish`, `credentials`, `destructive-fs`) delivered as an *opt-in template* `@tap/safe-defaults` rather than baked into the binary. Editorial wisdom available without making the maintainer carry forever-API.
- **From D** — per-capability god-mode delivered as a *shipped template library* (`@tap/god-mode-bash`, `@tap/god-mode-git`, etc.) so users can compose "wide-open bash + locked-down network" via includes without writing `permit-all` boilerplate.

## 15. zccache workflow validation — the third bake-off

Tested A, D, and C against a *real* 4-phase workflow: refactoring zccache to async-everywhere (brainstorm → fixup → cross-compile → commit & PR). All three returned the same one-word verdict — **"awkward"** — but for completely different reasons:

- **A**: graceful at additive phases (2/3/4), **awkward at restrictive Phase 1**. The brainstorm phase wants "thinking only — take everything away," which under god-mode default forces either 9 hand-written revokes or hard dependency on shipped negative templates that A's grammar can't natively express. The structural critique: *"the default is unsafe, and 'safe' is a subtractive exercise over a moving target."*
- **D**: graceful at coarse phases (1/2/3), **awkward at fine-grained Phase 4**. The dev needs `git=commit-push-feature-branch-only`, which doesn't exist in the four-level lattice. Falls back to `loose` + revoke + 3 also-permits referencing upstream-owned rule-IDs. Plus the "5 lines per profile" claim was false (9–14 lines in practice). Plus `bash test` vs `cargo test` taxonomy mismatch — project-specific tools aren't in the matrix.
- **C**: awkward at Phase 1 (5 root-level `capability { mode "deny" }` overrides) **plus two architectural breaks**: (1) `cargo` spans 4 capabilities (build, tests, publish, install) so the capability-≠-binary problem is fatal at the shim layer; (2) verb-constraint rule has an unresolved compiler ambiguity under profile-level mode flips.

A's awkwardness is fixable with a template library. D's and C's are structural.

## 16. Final design additions from zccache

The workflow exposed three new design requirements that weren't surfaced in earlier rounds:

1. **Negative templates are mandatory, not optional.** Ship `@tap/no-toolchain`, `@tap/no-git`, `@tap/no-network`, `@tap/no-mutation`, `@tap/read-only` alongside the positive templates. Without them, restrictive phases are subtractive-by-hand and silently broken when new shims are added to the farm later.
2. **`tap profile diff` is a first-class tool.** Shows "added X, revoked Y, swapped Z" between profiles. Table stakes for team review and incident forensics — borrowed unchanged from D's strongest moment.
3. **Stuck-on-denial detection in the daemon.** Track denial-rate per agent per minute and surface a toast to the dev when an agent grinds on a missing capability. Without it, "dev forgets to flip phase" becomes "silent agent deadlock."

These add to the resolved design, they don't change it.

## 17. Naming: clud-hook → tap

The original name `clud-hook` was a working title — descriptive but coupled to a parent project. Once the design solidified enough to warrant its own repo (so we can build with fast unit tests on grammar compilation, deferring integration), the question became: what should it actually be called?

Brainstormed 10 alternatives, narrowed to 3-letter and 4-letter shortlists:
- **3-letter**: `mux`, `tap`, `vet`, `cap`, `hub`, `cog`, `lid`, `pin`, `arc`, `dog`
- **4-letter**: `gait`, `shim`, `ward`, `moat`, `gate`, `bind`, `rail`, `wrap`, `clip`, `norm`

Tested against the actual call patterns: `<name> git push`, `<name> write`, `<name>-bash test`, etc.

**`tap` wins.** Three letters; reads idiomatically as both noun ("a tap") and verb ("to tap into"); `tap git push` parses as *"tap into the git push"* which is the wiretap metaphor matching the architecture exactly. `tap-write` is one of the rare compound shim forms that reads as a single concept. Visually clean in shell prompts and logs.

Repo created at https://github.com/zackees/tap.

---

## Where we are now

The resolved architecture is documented in `ARCHITECTURE.md`. The alternatives we considered (and the specific reasons each was rejected) are in `REJECTED.md`. The conceptual insights that shaped the design are in `CONCEPTS.md`. Implementation has not yet started — the next move is to scaffold the Rust workspace and the grammar-compilation unit tests that motivated hoisting this out of the parent project in the first place.
