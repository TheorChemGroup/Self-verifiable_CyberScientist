# VerifiableCyberScience

A human-in-the-loop harness for scientific reasoning with Codex or Claude Code and Lean. The agent proposes a model, requests information when needed, and develops a proof. Independent checks prepare the assumptions, definitions and conclusions for the researcher's decision. Rejection returns feedback to the agent for revision.

[PROTOCOL.md](PROTOCOL.md) describes the workflow. [protocols/instructions.txt](protocols/instructions.txt) contains the scientist instructions.

## Recorded runs

[Browse the eight runs](https://theorchemgroup.github.io/Self-verifiable_CyberScientist/traces/) or download the repository and open [traces/index.html](traces/index.html). Each run has JSON and HTML containing the task, messages and tool activity, produced Lean code and feedback. Accepted submissions show `accepted`. Article references use DOI links; PDFs are not included.

The [trace format and export command](traces/README.md) describe how to generate the same views from saved run records.

## Requirements

- Python 3.11 or newer and [uv](https://docs.astral.sh/uv/).
- Docker with the Python 3.13 image identified by IMAGE in src/vcs_protocol.py available locally. The launcher uses --pull=never.
- Lean 4.35.0 and compiled Mathlib dependencies matching lake-manifest.json. Supply their checkout with --library-root.
- An authenticated native client using its subscription login:

| Client | Version | Default model | Reasoning effort |
| --- | --- | --- | --- |
| Codex CLI | 0.166.0 | gpt-6.1-sol | medium |
| Claude Code | 2.1.325 | claude-sonnet-5-5 | medium |

The launcher checks the client version and records the executable's hash, requested model, reasoning effort and library manifest. Codex requires its native codex-code-mode-host companion. Use --codex-binary or --claude-binary to select an executable explicitly.

Install the Python dependencies:

~~~sh
uv sync --group dev
~~~

## Start a task

The default question is balls in a box:

~~~sh
uv run vcs-protocol start --agent codex --run-dir artifacts/balls-codex --library-root /path/to/pinned-cache-checkout
~~~

To use Claude Code, choose --agent claude and a distinct run directory. The --model and --reasoning-effort options select the model settings. Memory defaults to 8g and CPUs to 2; --memory and --cpus apply to the scientist and its independent checker.

Choose another question with --task-file:

| Question | File |
| --- | --- |
| Balls in a box | protocols/task-balls.txt |
| Fish species and population | protocols/tasks/lake-inventory.txt |
| Salt addition when boiling water | protocols/tasks/boiling-salt.txt |
| Coffee cooling | protocols/tasks/coffee.txt |

~~~sh
uv run vcs-protocol start --agent claude --task-file protocols/tasks/coffee.txt --run-dir artifacts/coffee-claude --library-root /path/to/pinned-cache-checkout
~~~

Supply your own natural-language question and scientist instructions with --task-file and --instructions-file. Optional --initial-feedback-file adds human task clarification. The harness freezes the exact input bytes and hashes before inference. The run uses those snapshots throughout; changes to source files do not change a run's question.

## Respond and review

The run stops when the agent requests information or submits a machine-checked candidate for human review.

~~~sh
uv run vcs-protocol status artifacts/coffee-claude
uv run vcs-protocol respond artifacts/coffee-claude --request-id REQUEST_ID --response-file response.txt --source 'citation, measurement method or human clarification'
~~~

Requests can ask for literature, calculations, experiments or clarification. Supply the actual response and its provenance. No physical measurement device is connected by the harness; identify simulated observations as simulated.

Inspect the frozen attempt under control/attempt-NNN/:

- Model.lean and report.txt: the submitted proof and scientific explanation.
- candidate.json and packet.json: declared conclusions, facts, modeling choices, sources and checking provenance.
- verification/extracted.lean and verification/extracted.json: the independently compiled review statements and dependency comparison.
- verification/lean.log and checker-trace.log: checking evidence.

Record a decision for the exact binding reported by status:

~~~sh
uv run vcs-protocol review artifacts/coffee-claude --decision reject --binding HASH_FROM_STATUS --reason 'Explain the scientific issue to revise'
uv run vcs-protocol review artifacts/coffee-claude --decision accept --binding HASH_FROM_STATUS --reason 'Record the human judgment'
~~~

After supplying input or rejecting a candidate, resume explicitly:

~~~sh
uv run vcs-protocol resume artifacts/coffee-claude
~~~

The respond and review commands do not launch inference. A resume uses the recorded client and input snapshots in a fresh context with the run's work and feedback. The agent cannot accept its own result. Acceptance is bound to the frozen artifacts; changing them invalidates the decision.

## Paired task queue

Run the lake, salt and coffee questions independently with both clients. Preview the six commands without creating files or invoking a model:

~~~sh
uv run vcs-matrix --task-dir protocols/tasks --instructions-file protocols/instructions.txt --run-root artifacts/paired-tasks --library-root /path/to/pinned-cache-checkout --max-concurrent 3 --memory 4g --cpus 2 --dry-run
~~~

Remove --dry-run to launch. The queue freezes common inputs and uses separate workspaces. It allows at most three jobs, 12 GiB memory and six CPUs concurrently. Each input or review checkpoint releases its slot. The queue does not answer requests, review science or resume runs. The files matrix.json, matrix.log and logs/ retain the plan, checkpoints and process results. Authentication or quota errors block queued jobs for that client.

## Isolation and checking

Each scientist invocation has a fresh Docker home and configuration. Its run workspace and temporary directories are writable. The selected harness files, task and instructions, native executable, Lean installation and compiled libraries are read-only. The host home, global instructions, repository checkout, other runs and Docker socket are excluded. Network access supports the native client and Internet research.

The selected subscription credential file is mounted read-only for native client authentication. Credentials are excluded from scientific evidence and logs. The launcher checks the subscription authentication type; authentication and quota failures stop execution. API-key fallback and additional scientist agents are disabled.

The host freezes each submission and verifies it in a separate container without network or credentials. Lean compilation, an axiom audit and a separate leanchecker process check the proof. The extractor compiles a standalone review file and compares its kernel expressions and reached declarations with the checked submission. Pinned imported libraries form the trust base. The researcher evaluates whether the model and conclusion answer the scientific question.

The verdict proved indicates a completed proof under the declared premises. Added axioms or sorry produce proved_if, including unused additions; human acceptance retains that condition. Errors, missing outputs and timeouts do not establish a result. Raw native-client traces, checker responses, external inputs and human decisions remain under host-managed control/.

The Lean interface accepts one proof file, explicit Mathlib imports and a body of at most 24,000 UTF-8 bytes. Final declarations use theorem name : statement := proof, with binders after the colon. Executable metaprogramming, custom syntax, namespaces, native_decide, IO and unsafe code are restricted; supported Mathlib tactics remain available.

The scientist timeout is two hours per cycle and independent checking has a 3,000-second timeout. Failed checks or incomplete submissions can return feedback for up to three cycles per start or resume invocation. Claude Code has an 80-turn limit. Client failures stop for intervention.

## Tests

The controls use mock candidates and do not run scientist inference:

~~~sh
uv run pytest -q
~~~

Enable Docker, Lean and subscription-metadata controls with the pinned library cache:

~~~sh
VCS_TEST_LIBRARY_ROOT=/path/to/pinned-cache-checkout VCS_TEST_CLAUDE=1 uv run pytest -q
~~~

These controls exercise filesystem isolation, native client version and authentication metadata, prompt transport, proof acceptance and rejection, conditional axioms, extraction identity and review bindings.

The project is licensed under the [MIT license](LICENSE.md).
