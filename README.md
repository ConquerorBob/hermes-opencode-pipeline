# Asymmetric agentic orchestration pipeline

This is the dev pipeline I use to keep token costs near zero on big build tasks. A paid orchestrator model handles the thinking; a free local worker model does the actual code writing. The orchestrator doesn't need to be good at writing code. It needs to plan, delegate, and read compiler output, so that's all I ask of it.

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

Worth spelling out that the arrows go both ways. Terminal and compiler output feeds back up to the orchestrator, which either sends more work down or calls the task done.

## How it's put together

**Orchestrator: GLM 5.3 Flash, via OpenRouter**

- Breaks the task into steps and decides what to delegate.
- Reads terminal feedback and keeps the whole thing moving.

**Worker: OpenCode CLI, running `openrouter/deepseek-v4.1-flash:free`**

- Runs locally as a one-shot subprocess per dispatch:

  ```bash
  opencode run --title "auth-module" \
    -m openrouter/deepseek-v4.1-flash:free \
    "Implement the JWT auth module in src/auth/ per the spec"
  ```

- Edits files, runs `cargo`/`git`/`python` directly, and chews through compiler errors on its own.

## Why it's cheap

Code diffs, test scaffolding, and refactors all come from the free tier, and that's the bulk of the output. The paid model only spends tokens on control logic, which is short.

## The compiler gate

The one rule that makes this actually work: nothing counts as done until local verification passes. `cargo check` exits 0, tests run green, or the task isn't finished.

To be honest about how that's enforced: it isn't, not automatically. It's a rule the orchestrator's prompt has to state, and I still spot-check the results myself, because a sub-agent saying "all tests pass" means exactly nothing.

It also plugs into the normal local workflow: `gh` for pushing, `cargo` for builds, `ssh` for deploys.

## Where it breaks

- OpenRouter's free tier has request caps. Firing off parallel sub-agents burns through them mid-task. Serialize the heavy dispatches and treat rate-limit errors as retryable.
- The worker can get stuck in a loop on a compiler error it can't fix, retrying the same change forever. Cap the fix attempts per dispatch and send the failing diagnostics back up instead.
- Sub-agents know nothing about the parent conversation. Every dispatch carries its own context: file paths, constraints, the spec.

## Quickstart

Prereqs:

- [Hermes Agent](https://hermes-agent.nousresearch.com/), installed with `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`
- OpenCode CLI, with an OpenRouter key set up for the free DeepSeek tier
- `gh` authenticated (only if the task pushes to GitHub)
- Rust toolchain (only for Rust tasks)

Run a build task like this:

```bash
hermes chat -q --oneshot \
  "Build a multi-module CLI utility in Rust. Delegate code construction \
   to OpenCode sub-agents (opencode run -m openrouter/deepseek-v4.1-flash:free) \
   and verify with cargo test before completion. Exit code 0 or report failure."
```

The orchestrator takes the query, splits it up, runs `opencode` subprocesses, and reads back what the compiler said.

MIT licensed, see [LICENSE](LICENSE).
