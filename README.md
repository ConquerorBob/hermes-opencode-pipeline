# Asymmetric Agentic Orchestration Pipeline

A low-cost, high-speed multi-agent development architecture that pairs a high-level **Orchestrator** model with a zero-cost local **Execution Engine** for automated software engineering and local terminal management.

```
                  +-------------------------+
                  |    User / Operator      |
                  +-------------------------+
                               |
                               v
                  +-------------------------+
                  |       Orchestrator      |
                  |     (GLM 5.3 Flash)     |
                  +-------------------------+
                               |  ^
              (Sub-Agent       |  |  Terminal / compiler
               Dispatch)       v  |  feedback loop
                               |
                               v
                  +-------------------------+
                  |     Execution Engine    |
                  |  (OpenCode / DeepSeek)  |
                  +-------------------------+
                               |  ^
                               v  |
                  +-------------------------+
                  |    Local System / CLI   |
                  |  (Cargo, Git, Tests)    |
                  +-------------------------+
```

The architecture is not a one-way pipeline: terminal and compiler output flow back up to the Orchestrator, which evaluates the result and either re-dispatches the Execution layer or reports completion.

## Architecture Overview

The pipeline uses a two-tier hierarchy (a hierarchical multi-agent topology) to maximize compute efficiency and minimize token expenditure:

1. **Orchestrator Layer (`GLM 5.3 Flash` via OpenRouter)**
   - Manages top-level task decomposition and workflow logic.
   - Generates minimal tool calls to delegate work to sub-agents.
   - Evaluates terminal feedback and coordinates multi-step objectives.

2. **Execution Layer (`OpenCode` CLI running `openrouter/deepseek-v4.1-flash:free`)**
   - Runs locally in sub-processes to execute heavy code generation and file edits.
   - Invoked as one-shot runs, e.g.:
     ```bash
     opencode run --title "auth-module" \
       -m openrouter/deepseek-v4.1-flash:free \
       "Implement the JWT auth module in src/auth/ per the spec"
     ```
   - Interacts directly with local developer tools (`cargo`, `git`, `python`).
   - Handles compiler diagnostic loops, fixing syntax and type errors automatically.

## Key Performance Characteristics

- **Cost Efficiency:** High-volume output (code diffs, test generation, refactoring) is offloaded to the OpenRouter free worker tier, while the paid orchestrator model handles low-token control logic.
- **Compiler-Gated Validation:** Tasks are expected to complete only when local verification passes (e.g., `cargo check` returns exit code `0`, unit tests pass). Note: this gate is a **prompt discipline**, not an enforced mechanism — the orchestrator's instructions must require it, and the operator should spot-check completions. Nothing in the pipeline enforces it automatically.
- **Local Tool Automation:** Integrates with local terminal workflows and environment credentials (`gh`, `cargo`, `ssh`) for automated project scaffolding and version control pushing.

## Constraints & Failure Modes

- **Free-tier rate limits.** The worker model (`deepseek-v4.1-flash:free`) is subject to OpenRouter's free-tier request-per-minute and daily caps. Parallel dispatch of many sub-agents can exhaust the cap mid-task; the orchestrator should serialize heavy dispatches and treat rate-limit errors as retryable.
- **Compiler error loops.** The Execution layer can spin on a compiler error it cannot fix (retrying the same failing change). Cap the number of fix attempts per dispatch; if the cap is hit, surface the failing diagnostics back to the orchestrator rather than retrying indefinitely.
- **Worker output is unverified.** Sub-agent self-reports ("done", "all tests pass") are not ground truth. Treat the compiler/test exit code — not the agent's claim — as the completion signal.
- **Context isolation.** Sub-agents know nothing about the parent conversation. Every dispatch must carry all needed context (file paths, constraints, spec) explicitly.

## Quickstart

### Prerequisites

- **Hermes Agent:** installed (`curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`).
- **OpenCode CLI:** installed locally, with an OpenRouter API key configured for the free DeepSeek tier.
- **GitHub CLI (`gh`):** installed and authenticated (only needed for push/version-control tasks).
- **Rust Toolchain:** `rustc` and `cargo` installed (only needed for Rust tasks).

### Basic Usage

To run a structured build task from the terminal:

```bash
hermes chat -q --oneshot \
  "Build a multi-module CLI utility in Rust. Delegate code construction \
   to OpenCode sub-agents (opencode run -m openrouter/deepseek-v4.1-flash:free) \
   and verify with cargo test before completion. Exit code 0 or report failure."
```

The orchestrator reads this as a one-shot query, decomposes the work, dispatches `opencode run` sub-processes, and evaluates the returned compiler/test output.

## License

MIT
