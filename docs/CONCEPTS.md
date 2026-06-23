# Concepts

The thinking tools that shaped `tap`. These are durable principles, separable from any specific decision in `ARCHITECTURE.md`. When a future design question comes up, these are the lenses to apply.

---

## 1. The single-lattice principle

Set-union composition has three useful algebraic properties: commutativity (`A ∪ B = B ∪ A`), associativity (`(A ∪ B) ∪ C = A ∪ (B ∪ C)`), and idempotence (`A ∪ A = A`). Together they mean *order doesn't matter*. That's what makes the resolved policy a literal set — reviewable, diffable, predictable.

These properties only hold for **one lattice at a time**. The moment a second composition lattice is layered on top — posture per capability, levels per capability, modes per profile — at least one of the properties breaks. Commutativity goes first. The reviewer can no longer read the union and know the answer; they have to mentally evaluate the *interaction* between the two lattices.

**The principle**: keep the composition surface one-dimensional. If a new feature seems to need a second lattice, push it down into the existing one (e.g., per-capability god-mode via *include of a template*, not via *attribute on a capability*).

---

## 2. Alpha command, top dog

The agent's view of the system is whatever resolves on its `PATH`. We don't compete for resolution priority — we own resolution. Whatever directory we prepend to `PATH` wins every lookup, before the OS even consults `/usr/bin` or `C:\Windows\System32`.

This flips the design from defensive to declarative:

- **Allowlist by presence, not by policy.** Tools we don't symlink into our bin directory don't exist from the agent's perspective. `command not found`, full stop. No interception logic needed for the long tail.
- **No race between "wrap it" and "agent finds it."** Adding a new tool is a one-line symlink. Not adding it is also a complete answer.
- **You become the resolver, not a participant in resolution.** The shim calls the real binary by absolute path (resolved once at install time, baked in or stored in config). Even inside the shim, there's no PATH lookup the agent could poison. The chokepoint is total.

The kind of guarantee normally expensive to get (seccomp filters, AppArmor profiles, container syscall allowlists), achieved with a directory and a sort order.

---

## 3. Filter and hook as orthogonal primitives

A productive interaction between an agent and a system has two components:

- **Filtering**: what set of actions is the agent allowed to *attempt* at all. (PATH ownership, shim presence/absence.)
- **Hooking**: what happens to each allowed action when it's invoked. (The policy evaluator, permit/revoke/rewrite, the structured refusal.)

Most existing agent tooling collapses these together — Cursor's allowlist is both filter and hook; Claude Code's hooks try to do filtering inside the tool dispatcher. Treating them as separate composable primitives is high leverage: it lets you change one without touching the other (swap the policy file mid-session; the filter is constant), it makes the security boundary single-purpose, and it lets each layer be debugged independently.

`tap` reifies the split: the **shim farm** is the filter (which commands exist), the **profile evaluator** is the hook (what happens per command).

---

## 4. Posture is a product question, not a configuration question

This was the strongest argument for Strategy C (capability-defined defaults), even though C ultimately lost. The insight survives the loss.

**The argument**: "should `git push --force` be off by default?" is a *product* decision. It has one right answer for a given audience and a given threat model. Configuration languages are a terrible medium for product decisions because every user re-litigates them, every team's config diverges, the cognitive load compounds, and the maintainer never gets to converge on a recommended baseline.

C wanted to bake these decisions into the binary's capability schema. We rejected that for editorial-burden and capability-≠-binary reasons. But the *insight* — that editorial wisdom is real and should be delivered to users, not invented by every team from scratch — is preserved in the resolved design as the **template library** (`@tap/safe-defaults`, `@tap/god-mode-*`, `@tap/no-*`). Same editorial wisdom; users opt in; maintainer carries no public-API commitment.

**The generalization**: when you're tempted to make something configurable that *should* have one right answer for your audience, ship the right answer as a default template and let users opt out. Don't make them re-litigate it.

---

## 5. The cardinal sin in security tools is silent precedence shifts

