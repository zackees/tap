# Architecture

The resolved design in implementation detail. This is current-truth; deviations should be explained in PR descriptions and propagated here in the same PR.

## Overview in one paragraph

`tap` is a single Rust binary deployed under `$TAP_BIN_DIR` as argv[0]-dispatched shims (`tap-git`, `tap-bash`, `tap-cargo`, `tap-gh`, …). The orchestrator prepends `$TAP_BIN_DIR` to the agent's `PATH`, so every tool the agent invokes resolves to a `tap` shim first. The shim consults the active profile (located via the `TAP_AGENT_DIR` env var → per-agent state directory → atomic-rename pointer to a compiled FlatBuffers artifact), evaluates the agent's command against the rule set, and either forwards to the real underlying binary (resolved via injected `TAP_REAL_PATH`) or refuses with a structured error. Profiles are KDL source files compiled by `tap compile` to content-addressed FlatBuffers artifacts; the hot path is zero-copy `mmap`. Profile switching is atomic-rename of the pointer inside `$TAP_AGENT_DIR`; the next shim invocation sees the new policy. The grammar has three verbs (`permit`, `revoke`, `include`), set-union composition, and an empty profile auto-synthesizes `permit-all *` (full god-mode).

## Components

### The binary (`tap`)

One Rust binary, shipped as a Cargo workspace member. Targets <300 KB stripped, 2–3 ms cold start. Release profile:

```toml
[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

Subcommands:
- `tap <tool> <args...>` — the primary invocation form the agent uses (e.g., `tap git status`, `tap cargo build`). Dispatches to the matching shim handler internally.
- `tap help` — the agent's discovery mechanism. Lists capabilities permitted under the active profile, with brief descriptions and example invocations.
- `tap install` — lay down `$TAP_BIN_DIR/tap` + the shim farm; idempotent.
- `tap compile <profile.kdl> [-o <out>]` — KDL → FlatBuffers artifact.
- `tap profile use <name>` — atomic-rename the active-profile pointer.
- `tap profile diff <a> <b>` — show added/revoked/swapped between two profiles.
- `tap why <rule-id|argv...>` — explain a decision; full provenance ladder.
- `tap doctor` — verify install integrity, PATH ordering, env-var sanity.

Invocation as a shim (argv[0] ∈ {`tap-git`, `tap-bash`, ...}) goes through the trampoline path, not the subcommand dispatcher. The shim path exists primarily as defense-in-depth — see "The agent's view of available commands" below.

### The agent's view of available commands

The orchestrator configures the agent's tool-allowlist (Claude's tool permissions, Codex's `allowed_tools`, etc.) to grant **exactly one entry: `tap *`**. Every action the agent wants to take is `tap <tool> <args>`. The agent has no other verb in its toolbox.

Discovery happens via `tap help`. The output enumerates capabilities permitted under the *active profile*, not the static install — so when the profile changes, the agent's reachable surface visibly changes too. Example:

```
$ tap help
Active profile: phase-2-fixup
Capabilities permitted:
  git       — read operations (status, log, diff, show, fetch)
  cargo     — build/test/check/fmt/clippy (no publish)
  bash      — lint/test/build scripts; no destructive ops
  fs        — read/write inside $CWD
Capabilities forbidden:
  release, gh, docker, network (except localhost)
