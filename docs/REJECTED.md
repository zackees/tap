# Rejected designs

The alternatives we explored and the specific reason each was rejected. Documented so we don't have to re-derive the conclusions later, and so anyone who proposes one of these again can be pointed at the exact failure mode without re-running the debate.

Entries are grouped by what they replaced. Each entry includes: what the alternative was, why it was tempting, the specific reason it lost.

---

## I. Posture and composition

### Bidirectional posture (god-mode AND allowlist as user-selectable modes)

**The proposal**: support both default-permit (god-mode) and default-deny (allowlist) as profile-level postures, with the user choosing per profile. Possibly per-capability within one profile.

**Why it was tempting**: it preserves the full product story — YOLO by default *and* strict allowlist for sensitive workflows — without forcing the user to commit to one philosophy.

**Why rejected**: a 5-lens adversarial review (security, composition, UX, implementation, migration) surfaced 50 raw issues that deduped to 29 themes, with 7 Critical. The mathematical heart of the failure: **posture is a second composition lattice layered on top of the rule lattice, and the pure-additive set-union invariant only works for one lattice.** Specific consequences:

- `revoke` becomes a polymorphic verb whose security gradient inverts with target posture — same line, opposite effect, undetectable in code review.
- Posture-in-templates is a privilege-escalation primitive (a community template flipping `posture(shell) = god-mode` invalidates every existing forbid).
- Set-union of postures per capability is *not* a commutative join; two templates disagreeing have no resolution.
- `forbid`/`permit` keywords carry opposite security weight under different postures (a `forbid` in god-mode is a guardrail; in allowlist it's a tautological no-op).
- Multi-capability binaries (e.g., `cargo` spans build/tests/publish/install) break per-capability-posture cleanly because shim-presence is per-binary, not per-capability.

Strategy A — drop posture entirely — resolves all 7 Critical issues by elimination. Bidirectionality was the source of the problems; removing it costs less than mitigating it.

### Strategy B — profile-scoped posture only

**The proposal**: keep both postures, but declared exactly once at the profile root, never per-capability, never settable by templates. Templates declare `assumes-posture` and the compiler refuses cross-posture includes. `revoke` is split into directional verbs (`allow-through` for god-mode, `also-deny` for allowlist).

**Why it was tempting**: preserves full bidirectional product story with bounded complexity.

**Why rejected**: B's own evaluator added an escape hatch (`relax "bash"` to opt one capability out of allowlist mode inside an allowlist profile) and then warned: *"if `relax` ends up used in >70% of allowlist profiles in the wild, B should retreat to A."* That's a strong self-test signal that the simpler strategy is the right one from day one. Plus B keeps two posture vocabularies and two evaluator paths, adding test-matrix and documentation surface that A doesn't pay.

### Strategy C — capability-defined defaults baked into the binary

**The proposal**: the binary owns a per-capability default table (`git=permit`, `shell=permit`, `release=deny`, `credentials=deny`, etc.). Users can never change defaults except via a narrow root-only `capability "X" { mode "deny" }` override. Verbs are asymmetric per capability (`forbid` on permit-default, `permit` on deny-default).

**Why it was tempting**: encodes real-world risk asymmetry (one-way doors should default-deny; reversible operations should default-permit) without making the user manage posture at all.

**Why rejected (three reasons)**:

1. **Capability ≠ binary.** `cargo` spans build, tests, publish, install — four capabilities for one binary. The shim layer routes by binary; the policy layer routes by capability. The schema author has to make editorial calls (is `cargo check` `build` or `tests`?) and every project disagrees.
2. **Unresolved verb-constraint ambiguity** under profile-level mode flips. If `capability "git" { mode "deny" }` is set at the profile root, does the compiler accept `permit "git-read" {...}` because the *effective* default is now deny, or reject it because the *schema* default is still permit? Strict-schema means you can't carve back in after flipping; dynamic-effective kills the asymmetric-verb "natural reading" property. C never reconciles this.
3. **Editorial burden becomes public API forever**. Changing `network` from permit-default to deny-default in a future version is a breaking change for every downstream profile. The maintainer is now in the policy-judgment business for the life of the project.

The editorial wisdom C wanted to encode (`release`, `package-publish`, `credentials`, `destructive-fs` default-deny) is preserved in the resolved design as an *opt-in template* `@tap/safe-defaults` — same wisdom, no public-API commitment.

### Strategy D — sandbox levels with monotone-join composition

**The proposal**: per-capability discrete levels (`locked < restricted < loose < god`) with pre-built rulesets shipped in the binary (~200 curated rules across ~8 capabilities). Composition by lattice meet (strictest wins). No user-defined levels, ever. `also-permit "<rule>"` for additive within-level diffs.

**Why it was tempting**: most economical UX in the entire bake-off; profile transitions are diffable as knob movements; "we run `git=restricted, network=restricted`" becomes shared team vocabulary.

**Why rejected (three reasons)**:

1. **The "5 lines per profile" claim doesn't survive contact with a real workflow.** Real zccache profiles are 9–14 lines because the matrix has 8 capabilities and unmentioned defaults to `god` (unacceptable for any non-trivial profile).
2. **Phase 4 (commit & PR) has no matching level.** The dev needs `git=commit-push-feature-branch-only`; the four-level lattice has nothing close. Fallback is `level "git" "loose"` + `revoke "git.push.to-protected-branches"` + 3 `also-permit`s — referencing upstream-owned rule IDs that can rename in `tap v0.7` and silently re-block the push.
3. **Project-specific tool mismatch.** The project's CI-faithful runner is `bash test`, not `cargo test`. `tests=loose` permits `cargo test` only. Forget the `also-permit` → agent runs `cargo test` → misses Python integration tests → declares green → ships broken code. The level *name* sounds permissive enough that the dev won't think to check.

D's curated-rule-matrix is also a forever-API commitment (~200 rules × stable IDs × cross-platform behavior) — same editorial burden as C, applied to rule content instead of capability defaults.

### Strategy E — two completely separate systems sharing one binary

**The proposal**: one binary, two modes (`TAP_MODE=god-mode` or `TAP_MODE=allowlist`), chosen at agent-spawn time and immutable for the process tree's lifetime. Source files carry mode in their extension (`.godmode.kdl` / `.allowlist.kdl`). Separate evaluators, separate template ecosystems.

**Why it was tempting**: provably no posture confusion by construction (mode is process-lifetime constant); 7/7 Critical resolved.

**Why rejected (two reasons)**:

1. **No live mode switch — agent must restart to change modes.** The product workflow is "now it's time to brainstorm / now it's time to ship-it"; restarting the agent between phases loses transcript context, in-flight tool state, daemon-managed PIDs. The "switch on a dime" requirement is foundational; an architecture that fails it is disqualified regardless of how clean the failure surface is.
2. **Template ecosystem fragments.** Every shareable policy needs `.godmode.kdl` and `.allowlist.kdl` siblings, and as E's own evaluator demonstrated, they aren't mechanical translations — the *shape* differs (`forbid * except X` vs `permit X, permit Y, permit Z`). Community ecosystem grows two parallel trees forever.

### Per-capability posture (as a feature within any strategy)

**The proposal**: posture set per capability inside a profile, e.g., `posture(git) = allowlist; posture(shell) = god-mode`.

**Why it was tempting**: matches real user intent — "lock down git, leave shell wide open" — directly.

**Why rejected**: per-capability posture is a *privilege escalation primitive* in third-party templates. A community template containing `posture(shell) = god-mode` flips the shell capability open even though the operator only wanted git conveniences. Set-union of forbids is monotone-safe; set-union of *postures* is not, because flipping posture invalidates every existing rule's meaning. The same per-capability differentiation users want is delivered safely in the resolved design via per-capability **includes** (`include "@tap/god-mode-bash"`) — user-authored composition, not template-set metadata.

---

## II. Composition strategies (first bake-off)

These five strategies were explored as alternatives to pure-additive + revoke. All used KDL (or close cousin) and a content-addressed compile artifact; they differed in *how rules compose*.

### CSS cascade

**The proposal**: profiles and templates are stacked like CSS rule blocks. Resolution by (layer rank, `!important` flag, specificity, source order); last wins.

**Why rejected**: **implicit order is the cardinal sin in security tools**. Reorder two `import` lines and behavior shifts silently. Accidentally-shadowed `forbid` = silent privilege escalation. CSS metaphor is great for *web* devs, hostile for *security* reviewers who need to point at one line and say "this is why it was denied." Also: specificity creep ends in everything-is-`!important` arms races, which `@layer` mitigates but doesn't eliminate.

### OOP inheritance + mixins

**The proposal**: profile `extends` a base + `includes` mixins; rules are named; subclass overrides by name; Ruby flat-include MRO.

**Why rejected**: **fragile base class problem**. A template author renames `push.main.deny` → `block.main.push`. Every downstream profile's override silently becomes a no-op because the inherited rule it was canceling no longer exists. Compiler can warn but warnings get suppressed in CI and the security hole ships. Diamond problem and name collisions across mixins add more silent-failure paths. Tombstones break encapsulation (child profile knows template's internal rule naming).