Cascade (CSS-style), MRO (OOP-style), and overlay-stack (Kustomize-style) all use implicit ordering to resolve rule conflicts. They all share a failure mode: **a permit accidentally shadows a forbid, and the agent gains a capability nobody granted**. The failure is silent — the file looks correct in isolation; only the interaction with the rest of the rule set causes the leak.

Cursor's denylist had 4 such bypasses found in their 1.3 release; they deprecated denylists entirely as a result. This is not a hypothetical class of bug — it's the dominant real-world failure mode of security tools that use implicit precedence.

**The principle**: in any composition mechanism where one rule can override another, prefer mechanisms where:
- The override is *explicit* in the source (a named verb, not an attribute interaction)
- The override is *local* to one file (no precedence rule spans multiple files)
- The reviewer can answer "did this rule fire?" by reading one line

Pure-additive + revoke satisfies all three. Cascade and MRO satisfy none.

---

## 6. Subtraction over a moving target

When the default is god-mode (permit everything), making the system "safe" requires *subtracting* every dangerous capability. That subtraction is hand-maintained against a *moving target* — new shims get added to the farm over time, and existing subtractive profiles don't automatically include them.

Concretely: a brainstorm profile that revokes `cargo`, `rustc`, `bash test`, `bash lint`, `docker`, `git`, `gh` looks complete today. Next quarter, `tap-aws` ships. The brainstorm profile silently grants AWS access until someone notices and writes a new revoke.

**The principle**: when the default points toward "open," restrictive profiles must be *positively constructed* (declare what's allowed) rather than *negatively constructed* (declare what's forbidden). In the resolved design, this means restrictive profiles `include "@tap/read-only"` + `include "@tap/research"` + a few more positive templates — and the negative templates (`@tap/no-toolchain`, `@tap/no-git`) are the structural answer to "I want to take a bunch of things away cleanly."

Negative templates aren't a workaround; they're the structural answer to the moving-target problem. They are *mandatory infrastructure* in any god-mode-default design, not optional sugar.

---

## 7. Cardinal sin inversion

The dangerous mistake differs based on the default-state semantics:

- **Allowlist (default-deny)**: the cardinal sin is *accidentally permitting* something dangerous. A typo in an allow rule, a too-broad pattern, a shadow from a precedence rule — all paths to silent privilege expansion.
- **God-mode (default-permit)**: the cardinal sin is *accidentally un-forbidding* something you needed. A typo in a revoke ID that silently no-ops; a template renaming a rule downstream profiles depended on; a new shim added to the farm that retroactively becomes permitted.

These failure modes are mirror images. A unified design that tries to handle both directions ends up failing-open in one direction or failing-closed in the other — there is no neutral default. The choice of which sin to make easier to commit is essentially a choice of which audience to serve.

`tap` chose god-mode (matches the YOLO product positioning), which means the cardinal sin to defend against is silent un-forbidding. The defenses: stable rule IDs as public-API contract, hard compile error on revoke-of-missing, content-addressed artifacts that diff cleanly, the `tap profile diff` tool that surfaces "what changed."

---

## 8. Lexical polarity collision

When the same syntax means opposite things in different contexts, you've built the JavaScript-`this`-binding class of bug. The reader can't determine semantics from the local code; they have to hold a distant context in their head.

This is what killed bidirectional posture. `revoke "rule-id"` in a god-mode profile loosens (re-grants); the same line in an allowlist profile tightens (takes away). The verb's direction is invisible. Walking through a code review of a PR that adds `revoke "fs/write.tmp.permit"`, a reviewer cannot determine whether this is a hardening or weakening change without chasing the target's posture across multiple files.

**The principle**: every load-bearing token should mean the same thing everywhere it appears. If a verb can mean two opposite things, the right answer is two verbs (or one verb plus elimination of one direction).

---

## 9. Cooperative agent ≠ adversarial agent

The threat model determines the design space. A cooperative agent is trying to do what the user asked; sometimes it's overeager or under-informed. An adversarial agent is actively trying to escape constraints. The two cases need different tools:

