# tap

Universal command filter for AI coding agents.

`tap` is a single binary that mediates every action an AI coding agent takes. The agent's tool-allowlist contains exactly one entry — `tap *` — so every action it performs is `tap <tool> <args>`: `tap git status`, `tap cargo build`, `tap bash -c "..."`. `tap` consults the active profile and decides: permit, revoke, or rewrite. Decisions are logged. The agent gets back the real tool's output, or a structured refusal explaining what's blocked and how to unblock it. A shim farm under `$PATH` provides defense-in-depth for any subprocess path where children look up tools by name.

## Status

**Design phase.** The architecture has converged after several rounds of adversarial review and a workflow-validation exercise. Implementation has not yet started. The repo currently contains design documentation only.

## Documentation

Start here, in order:

1. **[`docs/MISSION.md`](docs/MISSION.md)** — what `tap` is, the cooperative-agent threat model, the "alpha command, top dog" framing, and the YOLO-by-default product positioning.
2. **[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)** — the resolved design in implementation detail. The grammar, the compile pipeline, the shim trampoline, the evaluator algorithm, the cross-platform install matrix.
3. **[`docs/CONCEPTS.md`](docs/CONCEPTS.md)** — the conceptual insights and thinking tools that shaped the design. The single-lattice principle, the cardinal-sin-inversion, the subtraction-over-moving-target trap, the editorial-burden axis.
4. **[`docs/HISTORY.md`](docs/HISTORY.md)** — the chronological design journey. How we got from "vague idea of agent hooks" to the current architecture, through three adversarial-review rounds and several rejected strategies.
5. **[`docs/REJECTED.md`](docs/REJECTED.md)** — alternative designs we explored and the specific reason each was rejected. Read this before proposing a variant — the failure mode that killed it is documented.

## License

MIT. See [`LICENSE`](LICENSE).