Switch profiles: tap profile use <name>
Explain a denial: tap why <argv...>
```

This is the agent's only discovery mechanism. It does *not* `ls $TAP_BIN_DIR` to learn what's available; it doesn't probe with `which`; it doesn't read the shim filesystem at all. The well-configured agent simply reads `tap help` and works from there.

Why this matters: cooperative agents react badly to `command not found`. They interpret it as "tool not installed" and try workarounds — alternate binaries, `brew install`, hallucinated scripts. The single-entry-point design eliminates the `command not found` path entirely. Every refusal is a structured message addressed to the agent, in the same shape as a permitted command's output. The agent learns to stop and report BLOCKED, not to try to route around the missing tool.

### The shim farm (defense-in-depth)

The agent's allowlist is the primary boundary. The shim farm is the *secondary* boundary, for any path where the agent (or a child process it spawns) reaches for tools by name on `PATH` rather than via the `tap` allowlist entry. The canonical case: the agent uses its allowlisted `tap bash -c "..."` to spawn a shell, and the shell's child processes look up `git` / `cargo` / `docker` on `PATH`. Without the shim farm, those children would resolve to the real binaries and escape the filter. With the shim farm, they resolve back into `tap`.

So the shim farm is real, necessary, and load-bearing — but it serves a different purpose than the agent's allowlist. The allowlist makes the design *legible* (one rule: only `tap`). The shim farm makes the boundary *leak-proof* (even spawned grandchildren route through `tap`).

`$TAP_BIN_DIR/` contains one real `tap` binary and N argv[0]-dispatched aliases. **Install strategy is OS-specific**:

| OS | Strategy | Reason |
|---|---|---|
| Linux, macOS | Symlinks | Atomic creation; survives in-place binary replacement |
| Windows | **Copies** by default; NTFS hardlink if source and dest share a volume | Symlinks require admin or Developer Mode; copies are universally safe; never `.cmd` shims |

Argv[0] parsing (cross-platform):

```rust
let invoked = std::env::args_os().next().unwrap();
let name = Path::new(&invoked)
    .file_stem()        // strips .exe on Windows, no-op on Unix
    .and_then(|s| s.to_str())
    .ok_or(BadInvocation)?;
let tool = name.strip_prefix("tap-").ok_or(NotAShim)?;
```

Use `file_stem()` (not `file_name()`) for unified `.exe`/no-extension handling. Compare argv[0] (unresolved) for dispatch and `current_exe()` (resolved via `canonicalize()`) for "am I the shim binary?" safety checks.

**Finding the real underlying binary**:
1. Read `TAP_REAL_PATH` env var (orchestrator injects an absolute path or a sanitized search PATH at agent launch). Preferred.
2. Fallback: walk `$PATH` skipping any directory containing `canonicalize(current_exe())`. Return first match.
3. Safety net: if resolved real path equals `canonicalize(current_exe())`, hard-abort with a clear error. Prevents fork bombs from misconfigured PATH.

**Shim farm composition is constant across profile flips**. When a tool is `revoke`d by the active profile, the shim stays present and refuses with a structured stderr message — it does *not* get unlinked from PATH. Reasons:
- Cooperative agents respond to structured refusal messages by stopping and reporting BLOCKED. Removing the shim entirely returns `command not found`, which the agent may interpret as "cargo isn't installed" and hallucinate workarounds like running rustup.
- Atomic profile flip needs zero filesystem churn on PATH. Symlink storms on Windows are slow and racy.

An opt-in `unlink-on-deny` flag may be added later for paranoid profiles that prefer command-not-found semantics — not v1.

### The profile system

Per-agent state lives under a directory pointed to by the env var `TAP_AGENT_DIR`:

```
$TAP_AGENT_DIR/
├── profile.active          # atomic-rename target — points to a compiled artifact
├── compiled/
│   ├── <sha256>.fbs        # content-addressed FlatBuffers artifacts
│   └── <sha256>.lock       # input-file hashes for the artifact
└── audit.log               # JSONL audit trail per agent
```

The orchestrator sets `TAP_AGENT_DIR` once at agent spawn time and includes it in the environment of every child process. Per-agent isolation is structural: different agents get different `TAP_AGENT_DIR` values; flipping Agent A's profile cannot affect Agent B.

**Profile flip mechanism**:

```bash
# orchestrator-side
tap compile profiles/ship.kdl -o $TAP_AGENT_DIR/compiled/staging.fbs
mv $TAP_AGENT_DIR/profile.active.tmp $TAP_AGENT_DIR/profile.active   # atomic
```

The shim reads `profile.active` on every invocation. Warm-cache read is ~10–50 µs (single `openat` + `read` + `close`), three orders of magnitude under the ~30–80 ms Windows `CreateProcess` floor — invisible in practice.

**Fallback when `TAP_AGENT_DIR` is unset**: `~/.tap/profile.active` for single-agent dev convenience. Log a one-time warning so the user notices if their orchestrator forgot to inject the env var.

## Profile grammar

Source format: **KDL**. Reasons KDL beat the alternatives: native ordered nested nodes, native comments, no YAML indentation footgun, peer-shaped data that maps cleanly to set-of-rules, slash-dash (`/-`) for commenting out a rule during debugging.

### Three verbs, set-union composition

The entire grammar:

```kdl
permit "rule-id" capability="git" {
    match argv-prefix="push origin"
    rewrite drop-flag="--force"   // optional
    reason "we don't force-push from this profile"
}

