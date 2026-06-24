# Mission

## What tap is

`tap` is a single binary that sits between an AI coding agent and every tool the agent reaches for — `git`, `bash`, `cargo`, `gh`, `docker`, anything. The agent's `PATH` is owned by `tap`; every command resolves to a `tap` shim first. The shim consults the active profile and decides: permit, revoke (block), or rewrite. The decision is logged. The agent gets back the real tool's output, or a structured refusal.

That is the entire elevator pitch. Three sentences, no marketing.

## Why this exists

AI coding agents do useful work autonomously, and their autonomy carries blast radius. Existing controls are either:

- **Too coarse** (containers, VMs) — high overhead, hostile to interactive workflows, all-or-nothing.
- **Too fine** (seccomp, syscall sandboxes) — adversarial-grade complexity for cooperative-grade need.
- **Application-specific** (Claude Code's hooks, Cursor's allowlist) — invisible to grandchild processes the agent's shell spawns, locked to one agent vendor.

`tap` is the missing middle: a single chokepoint at the **tool-invocation layer**, agnostic to which agent backend is running (Claude Code, Codex, Aider, custom), shaped by swappable profiles, and cheap enough to run on every command without measurable overhead.

## Threat model

`tap` assumes a **cooperative agent**. The model is trying to do what the user asked, sometimes too eagerly or with insufficient context. `tap` is the final check on its intentions before they touch the system.

`tap` is **not** designed to contain a hostile model that is actively trying to escape the sandbox. The cooperative assumption is load-bearing. If you need adversarial sandboxing, use a microVM or `gVisor` or `bubblewrap`. `tap` is what runs *inside* the microVM (or on the laptop) to make day-to-day agentic work safer without giving up the speed and interactivity of running directly against the host.

The distinction matters because the design choices differ. Cooperative-agent containment can lean on social signals (the agent reads error messages and adjusts), explicit refusal messages, and grammar-level guardrails. Adversarial containment cannot trust any signal the contained process produces and must rely on kernel mechanisms.

## What tap is NOT

- Not a container runtime.
- Not a syscall sandbox.
- Not a policy DSL engine (Cedar, OPA — those are too heavy for this use case).
- Not an agent backend (it doesn't run the model).
- Not a CI/CD tool.
- Not a credentials manager.
- Not a replacement for code review.

It's a command filter. Nothing more, nothing less.

## The single-entry-point framing

The agent's allowlist (Claude's tool permissions, Codex's `allowed_tools`, etc.) is configured by the orchestrator to grant exactly one entry: **`tap *`**. Anything the agent wants to run goes through `tap <tool> <args>` — `tap git status`, `tap cargo build`, `tap bash -c "..."`. There is no second verb in its toolbox.

This matters because of how cooperative agents react to refusal. If the agent sees `command not found`, it treats the tool as missing and tries workarounds — `which`, alternate binaries, suggesting `brew install`, hallucinating scripts that approximate the missing tool. Exactly the wrong response. Instead, `tap` returns a **structured refusal** addressed to the agent: *"blocked by profile `phase-1-brainstorm`; tests are forbidden in this phase; switch to `phase-2-fixup` to unblock."* That's actionable conversation, not a dead end.

Discovery is `tap help` (which lists what the active profile permits), not filesystem inspection. The agent learns its capability surface from a single command it already has permission to call.

PATH manipulation — the shim farm under `$TAP_BIN_DIR` — exists as **defense-in-depth** for any path where the agent spawns a shell (Bash tool, `tap bash`). Inside that shell, `git`/`cargo`/`bash` still resolve to `tap` shims, so even grandchildren routes through the filter. But the *primary* mechanism is the agent's tool-allowlist being a single entry, not PATH games. The shim farm makes the boundary leak-proof; the allowlist makes the design legible.

That's the kind of guarantee normally expensive to get (seccomp filters, AppArmor profiles, container syscall allowlists), achieved with one allowlist entry and a directory of shims.

## High-level architecture

- **One binary** (`tap`), deployed under `$TAP_BIN_DIR` as argv[0]-dispatched shims (`tap-git`, `tap-bash`, `tap-cargo`, `tap-gh`, …). The orchestrator prepends `$TAP_BIN_DIR` to the agent's `PATH`.
- **Profile selection** via per-agent state directory pointed to by `TAP_AGENT_DIR`; the active-profile pointer is atomic-renamed for live flips between profiles.
- **Profile grammar**: three verbs (`permit`, `revoke`, `include`); set-union composition; empty profile auto-synthesizes `permit-all *` (full god-mode default).
- **Compile step** produces a content-addressed FlatBuffers artifact with embedded provenance; hot-path is zero-copy `mmap`.
- **Audit tool** (`tap why`) answers "why did this decision happen" with the full provenance ladder.
- **Cross-platform**: Linux, macOS, Windows. Same binary, same grammar, same artifact.

The full architecture is in `ARCHITECTURE.md`.

## Product positioning

> **YOLO by default — your AI agent has god-mode, unless you say otherwise.**

The empty profile permits everything. Adding rules narrows from there. Tightening is opt-in, never automatic.

This positioning is load-bearing: it shapes who the tool is for (developers who already trust their agent broadly and want footgun-removal, not enterprise security teams writing exhaustive allowlists), what the cardinal sin is (silent privilege expansion, not silent privilege restriction), and which strategy the architecture committed to (pure-additive `permit` rules with explicit `revoke`, no posture flag).

## Read next

- **`HISTORY.md`** — the chronological design journey that produced this architecture
- **`ARCHITECTURE.md`** — the full resolved design in implementation detail
- **`REJECTED.md`** — alternative designs explored and the specific reasons each was rejected
- **`CONCEPTS.md`** — the conceptual insights and mental models that shaped the design