- **Cooperative**: can lean on social signals (the agent reads error messages and adjusts), explicit refusal messages, grammar-level guardrails. The agent stops when told to stop. PATH-prefix interception + structured refusal is sufficient.
- **Adversarial**: cannot trust any signal the contained process produces. Needs kernel-level mechanisms (seccomp, namespaces, syscall filters, capabilities). The agent will probe for escape vectors, retry with different argv, attempt to corrupt the policy, exploit race conditions.

`tap` is in the cooperative camp by design. Adversarial-grade containment is out of scope; if you need it, run `tap` *inside* a microVM or container. The cooperative assumption is load-bearing — much of the design (refusal messages addressed to the agent, PATH-prefix that the agent could theoretically subvert if it tried, fail-closed defaults that depend on the agent not actively corrupting state) is justified by the cooperative threat model.

---

## 10. Curation scales better than expression — and the counter

Strategy D's central bet: a curated rule library (~200 hand-maintained rules across 5 capabilities × 4 levels) gives users a shared vocabulary and saves them from re-deriving safety judgments. "We run `git=restricted, network=restricted`" communicates more than any rule list.

The counter that won: **curation scales better than expression *only when the user's needs map onto the curated set*.** When they don't — when the dev needs `git=commit-push-feature-branch-only` and no such level exists — the user falls back to expression anyway, and now has *both* abstractions to debug. The leak is worse than starting with expression.