revoke "template-id/rule-slug"

include "template-name"
```

- **`permit`** introduces a rule. Optional `id`, capability, match predicate, rewrite directives, human-readable reason.
- **`revoke`** removes a rule by stable ID. Idempotent. Order-independent. Revoke-of-nonexistent-ID is a hard compile error.
- **`include`** pulls in a template by name. Resolved by the compiler against the template search path.

**Composition algebra** is two-phase and explicit:

1. **Union** all `permit` rules across the profile + all included templates.
2. **Subtract** all `revoke` rules as a final set-difference pass.

The resolved policy is a literal set. Order doesn't matter. There is no precedence, no specificity, no cascade, no MRO. The reviewer reads the union, sees every rule, knows that's the full universe.

### The synthesized default

An *empty* profile (no `permit` rules surviving the revoke pass) → the compiler emits a synthetic `permit-all *` rule covering every shim. This is the "YOLO by default" guarantee: `tap` with no profile authored = god-mode for everything.

The instant any `permit` rule survives, the synthesis stops. The profile is now the literal union of its surviving permits, minus the union of its revokes. This is the single biggest UX win in the grammar: simple profiles say what they want, restrictive profiles say what they want, and the boundary between "empty/god-mode" and "narrowed" is structurally explicit.

### Rule IDs

Namespaced: `<template-id>/<rule-slug>`. Examples:
- `tap.safe-git/push.main.deny`
- `acme.network.corp/host.github.com.permit`
- `local/<content-hash>` (synthesized for unnamed inline rules; users never type these)

Stable rule IDs are a *public API contract* for any template that expects to be revoked downstream. Renaming a rule in a published template is a breaking change; template authors should follow semver and publish a `RENAMES` table for migrations.

### Capability taxonomy

Capabilities are a coarse grouping axis for rules: `git`, `shell`, `network`, `filesystem`, `package-mgmt`, `release`, `tests`, `build`, `container`, `sub-agent`, `fanout`. The taxonomy is fixed by the binary (extensions added via PR) but is purely organizational — capabilities don't carry default semantics, don't gate verb choice, and don't compose. They exist for:

- Visual grouping in `tap why` output
- Rule-ID namespacing
- Filterable audit logs

## Template library

Ships with the binary under a well-known namespace `@tap/`:

**Positive templates** (additive — open capabilities):
- `@tap/god-mode-bash` — `permit-all "shell"`
- `@tap/god-mode-net` — `permit-all "network"`
- `@tap/god-mode-git` — `permit-all "git"`
- `@tap/rust-toolchain` — `cargo`, `rustc`, `rustup`, `soldr` permits
- `@tap/docker` — `docker build`, `docker run --rm`, mount paths under `$CWD`
- `@tap/git-pr` — `git commit`, `git push origin HEAD`, `gh pr *`, `gh api`
- `@tap/read-edit` — `fs.read`, `fs.write`, scratch dirs
- `@tap/research` — `net.http.get`, `gh issue read`, web search
- `@tap/swarm` — `agent.spawn`, `agent.message`

**Negative templates** (subtractive — close capabilities; revoke-only contents):
- `@tap/no-toolchain` — revokes cargo/rustc/bash test/lint/build/soldr/docker
- `@tap/no-git` — revokes git, gh
- `@tap/no-network` — revokes all net.* except localhost
- `@tap/no-mutation` — revokes fs.write, all git mutating verbs

**Editorial templates** (curated wisdom, opt-in):
- `@tap/safe-defaults` — revokes the one-way-door capabilities: release, package-publish, credentials, destructive-fs (`rm -rf /`, etc.)

The negative templates are *mandatory* infrastructure, not optional sugar. They are the structural answer to "god-mode by default" when the user wants restrictive phases. Without them, restrictive profiles become hand-maintained denylists that silently break when new shims are added to the farm later.

## Compile step

`tap compile <profile.kdl>` produces a FlatBuffers artifact:

```
input:  profile.kdl + all transitively-included template KDL files
output: $TAP_AGENT_DIR/compiled/<sha256>.fbs   (atomic write)
        $TAP_AGENT_DIR/compiled/<sha256>.lock  (input hashes)