### Kustomize-style overlay patches (CUE-flavored)

**The proposal**: base + ordered overlays via Strategic Merge Patch; merge key on rule `id`; `$patch: delete` to remove.

**Why rejected**: **merge-key drift silently no-ops deletes**. Template author renames `safe-git/push-any` → `safe-git/push-nonmain` and every downstream overlay's `$patch: delete` becomes invisible. Compiler errors on delete-of-missing help but don't cover `$patch: replace` that just stops matching. Multi-layer debuggability degrades fast (two overlays is fine; five is a forensics exercise). CUE's schema enforcement is the right idea (and pure-additive borrowed the "validate at compile-time" principle), but the patch-based composition model adds verbosity and merge-key fragility.

### Functional composition (Starlark embedded via `starlark-rust`)

**The proposal**: templates are pure functions; `profile brainstorm = compose([safe_git(["main"]), no_network(), swarm_fanout(maxConcurrent=10)])`; composition operators (`merge`, `union`, `override`, `without`) are first-class.

**Why rejected (three reasons)**:

1. **Learning curve for template authors.** The simple case ("a profile with three rules") is fine, but the moment a user writes their first `def template():`, they're learning a programming language to configure a security tool. Most security-tool maintainers won't.
2. **Ships ~3 MB Starlark evaluator inside the security-critical binary.** Even with Starlark's strong sandbox, that's a meaningful expansion of the binary's blast radius. Mitigated by making eval *build-time only* (runtime is pure Rust over MessagePack), but the build-time evaluator is still real code with real bugs.
3. **Two-file model surprises people.** "Why didn't my edit take effect?" → "you didn't recompile." File watcher helps but doesn't eliminate.