**The synthesis we landed on**: ship curated templates (so users *can* use them as a shared vocabulary) without baking curated defaults into the binary (so users aren't *forced* to). Editorial wisdom delivered as opt-in artifacts is the best of both worlds — discoverable, ignorable, replaceable.

---

## 11. The editorial burden axis

Every opinion baked into the binary becomes public API forever. Changing it is a breaking change for every downstream consumer. The maintainer is now in the policy-judgment business for the life of the project.

The axis runs from "user-authored" (no editorial burden on the maintainer; users own all decisions) to "binary-baked" (full editorial burden; every default is a forever-API). Strategies cluster along it:

- **A** (drop posture): minimal editorial burden — three verbs, no opinions
- **B** (profile-scoped posture): minimal burden — bidirectionality is a mechanism, not an opinion
- **E** (two-system split): minimal burden — same as B
- **C** (capability-defined defaults): high burden — the capability schema is forever-API
- **D** (sandbox levels): high burden — the ~200-rule level matrix is forever-API

For a solo-maintainer or small-team project, "high burden" is too expensive. The opinions are still real, still useful — but deliver them as templates the user includes, not as defaults the user can't escape.

---

## 12. Content-addressing as a security primitive

The compiled artifact is stored as `<sha256>.fbs`. The lockfile records hashes of every input source. Together this gives you four properties for free:

1. **Reproducible builds**: identical resolved policy → identical artifact → cache hit. Lockfile + hashes catches drift.
2. **Easy rollback**: `ln -sf <old-sha>.fbs profile.active` is the entire rollback operation.
3. **Tamper-evidence**: the artifact's filename IS its hash. Modifying the artifact requires renaming it. A future signing layer verifies the maintainer-signed sha matches the filename.
4. **Diffable history**: every transition leaves an artifact in `$TAP_AGENT_DIR/compiled/`. Audit answers "what was the policy at 14:32 yesterday" by walking the audit log to find the sha and loading that artifact.

This costs nothing at the architecture level — it's the natural way to handle compile output — but pays for itself in operational properties most config systems lack.

---

## 13. Fail-closed as the only honest default

A security tool with permission to refuse must default to refusing when its state is missing, unreadable, or corrupt. *"I can't read the policy"* should never mean *"everything is permitted,"* even when the product positioning is YOLO-by-default.

The two have different domains:
- "YOLO by default" applies when the policy is *intentionally empty* (empty profile → synthesized `permit-all *`). The user has explicitly chosen god-mode.
- "Fail-closed" applies when the policy is *unintentionally absent* (env var unset, file missing, schema mismatch). The user has not made any choice; the system is in an undefined state.

Conflating these two is the path to silent privilege expansion under transient failure (NFS hiccup, disk-full, version skew, attacker who can `unset TAP_AGENT_DIR`). The resolved design fails closed by default and gates fail-open behind an explicit `TAP_FAIL_OPEN=1` env var intended only for bootstrap and debugging.

---

## 14. Profile diff as a first-class operation

Borrowed unchanged from Strategy D's strongest moment in the bake-off. The insight: **profile transitions are the most important review surface in the system, and making them structurally diffable is table-stakes regardless of composition strategy.**

A 40-line rule diff between `brainstorm.kdl` and `fixup.kdl` is unreadable. A two-line summary — *"added include `@tap/rust-toolchain`, revoked `exec.git.**`"* — is reviewable in seconds. The summary is generated by walking the *resolved policy* (after compile) and diffing rule-sets, not by diffing source text.

This unlocks team review (PR descriptions can show the policy delta), incident forensics ("when did we go from `brainstorm` to `ship-it` and what changed?"), and onboarding (new contributors can read profile diffs to understand workflow boundaries).

---

## 15. Optional infrastructure must stay optional

The daemon (`tapd`) for stuck-on-denial detection, the future template registry, the future signing layer, the future audit-log aggregation — all of these should be optional. The hot path (every shim invocation) must work with zero external dependencies beyond the binary, the env var, and the file system.

The principle generalizes: **anything that makes the system better is fine to add as opt-in; anything that's required for the system to function is on the critical path and must be reviewed against the single-binary single-responsibility design**. The daemon being optional means the design can ship in v1; the daemon being required would mean another v0 of design work on lifecycle, supervision, failover.

---

## 16. The "boring wins" principle

Across nearly every design decision, the boring option won:

- File + atomic rename, not shared memory or daemon IPC
- Plain Rust binary, not script-compile or APE
- KDL (existing format), not custom DSL
- Set-union, not cascade or MRO
- Three verbs, not nine
- Content-addressed FlatBuffers, not signed protobuf with extension fields
- Static capability taxonomy, not user-extensible
- Constant shim farm, not dynamic-by-profile

This isn't accidental conservatism — it's the consequence of operating in a domain (security-adjacent, performance-sensitive, cross-platform, long-lived) where every additional moving part compounds into bug surface. The boring options are boring because they've been debugged for decades. The exciting options are exciting because they haven't.

**The principle**: when in doubt, ship the boring option. The exciting option can replace it in v2 if the boring option turns out to be the bottleneck.

---

## 17. The "agent and the filter are not peers" reframe

Most subprocess-sandboxing tools assume the contained process and the sandbox compete for resources, time, syscalls, and namespace entries. They're treated as adversaries even when the relationship is cooperative.

`tap` rejects this framing. The agent doesn't compete with `tap` for `PATH` priority because `tap` *constructs* the `PATH` the agent inherits. The agent doesn't have a way to find the real `cargo` because the real `cargo` isn't on its `PATH` at all. The chokepoint is total *by construction*, not by interception.

This reframe is what makes universal interception cheap. Adversarial sandboxes pay for every syscall because they have to assume the process is hostile. `tap` pays once per command (the shim invocation overhead) because it owns the namespace the command resolves in.

---

## 18. Implementation cost is a feature, not a bug

The simpler design always won in this domain. Across five composition strategies, five resolution strategies, five language choices, six IPC mechanisms, multiple syntax candidates — the winners were consistently the lowest-complexity option that met the hard requirements.

The reason: **complexity in a security-adjacent tool compounds into bug surface**. Every additional axis of configurability is another way for the user to misconfigure; every additional dispatch path is another way for the evaluator to silently disagree; every additional dependency is another supply-chain risk.

The principle generalizes beyond `tap`: when designing a tool whose job is to *constrain* something, the design itself must be constrained. A policy engine with a Turing-complete config language is its own attack surface. A composition mechanism with implicit precedence is its own footgun. The discipline of "what can we cut" is more valuable than the discipline of "what can we add."

---

## Where these came from

Most of these concepts crystallized during the second round of adversarial review (the 5-lens bidirectional-posture stress test) and the zccache workflow validation. They weren't designed; they emerged from watching specific failure modes recur across very different strategies and noticing the shared cause.

If a future design proposal violates one of these, the proposal isn't automatically wrong — but the burden is on the proposal to articulate why the principle doesn't apply, not on the maintainer to re-derive why it does.