```

The artifact is **content-addressed** by sha256 of the canonicalized resolved policy. Reasons:
- Identical resolved policy → identical artifact → cache hit
- Rollback is `ln -sf <old-sha>.fbs profile.active`
- Tampering is detectable (and a future signing layer can verify provenance)

The lockfile records hashes of every input KDL file. `tap compile --check` re-hashes inputs and exits non-zero if drift is detected.

**FlatBuffers schema** (sketch):

```fbs
table CompiledProfile {
    schema_version: uint16 (required);     // bump on breaking changes
    min_tap_version: string (required);    // semver
    posture_marker: ubyte (required);      // always 0 in v1 (no posture); reserved
    rules: [CompiledRule] (required);
    provenance: [SourceSpan] (required);
}

table CompiledRule {
    id: string (required);
    capability: uint32;                    // interned
    action_pattern: uint32;                // interned, Aho-Corasick-compiled
    decision: Decision (required);         // Permit | Rewrite | (no Forbid — see below)
    rewrite_directives: [RewriteOp];
    provenance_idx: uint32;                // index into provenance table
    reason: string;
}

enum Decision : ubyte { Permit = 0, Rewrite = 1 }
```

Note: there is no `Forbid` decision in the compiled form because the grammar has no `forbid` verb. The compiled rule set is *only* the surviving permits after the revoke pass; "forbid" is the *absence* of a matching permit.

**Schema versioning is mandatory from v1**. Adding `schema_version` and `min_tap_version` upfront prevents the silent fail-open on version skew that would otherwise happen if an old `tap` shim read a new artifact (ignoring unknown fields and falling through to a hardcoded default).

## Evaluator algorithm

Hot path on every shim invocation:

1. Resolve `TAP_AGENT_DIR`. Fallback to `~/.tap/` with a one-time warning. If unset and no fallback, **fail closed**.
2. Open `$TAP_AGENT_DIR/profile.active`, mmap.
3. Validate `schema_version`. Older shim than artifact requires → refuse, fail closed.
4. Look up `(capability, argv)` against the compiled rule set. Aho-Corasick over the action-pattern table; rules are pre-sorted by capability for cache locality.
5. If a `Permit` matches → exec the real binary with optional rewrite directives applied.
6. If no rule matches → refuse with a structured stderr message; exit 126.
7. Append a JSONL audit event to `$TAP_AGENT_DIR/audit.log`.

Steps 2–4 are zero-copy mmap reads. Step 4 is `O(log n + m)` where m is the matched-pattern length. Realistic per-invocation overhead is dominated by `CreateProcess` (Windows: 30–80 ms; Linux/macOS: 2–10 ms), not by the evaluator.

## Error message protocol

When the shim refuses:

```
Exit code: 126 (semantically: "command found but cannot execute")
Stderr (structured + human-readable):

  tap: refused
  command:  git push --force origin main
  capability: git
  reason:   no active permit covers this action
  profile:  phase-4-ship (compiled from profile.kdl)
  why:      tap why git.push.force-main
  next:     to permit, run `tap profile use phase-1-brainstorm` or edit profile.kdl