Starlark's lockfile + content-addressed artifact pattern was *adopted* into the resolved design — that part of the proposal was good. The eval-time function composition was the part that lost.

---

## III. Surface syntax

### YAML

**Rejected**: indentation footgun (every YAML config tool eventually hits the `no: true` → `no: false` class of bug); anchors are textual inheritance not semantic; no schema in the language itself.

### JSON

**Rejected**: no comments. Configuration files for human authoring need comments. Period.

### TOML

**Rejected**: lists-of-tables get ugly past two nesting levels. The grammar wants to express "a profile with N templates, each with M rules" — that's three nesting levels minimum.

### CUE

**Considered carefully, not chosen**. CUE has the best schema/type story of any candidate (unification semantics, `#Schema` constraints, deterministic JSON export) and made the Kustomize-overlay proposal's case stronger than it otherwise would have been. Lost on: learning curve for users who "just want a config file" and the Kustomize composition model itself losing to pure-additive. If the composition model ever shifts toward overlay-patches in a future major version, CUE deserves a re-evaluation.

### Custom DSL

**Rejected**: parser maintenance tax over the project's life is non-trivial; KDL gives us the same expressiveness with a parser library somebody else maintains; reusing existing grammar means existing editor support.

### Why KDL won

Native ordered nested nodes, native comments, no indentation footgun, peer-shaped data that maps cleanly to set-of-rules, slash-dash (`/-`) for commenting out a rule during debugging, multiple existing Rust parsers. Three of the five first-bake-off strategies independently picked it — strong signal.

---

## IV. IPC for live profile state

### UDS / Windows named pipe to a `tap-daemon`

**Rejected**: introduces a daemon lifecycle bug surface (every hook call hangs or fails-open if the daemon dies); Win10 AF_UNIX port lacks SCM_RIGHTS / abstract namespace / SEQPACKET, so a unified codebase isn't possible — you write two backends anyway. Buys ~5 µs RTT on Unix, none of it visible under the ~30–80 ms Windows `CreateProcess` floor.

### HTTP loopback to a local daemon

**Rejected**: ~130–334 µs per call (UDS beats TCP loopback by 30–66% in benchmarks); AV scans every loopback connect on Windows; port collisions; needs supervision. Strictly worse than UDS for no compensating benefit.

### Shared memory / `mmap` (POSIX `shm_open`; Windows `CreateFileMapping`)