```

Stdout: empty (so calling tools that parse stdout don't get garbage).

Exit code 126 was chosen over 1 because it's the POSIX convention for "command found but not executable," and cooperative agents are more likely to inspect stderr and emit BLOCKED than they are to retry with `--force` or `sudo`.

## Audit tool: `tap why`

`tap why <argv...>` answers "why did this decision happen" by replaying the evaluation with verbose provenance:

```
$ tap why git push --force origin main
DENIED (no matching permit)
evaluated against profile: phase-4-ship
rules considered for capability "git": 8
  permit "tap.git-pr/push.origin.head"     argv-prefix="push origin HEAD"      no match
  permit "tap.git-pr/push.upstream"        argv-prefix="push --set-upstream"   no match
  ...
no rule covered (capability=git, argv=push --force origin main)
default: DENY (allowlist mode; 8 permit rules narrowed away the synthesized permit-all *)

provenance for closest miss:
  tap.git-pr/push.origin.head defined at @tap/git-pr.kdl:14
    included via: profile.kdl:5  include "@tap/git-pr"
```

For decisions that hit the synthesized `permit-all *` (empty profile), `tap why` says so explicitly: *"permitted by default (no rules in profile; synthesized permit-all * is active)"*.

## Profile diff: `tap profile diff`

```
$ tap profile diff brainstorm fixup
brainstorm → fixup
  +include "@tap/rust-toolchain"
  +include "@tap/project-scripts"
  -revoke "exec.cargo.**"
  -revoke "exec.rustc.**"
  -revoke "exec.bash" args="lint" args="test"
  rules: +12 -3 net change +9 surviving permits
```

For team review, incident forensics, and "what changed between Monday and Friday." Borrowed unchanged from Strategy D's strongest moment in the bake-off.

## Orchestrator setup

The orchestrator has two responsibilities at agent launch:

1. **Configure the agent's tool-allowlist to contain exactly one entry: `tap *`.** This is the primary boundary. How this is done depends on the agent backend — Claude Code's `--allowedTools`, Codex's `allowed_tools` config, etc. The agent literally has no other tool permission.

2. **Set up environment + shim PATH as defense-in-depth.** In case the agent's allowlist permits a shell that itself spawns child processes by name, those children must still route through `tap`.

```bash
# Allowlist configuration (agent-backend-specific)
export CLAUDE_ALLOWED_TOOLS="tap"      # or equivalent for whichever backend

# Defense-in-depth: PATH + env
export TAP_BIN_DIR=$HOME/.local/share/tap/bin
export TAP_AGENT_DIR=/run/tap/agents/$(uuidgen)
export TAP_REAL_PATH="$PATH"
export PATH="$TAP_BIN_DIR:$PATH"

mkdir -p $TAP_AGENT_DIR
tap compile profiles/brainstorm.kdl -o $TAP_AGENT_DIR/compiled/brainstorm.fbs
ln -sf $TAP_AGENT_DIR/compiled/brainstorm.fbs $TAP_AGENT_DIR/profile.active

exec claude   # or whichever agent backend
```

The agent inherits `TAP_BIN_DIR`, `TAP_AGENT_DIR`, `TAP_REAL_PATH`, and the modified `PATH`. The allowlist ensures the agent only ever invokes `tap` directly. The PATH manipulation ensures any subprocess the agent spawns (via `tap bash -c "..."`) still has its children intercepted. `TAP_REAL_PATH` is the only way past the shim farm — and the only thing that uses it is `tap` itself, looking up the real underlying binary.

## Cross-platform

| Concern | Linux | macOS | Windows |
|---|---|---|---|
| Shim install | symlinks | symlinks | copies (+ NTFS hardlink optimization) |
| argv[0] dispatch | `args_os().next()` | same | same (`file_stem()` strips `.exe`) |
| Atomic rename | `rename(2)` | `rename(2)` | `MoveFileExW + MOVEFILE_REPLACE_EXISTING` |
| Profile file read | `open + read + close` | same | same |
| Real-binary lookup | `TAP_REAL_PATH` env, else PATH walk | same | same |
| Cold-start budget | 2–4 ms | 5–10 ms | 30–80 ms (Defender RT scan dominant) |
| Defender exclusion | n/a | n/a | document: exclude `$TAP_BIN_DIR` to amortize first-call scan |

The hot path is identical across platforms. Install diverges per-OS for symlink-vs-copy. WSL2: `tap.exe` invoked from inside WSL via interop has a Linux-style argv[0]; this path is tested.

## Failure modes and defaults

**Fail closed by default** when any of the following is detected:
- `TAP_AGENT_DIR` unset *and* no `~/.tap/profile.active` exists
- `profile.active` is missing or unreadable
- Compiled artifact fails FlatBuffers schema validation
- `schema_version` of artifact > supported by this shim
- Resolved real-binary path equals `current_exe()` (PATH loop / fork bomb risk)

**Fail open** is gated by `TAP_FAIL_OPEN=1` env var, intended for bootstrap and orchestrator debugging. Loud warning on stderr every invocation when fail-open is active.

**Schema version mismatch** (artifact requires newer `tap` than the shim has): refuse to evaluate, refuse to execute, exit 126 with a clear error pointing at the install/upgrade path. **Never silently downgrade.**

## Stuck-on-denial detection

Optional companion daemon (`tapd`) watches `$TAP_AGENT_DIR/audit.log` and surfaces a notification to the dev when an agent has been emitting denials at a high rate (default threshold: ≥3 denials of the same capability in 30 s). Without this, "dev forgets to flip phase" becomes "silent agent deadlock."

The daemon is optional infrastructure, not part of the hot path. Agents work fine without it; the daemon only affects observability.

## Implementation roadmap (rough phases)

1. **Grammar + compile**: parse KDL, validate, produce FlatBuffers artifact, content-addressed cache, lockfile, unit tests for every grammar edge case. This is what justified hoisting `tap` out of the parent project — fast unit tests on grammar compilation, no integration cost.
2. **Shim trampoline**: argv[0] dispatch, real-binary resolution, fork-bomb safety net, cross-platform install/uninstall, doctor command.
3. **Evaluator**: mmap loader, Aho-Corasick over patterns, decision lookup, structured refusal, audit-log JSONL.
4. **CLI**: `tap install`, `tap compile`, `tap profile use`, `tap profile diff`, `tap why`, `tap doctor`.
5. **Template library**: `@tap/safe-defaults`, `@tap/god-mode-*`, `@tap/no-*`, `@tap/read-edit`, `@tap/rust-toolchain`, etc. Ships with the binary.
6. **Daemon (optional)**: `tapd` for stuck-on-denial detection and audit-log aggregation.
7. **Integration**: orchestrator-side glue for spawning agents under `tap`. Lives in whichever parent project drives it (clud, or a new launcher).

## Where to extend

- New shimmed tool → add to the install list; verify argv[0] parsing handles its invocation style; ship a `@tap/<tool>-defaults` template if it has nontrivial sub-commands.
- New capability → add to the taxonomy enum; document grouping intent; no code changes in the evaluator (capabilities are organizational, not semantic).
- New rule predicate → extend the KDL parser; add corresponding match logic in the evaluator; bump `schema_version`.
- New decision type beyond `Permit`/`Rewrite` (e.g., `LogOnly`) → bump `schema_version`; older shims refuse to load and fail closed.