**Rejected**: sub-µs reads are real but invisible under the `CreateProcess` floor. Two different APIs to implement and test (POSIX vs Win32). Requires a seqlock or `memfd` sealing to avoid torn reads. Pays implementation cost for a perf win nobody can perceive.

### Signal-based (SIGUSR1)

**Rejected**: wrong direction. Signals push *to* a process; the hook needs to *pull*. And Windows has no real signals.

### Linux kernel keyring (`keyctl_search(@us, ...)`)

**Rejected** as primary, **kept as future fast-path**. Genuinely elegant — sub-µs reads, kernel-GC'd at logout, atomic at the key level. But Linux-only. If profiling ever shows the file `open`+`read` in the hot path on Linux specifically, swap in keyring. Not now.

### Abstract Unix sockets

**Rejected**: Linux-only.

### CWD-walk for `.tap/profile` (like `.git`)

**Rejected**: seductive because it mirrors `.git` discovery, but silently couples the design to "one agent per worktree forever." The moment two agents share a directory (or one agent `cd`s around inside a monorepo), profile state is ambiguous. The env-var anchor in the resolved design is one extra step at spawn and removes the coupling entirely.

### Process-tree walk (read ancestor's env/cmdline)

**Rejected**: slow on the hot path (`/proc/<ppid>` chains repeated for grandchildren is asymptotic pain); three platform implementations (`/proc` on Linux, `sysctl` on macOS, `NtQueryInformationProcess` on Windows).

### POSIX session id / process group id

**Rejected**: Windows has neither.

### Cgroup membership

**Rejected**: Linux-only and heavy (requires cgroup management).

### PATH-encodes-profile (different shim dir per profile)

**Rejected**: freezes profile at agent-spawn time. Children inherit env. Can't flip mid-flight. Fails the "switch on a dime" requirement.

### Binary-location-encodes-profile (different exe path per profile)

**Rejected**: same as PATH-encodes-profile. Mid-flight flip impossible.

### Inherited file descriptor (`TAP_PROFILE_FD=3`)

**Rejected**: Linux FD inheritance works but is fiddly; Windows FD inheritance has different semantics and known footguns. Not worth the cross-platform burden for a perf win that isn't visible.

### Why file + atomic rename won

Cross-platform identical (rename is atomic for live readers on ext4 / APFS / NTFS); zero daemon (no lifecycle bug surface); ~10–50 µs warm-cache read (three orders of magnitude under the `CreateProcess` floor); ~30 lines of code total. Boring is correct.

---

## V. Language / compile-time

### `rust-script` / `cargo -Zscript`

**Rejected**: nightly-only; invokes `cargo` to stat the script and check cache on every run — measurable tens of ms in practice. Cargo-in-the-loop on the hot path is fatal.

### `scriptisto`

**Rejected**: author claims <1 ms cached overhead, but the project is Unix-first and Windows isn't documented as a first-class target. Cross-platform parity is non-negotiable.

### C++ with a shebang-compile shim (`seabang`, `cpas`, etc.)

**Rejected**: shebang is a Unix kernel feature; Windows always requires a `.cmd` / registry-shim workaround. Plus C++ STL bloats binaries vs Rust/Zig without disciplined effort. The user's intuition that "a script that compiles itself" would be ergonomic was romantic but the cross-platform reality is ugly.

### Zig

**Considered, kept as fallback**. Best-in-class single-file static binaries, fast compile, no Docker for cross-compile, first-class Windows. Lost on: existing toolchain investment is in Rust; Zig is a second compiler on every CI runner. **Revisit if** a standalone single-file distribution outside the existing wheel ever becomes the goal.

### Go

**Rejected**: 2–10 MB minimum binaries (vs ~300 KB stripped Rust); GC runtime pulls in initialization overhead on every cold start; introduces a second toolchain.

### Nim / Crystal / V

**Rejected**: small communities, bus-factor risk for a tool that needs to be maintained for years.

### `tcc -run`

**Rejected**: abandonware risk on the security-critical hot path. Windows port unmaintained; no ARM64.

### Cosmopolitan APE (Actually Portable Executable)

**Rejected**: gives up the Rust ecosystem (cosmo is C/C++; experimental Rust port); Windows AV often flags APE binaries as suspicious because of the polyglot header; macOS Gatekeeper / codesigning is hostile to the format. Multi-OS-from-one-file is a cool property; not worth losing Rust's tooling and signing story.

### Why plain Rust won

Stripped release Rust binary hits <300 KB; cold start 2–3 ms; cross-platform via existing toolchain; no new build dependencies; existing engineering investment. The "scripted policy" angle was a solution to a problem we don't have — users author *profiles* in KDL, not Rust.

---

## VI. Architectural alternatives

### Daemon-based architecture (long-running `tapd` that all shims query)

**Rejected**: every daemon answer (UDS, named pipe, HTTP, shm) trades a ~10 µs file read for: a second process to supervise, a lifecycle bug surface (daemon crash = every tool call breaks or fail-opens), two code paths (Windows can't share daemon code with Unix), and AV-scan tax on Windows connects. The file approach has zero of those failure modes, is one implementation across six platforms, satisfies "switchable on a dime" (rename is instantaneous), and is atomic by POSIX/Win32 rename semantics. The companion `tapd` for stuck-on-denial detection in the resolved design is *optional observability*, not on the hot path.

### Adversarial sandboxing (seccomp / AppArmor / gVisor / bubblewrap / nsjail)

**Rejected as out-of-scope**. `tap`'s threat model is the cooperative agent. Adversarial sandboxing solves a different problem — containment of a hostile process — and requires kernel-level mechanisms with massive implementation and maintenance cost. `tap` runs *inside* a container or microVM if you need adversarial defense; it doesn't replace one.

### Policy DSL engines (OPA / Rego / Cedar-as-engine)

**Rejected**: OPA is heavy (full Rego evaluator); overkill for a CLI hook. Cedar's *shape* (`permit`/`forbid`/`when`) was borrowed as inspiration for the grammar; Cedar's engine wasn't, because (a) the pure-additive set-union model doesn't need a general-purpose policy evaluator, (b) shipping Cedar's Rust crate would meaningfully bloat the binary, (c) Cedar's value is in expressive policy languages and we deliberately chose a minimal grammar.

### Python-file profiles (OpenInterpreter style)

**Rejected**: Turing-complete config language destroys auditability. A reviewer can't tell from reading the file what rules it produces without executing it.

### Symmetric template maintenance (templates published as `.allowlist` AND `.godmode` flavors)

**Rejected** (implicitly, with bidirectional posture). Doubles the maintenance burden on every template author; the two flavors aren't mechanical translations; ecosystem fragments. Single-posture (allowlist semantics with `permit-all` as inversion) gives one set of templates that work everywhere.

### Per-tool wrapper scripts instead of one binary

**Rejected**: every wrapper script is a new mental model, a new failure mode, a new file to keep in sync. The argv[0] trampoline pattern (one binary, many symlinks/copies) is the same proven shape used by busybox, toybox, git's subcommand dispatch, kubectl plugins, gh extensions, mise, volta — wide prior-art convergence.

### Hook-fires-from-inside-agent (Claude Code's PreToolUse model)

**Rejected as primary mechanism, supported as complement**. The in-agent hook approach is blind to grandchild processes — `Bash(cmd="make")` spawning `git push` is invisible to `PreToolUse`. `tap`'s PATH-prefix interception catches the grandchild because the grandchild's `git` also resolves to a `tap-git` shim. The two mechanisms are complementary: in-agent hooks have richer context (they know which tool the agent is *intending* to call, vs `tap` which only sees what was actually exec'd), but only `tap` sees the full process tree.

---

## VII. Out of scope (deliberately)

- **Mid-flight credentials injection / vault integration** — `tap` doesn't manage secrets; it can refuse commands that would exfiltrate them, but it doesn't supply them.
- **Network-level policy** (host allowlist / DNS filtering at L3/L4) — different layer; complementary tools exist (`pi-hole`, network namespaces, container network policy).
- **Cost / rate limiting on AI API calls** — different problem; lives in the agent backend or the orchestrator.
- **Auditing / SIEM integration beyond the JSONL audit log** — adapters can be built downstream; `tap`'s job is to produce a clean log, not to ship it.
- **GUI / web dashboard** — possibly a future companion project; not part of `tap` itself.

---

## How to propose a rejected design (or unblock one)

If you want to revisit something here:

1. Find the specific failure mode documented above.
2. Show how your variant of the proposal *resolves that failure mode* — not just "I think it could work."
3. Open a PR against this doc with the variant, the resolution, and (if applicable) a new failure mode discovered.

The bar is "explain why the original rejection reason no longer applies," not "argue that the alternative has other merits." Other merits are usually real but irrelevant if the original failure mode still bites.
