---
name: autoresearch
summary: Run Karpathy-style AutoResearch or an AutoResearch-style coding-research loop for a user-specified task.
description: Use this skill when Biomni needs to set up, supervise, document, or review an AutoResearch run for a user-specified objective. AutoResearch is a git-and-agent research loop: define a protected benchmark and editable implementation, create a task-specific program.md, run a baseline, then repeatedly edit one implementation file, run the fixed evaluation, log results, keep improvements, and revert failures. This skill covers the original karpathy/autoresearch nanochat setup and adaptations of the same pattern to other ML/scientific tasks.
---

# AUTORESEARCH for Biomni

Use this skill when a user asks Biomni to “run AutoResearch,” “set up autoresearch for this task,” “let an agent improve this model overnight,” or provides an API key and asks Biomni to oversee iterative research experiments.

AutoResearch is not a conventional optimizer API. It is a **repository-local autonomous research loop** driven by a task-specific `program.md`, a protected evaluation harness, an editable implementation file, git commits, repeated runs, and a results log. Biomni’s job is to set up the loop safely, verify it works before spending LLM/GPU budget, supervise progress honestly, and leave behind a reproducible record.

## Deliverables

Produce or update the smallest useful set of artifacts in the repository:

1. **Task-specific `program.md`**: the instructions the coding agent follows. It must include the user’s task, allowed edits, forbidden edits, metric, logging rules, keep/discard rules, crash handling, and stop/budget policy.
2. **Run directory**: `results/autoresearch/<run_tag>/` containing setup notes, logs, progress files, snapshots, and plots. Keep all Biomni-created run artifacts repository-local.
3. **Setup notes**: document the task, repo state, branch, hardware, API-provider handling, protected files, editable files, metric, baseline score, timeout, budgets, and commands used.
4. **Baseline run**: a successful first run of the unmodified implementation, with parsed metric and runtime.
5. **Experiment log**: a tab-separated `results.tsv` or `results/autoresearch/<run_tag>/results.tsv` containing every attempted experiment, including crashes and discarded ideas.
6. **Progress summaries**: periodic compact summaries of best score, current branch/commit, number of experiments, last results, cost/time used if available, and notable ideas tried.
7. **Final report**: best commit, best score, comparison against baseline, key diffs, retained changes, failed ideas, reproducibility command, and any caveats.

If the user only asks for a plan, provide concrete instructions and code snippets. If editing a repository, create the files and keep them self-contained.

## Core AutoResearch contract

For the original `karpathy/autoresearch` repository, preserve these invariants:

- Use a clean git branch named `autoresearch/<run_tag>` from the current main/master branch.
- The first experiment is always the unmodified baseline.
- Read and respect `README.md`, `prepare.py`, `train.py`, and `program.md` before launching the loop.
- `prepare.py` is protected. It contains fixed constants, data preparation, tokenizer, dataloader, and evaluation. Do not modify it.
- `train.py` is the default editable file. Architecture, optimizer, hyperparameters, batch size, model depth, attention pattern, and training-loop details are fair game, as long as the script runs correctly.
- Do not modify the evaluation harness, especially the validation metric function.
- Do not add dependencies or install new packages during candidate experiments unless the user explicitly approved adapting the framework.
- Run experiments with `uv run train.py`, redirecting logs to a file instead of streaming full output into the agent context.
- Parse the final summary line `val_bpb: <number>`. For the original benchmark, **lower `val_bpb` is better**.
- Keep candidate commits only when they improve the best valid score after accounting for the simplicity criterion and any agreed noise threshold.
- Revert candidate commits that crash, worsen the metric, fake the metric, alter protected files, or complicate the code without meaningful gain.
- Log every experiment, including crashes, before moving on.

The original project’s default training script uses a fixed 5-minute training budget, and runs should normally finish within a few extra minutes for startup/evaluation overhead. Treat a run exceeding 10 minutes as failed unless the user explicitly changed the benchmark.

## When the user specifies a task

First classify the task:

### 1. Native AutoResearch task

Use the original repository directly when the task is essentially:

- Improve the single-GPU nanochat/GPT pretraining script.
- Lower validation bits-per-byte (`val_bpb`) under the fixed time budget.
- Experiment only by editing `train.py`.

In this case, do not invent a new benchmark. Use the repository’s `prepare.py`, `train.py`, and `program.md` pattern.

### 2. AutoResearch-style adaptation

Use the AutoResearch pattern, but say explicitly that this is an **AutoResearch-style adaptation**, when the user’s task is a different ML/scientific problem, such as:

- Improve a classifier, predictor, ranking method, or scientific model.
- Optimize a training script in another repository.
- Run iterative research against a custom validation set.

For adaptations, create the same contract:

- **Protected benchmark file**: e.g. `prepare.py`, `benchmark.py`, or `evaluate.py`, containing data loading, fixed split, constants, and metric.
- **Editable implementation file**: ideally one file such as `train.py`, `method.py`, or `model.py`.
- **Fixed run command**: e.g. `uv run train.py`, `python train.py`, or `python results/autoresearch/<tag>/runner.py`.
- **Parseable metric**: one final line such as `val_bpb: ...`, `val_accuracy: ...`, `autoresearch_metric: ...`, or `score: ...`.
- **Clear direction**: lower-is-better or higher-is-better.
- **Bounded runtime**: fixed wall-clock, fixed epochs, fixed examples, or fixed budget so experiments are comparable.
- **Git keep/revert loop**: commit before running; keep only improvements; reset/discard failures.

Do not call an arbitrary task “AutoResearch” if it cannot be reduced to a repeatable code-edit/evaluate loop.

## Intake and defaults

Collect these inputs from the user when available, but do not block on missing nonessential details. Make reasonable defaults and document them.

Required or inferred:

- Task objective in one sentence.
- Repository path or whether to clone `https://github.com/karpathy/autoresearch`.
- API provider/key source for the coding agent, e.g. `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`.
- Editable file(s), default `train.py`.
- Protected file(s), default `prepare.py`, `README.md`, lockfiles, data, and evaluation code.
- Metric name and direction, default `val_bpb` lower-is-better for the original repo.
- Hardware target, default one NVIDIA GPU.
- Run tag, default `<mon><day>-<short_task>` or `<yyyy-mm-dd>-<short_task>`.
- Budget, default bounded unless the user explicitly authorized an indefinite persistent run.

Recommended default budgets:

- **Original AutoResearch**: one baseline plus as many 5-minute experiments as fit in the user’s wall-time budget.
- **Overnight run**: 8-12 hours, unless the user requests a different duration.
- **LLM cost cap**: $10 by default if the provider/API exposes usage; otherwise track calls/tokens where possible.
- **Crash guard**: stop or pause for review after 5 consecutive crashes, protected-file modification attempts, or repeated suspicious metric changes.
- **Timeout**: 10 minutes per original AutoResearch run; for adaptations, 2-3x measured baseline runtime.

## API key and secret handling

The user may provide an API key so Biomni or an external coding agent can generate edits. Handle it as a secret:

- Prefer environment variables or the platform secret store. Never write keys into `program.md`, source code, shell scripts, logs, git commits, notebooks, or results files.
- Do not echo environment variables. Avoid commands such as `env`, `printenv`, or verbose tool diagnostics that could leak secrets.
- Redact keys from any captured output before writing logs or summaries.
- The candidate implementation file, such as `train.py`, must not import LLM SDKs or call the LLM API. The API key belongs to the supervising research agent, not the model training code.
- Do not pass API keys into subprocesses unless they are needed by the coding agent itself.
- If using a third-party coding agent CLI, configure its authentication outside the repository and confirm it can operate before launching the long loop.

Example environment handling:

```bash
# Set in shell/session only; do not commit this to any file.
export OPENAI_API_KEY="..."
# or
export ANTHROPIC_API_KEY="..."
```

## Repository setup for original AutoResearch

Use this setup when starting from the original repository.

```bash
git clone https://github.com/karpathy/autoresearch.git
cd autoresearch

git status --short
python --version
command -v uv
nvidia-smi

uv sync
uv run prepare.py
uv run train.py > /tmp/autoresearch_smoke.log 2>&1
grep "^val_bpb:\|^peak_vram_mb:\|^training_seconds:\|^total_seconds:" /tmp/autoresearch_smoke.log
```

If the user already has a checkout, do not overwrite it. Inspect the current branch, uncommitted changes, remote, and commit hash first.

```bash
git rev-parse --show-toplevel
git branch --show-current
git status --short
git log -1 --oneline
```

If the working tree is dirty, preserve user changes before running experiments: commit them if they belong to the run, stash them, or create a separate copy. Never discard user work silently.

## Results directory policy

Create a repository-local run directory:

```bash
RUN_TAG="<run_tag>"
RUN_DIR="results/autoresearch/${RUN_TAG}"
mkdir -p "$RUN_DIR/logs" "$RUN_DIR/snapshots"
```

Recommended files:

```text
results/autoresearch/<run_tag>/
  setup_notes.md
  program.md.snapshot
  results.tsv
  progress.jsonl
  progress.md
  logs/
    run_000_baseline.log
    run_001_<short_commit>.log
    run_002_<short_commit>.log
  snapshots/
    baseline_train.py
    best_train.py
  plots/
    val_bpb_progress.png
```

The original `program.md` suggests `results.tsv` in the repository root. For Biomni-managed runs, prefer `results/autoresearch/<run_tag>/results.tsv` and update `program.md` to point to that exact path. If an external coding agent assumes root `results.tsv`, create a symlink or copy, but keep the authoritative log in the run directory.

Do not commit logs, result TSVs, snapshots, caches, or secrets. Add them to `.git/info/exclude` rather than changing `.gitignore` unless the user asks.

```bash
cat >> .git/info/exclude <<'EOF_EXCLUDE'
results/autoresearch/
run.log
results.tsv
EOF_EXCLUDE
```

## Branch setup

Use a fresh run branch.

```bash
RUN_TAG="<run_tag>"
git checkout master 2>/dev/null || git checkout main
git pull --ff-only || true
git checkout -b "autoresearch/${RUN_TAG}"
```

If adapting another repository, branch from the user-approved baseline branch.

Create a setup commit only if `program.md` or benchmark scaffolding is intentionally part of the experiment definition. Candidate experiment commits should usually contain only editable implementation changes.

## Task-specific `program.md`

The `program.md` file is the “research org code.” It should be concrete enough for a coding agent to run without repeatedly asking the user, but strict enough to prevent benchmark tampering.

### Required sections

- **Task**: one-sentence objective.
- **Setup**: branch, run tag, run directory, files to read, data prep check, baseline run.
- **Allowed edits**: default only `train.py`; for adaptations, list exact editable files.
- **Forbidden edits**: benchmark/evaluation/data-prep files, dependencies, lockfiles, data, hidden labels, metric printing, API keys.
- **Metric**: parseable name, direction, improvement threshold, and simplicity rule.
- **Command**: exact run command and timeout.
- **Logging**: exact TSV path and columns.
- **Loop**: propose one idea, edit, commit, run, parse, log, keep or reset.
- **Crash handling**: fix trivial implementation errors once or twice; otherwise log crash and revert.
- **Progress**: write compact summaries without flooding context.
- **Stop policy**: user stop, explicit wall-time/experiment/cost budget, safety failure, or repeated environment failure.

### Template for original AutoResearch

```markdown
# AutoResearch program: <task_name>

## Task
Lower `val_bpb` for the single-GPU AutoResearch training script under the fixed training-time budget. The user’s task focus is: <user_task_focus>.

## Setup
- Run tag: `<run_tag>`.
- Branch: `autoresearch/<run_tag>`.
- Run directory: `results/autoresearch/<run_tag>/`.
- Read `README.md`, `prepare.py`, and `train.py` before proposing changes.
- Verify data/tokenizer exist in `~/.cache/autoresearch/`; if missing, run `uv run prepare.py`.
- First run is the unmodified baseline.

## Files
You may edit:
- `train.py` only.

You must not edit:
- `prepare.py`
- `README.md`
- `pyproject.toml`
- `uv.lock`
- validation/data/tokenizer artifacts
- `results/autoresearch/<run_tag>/results.tsv` except to append experiment records
- any file containing API keys or secrets

## Metric
- Run command: `uv run train.py > results/autoresearch/<run_tag>/logs/run_<N>_<commit>.log 2>&1`.
- Parse: `grep "^val_bpb:\|^peak_vram_mb:\|^training_seconds:\|^total_seconds:" <log>`.
- Primary metric: `val_bpb`.
- Direction: lower is better.
- Keep a change only if it lowers the best valid `val_bpb` enough to justify any added complexity.
- If a change produces the same metric but meaningfully simpler code, it may be kept.
- Never fake, short-circuit, or alter the metric output.

## TSV log
Append one tab-separated row per experiment to:
`results/autoresearch/<run_tag>/results.tsv`

Columns:
`experiment	commit	val_bpb	memory_gb	training_seconds	total_seconds	status	description`

Statuses:
- `keep`: improved best score or approved simplification
- `discard`: valid run but not worth keeping
- `crash`: failed run, OOM, timeout, or no parseable metric

## Experiment loop
Repeat until the stop policy is met:
1. Record current commit as the start point.
2. Form one clear hypothesis for improving `val_bpb`.
3. Edit `train.py` only.
4. Check that protected files are unchanged: `git diff --name-only`.
5. Commit the candidate edit.
6. Run the experiment with the exact command above.
7. If the run exceeds 10 minutes, kill it, log `crash`, and reset to the start point.
8. Parse `val_bpb` and peak VRAM.
9. Append a row to the TSV.
10. If the result improves the best score and passes the simplicity criterion, keep the commit.
11. Otherwise reset back to the start point.
12. Write a compact progress update.

## Crash handling
If a candidate crashes because of a trivial typo/import/shape bug, fix it in the same experimental idea and rerun once or twice. If the idea is fundamentally broken, log it as `crash`, reset, and move on. Do not keep partial broken commits.

## Stop policy
Continue autonomously until one of these occurs:
- the user stops the run;
- the wall-time, experiment-count, or cost budget is reached;
- 5 consecutive crashes occur;
- protected files or evaluation integrity are at risk;
- the environment/GPU becomes unavailable.
```

### Template for an AutoResearch-style custom task

```markdown
# AutoResearch-style program: <task_name>

## Task
<Plain-language objective.>

## Contract
- Editable file(s): `<editable_file>`.
- Protected benchmark file(s): `<benchmark_file>`, `<data_split_file>`, `<metric_file>`.
- Run command: `<command>`.
- Metric line: `<metric_name>: <float>`.
- Direction: `<higher|lower>` is better.
- Runtime budget per experiment: `<timeout>`.
- First run: unmodified baseline.

## Data and benchmark integrity
Do not modify protected files, validation labels, split manifests, dataset downloads, metric code, or dependency files. Do not hard-code validation answers. Do not reduce the benchmark to make results look better unless the benchmark itself is being explicitly redesigned before the baseline.

## Experiment loop
Commit each candidate before running; keep only valid improvements; reset and log everything else.
```

## Baseline validation before launching the loop

Run these checks before any autonomous optimization:

1. Confirm the working tree is clean or user changes are preserved.
2. Confirm the active branch and run tag.
3. Confirm `uv sync` or equivalent environment setup completes.
4. Confirm GPU availability and enough VRAM.
5. Confirm data/tokenizer/cache exists, or run one-time preparation.
6. Confirm protected files are readable and writable only by the user, not altered by candidates.
7. Run the unmodified baseline and parse the metric.
8. Save baseline log and `train.py` snapshot.
9. Initialize the TSV with a header and a baseline row.
10. Confirm the results directory is excluded from git commits.

Example baseline commands for original AutoResearch:

```bash
RUN_TAG="<run_tag>"
RUN_DIR="results/autoresearch/${RUN_TAG}"
mkdir -p "$RUN_DIR/logs" "$RUN_DIR/snapshots"
cp train.py "$RUN_DIR/snapshots/baseline_train.py"

printf "experiment\tcommit\tval_bpb\tmemory_gb\ttraining_seconds\ttotal_seconds\tstatus\tdescription\n" > "$RUN_DIR/results.tsv"

git rev-parse --short HEAD
uv run train.py > "$RUN_DIR/logs/run_000_baseline.log" 2>&1
grep "^val_bpb:\|^peak_vram_mb:\|^training_seconds:\|^total_seconds:" "$RUN_DIR/logs/run_000_baseline.log"
```

Parse the baseline robustly. If no `val_bpb` line appears, inspect the last 50 lines of the log and fix setup before launching autonomous edits.

```bash
tail -n 50 "$RUN_DIR/logs/run_000_baseline.log"
```

## Experiment log format

Use TSV, not CSV, because experiment descriptions often contain commas.

Recommended columns:

```text
experiment	commit	val_bpb	memory_gb	training_seconds	total_seconds	status	description
```

For the original minimal format, the repository’s `program.md` uses:

```text
commit	val_bpb	memory_gb	status	description
```

Either is acceptable if `program.md` and analysis scripts agree. Prefer the extended format for Biomni-managed runs.

Rules:

- Use `0.000000` or blank metric for crashes, but status must be `crash`.
- Record peak memory in GiB as `peak_vram_mb / 1024`, rounded to 0.1.
- Record `training_seconds` and `total_seconds` when available.
- Always include a concise description of the hypothesis.
- Append before reset so failed ideas are not lost.
- Keep `results.tsv` untracked.

## Keep/discard policy

For original AutoResearch:

- Best score is the lowest valid `val_bpb`.
- Keep if `val_bpb < best_val_bpb - epsilon`.
- Default `epsilon = 0.0` for the original repo unless repeated baselines show noise.
- If repeated baseline scores vary, set `epsilon` slightly above observed noise.
- Keep equal-score changes only when they significantly simplify the code, reduce memory, or increase speed without harming `val_bpb`.
- Discard if the code is uglier/fragile and the improvement is negligible.
- Discard any candidate that changes protected files, alters metric printing, shortens evaluation, changes data splits, disables validation, or hard-codes outputs.

For higher-is-better custom tasks, reverse the comparison and document the score direction in `program.md`.

## Candidate idea policy

AutoResearch works best when each experiment tests one coherent idea. Encourage the agent to explore broadly but make diffs reviewable.

Good experiment types for the original repo:

- Model depth, width, aspect ratio, head dimension, KV heads, or grouped-query attention.
- Window pattern and context-locality changes.
- Embedding, unembedding, residual, value embedding, or normalization changes.
- Learning-rate schedule, warmup/warmdown, optimizer group settings, Muon/AdamW details.
- Batch size and gradient accumulation choices that keep runtime comparable.
- Stability fixes that reduce crashes or NaNs.
- Simplifications that preserve or improve `val_bpb`.

Risky or usually forbidden:

- Changing `prepare.py`, `TIME_BUDGET`, `EVAL_TOKENS`, tokenizer, validation shard, or `evaluate_bpb`.
- Changing output to print fake `val_bpb`.
- Adding package dependencies.
- Downloading new data during experiments.
- Creating hidden caches that leak validation labels or persist candidate-specific state.
- Large, multi-file rewrites that make keep/discard decisions hard.

## Running with an external coding agent

The original README describes using a coding agent such as Claude or Codex in the repository and pointing it at `program.md`. Biomni may either act as the coding agent itself or supervise an external coding agent if the environment supports it.

Before launching external automation:

- Confirm the coding agent is authenticated via secret environment variables or its own login flow.
- Confirm it is operating in the correct repository and branch.
- Give it only the task-specific `program.md` instructions, not the user’s raw API key.
- Restrict or monitor shell permissions where possible. If the agent supports auto-approval, allow only repository-local file edits and known experiment commands.
- Require that it logs experiments to the designated TSV and does not commit results artifacts.

Example prompt to the coding agent:

```text
Read program.md completely. Follow it exactly. Start with setup validation and the baseline run. Then begin the autonomous experiment loop. Do not edit protected files. Do not ask whether to continue between experiments. Keep only improvements and log every attempt.
```

## Biomni supervision loop

Biomni should supervise the process rather than just start a command and disappear.

At each checkpoint, record:

- Current branch and commit.
- Running process status, GPU utilization, and log file path.
- Last parsed metric and memory.
- Best metric and best commit.
- Number of experiments: kept, discarded, crashed.
- Consecutive crashes.
- Approximate elapsed time and remaining budget.
- Any suspicious behavior, such as protected-file diffs or missing metric lines.

Suggested progress cadence:

- After baseline.
- After every experiment for short runs.
- Every 15-30 minutes for long persistent runs.
- Immediately on crash streaks, protected-file modifications, GPU failure, or budget exhaustion.

A compact progress entry should look like:

```markdown
## Progress <timestamp>
- Branch: `autoresearch/<run_tag>` at `<commit>`
- Experiments: 14 total / 3 kept / 8 discarded / 3 crash
- Best: val_bpb 0.993200 at `<commit>`
- Last: discard, val_bpb 0.995100, 44.2 GiB, “increase LR to 0.06”
- Status: continuing; no protected-file diffs
```

## Persistent runs and process management

Only claim to be supervising a long run when Biomni actually has a persistent session or process manager available.

Acceptable persistent patterns:

```bash
tmux new -s "autoresearch_<run_tag>"
# run the coding agent or Biomni-controlled loop inside tmux
```

or:

```bash
nohup bash results/autoresearch/<run_tag>/launch_loop.sh \
  > results/autoresearch/<run_tag>/supervisor.out \
  2> results/autoresearch/<run_tag>/supervisor.err &
```

But a plain `nohup` is not enough unless the launched process can make code-editing decisions. A shell loop can rerun a fixed command, but it cannot invent experiments unless paired with a coding agent or Biomni process.

If Biomni does not have persistent execution, do not pretend it can continue later. Instead, run what can be run in the current session, write the setup artifacts and launch instructions, and report the exact state.

## Minimal helper scripts

For robustness, create small helper scripts in the run directory. Do not modify the benchmark to support logging.

### `parse_log.py`

```python
#!/usr/bin/env python3
import json
import re
import sys
from pathlib import Path

PATTERNS = {
    "val_bpb": r"^val_bpb:\s*([0-9.]+)",
    "peak_vram_mb": r"^peak_vram_mb:\s*([0-9.]+)",
    "training_seconds": r"^training_seconds:\s*([0-9.]+)",
    "total_seconds": r"^total_seconds:\s*([0-9.]+)",
}

def parse(path: Path) -> dict:
    out = {}
    for line in path.read_text(errors="replace").splitlines():
        for key, pattern in PATTERNS.items():
            m = re.match(pattern, line)
            if m:
                out[key] = float(m.group(1))
    out["ok"] = "val_bpb" in out
    return out

if __name__ == "__main__":
    for arg in sys.argv[1:]:
        print(json.dumps({"log": arg, **parse(Path(arg))}, sort_keys=True))
```

### `plot_progress.py`

```python
#!/usr/bin/env python3
from pathlib import Path
import pandas as pd
import matplotlib.pyplot as plt

run_dir = Path(__file__).resolve().parent
tsv = run_dir / "results.tsv"
out_dir = run_dir / "plots"
out_dir.mkdir(exist_ok=True)

df = pd.read_csv(tsv, sep="\t")
if "val_bpb" not in df.columns:
    raise SystemExit("results.tsv has no val_bpb column")
valid = df[(df["status"].isin(["keep", "discard"])) & (df["val_bpb"] > 0)].copy()
if valid.empty:
    raise SystemExit("no valid results yet")
valid["best_so_far"] = valid["val_bpb"].cummin()

plt.figure()
plt.plot(valid["experiment"], valid["val_bpb"], marker="o", linestyle="none", label="run")
plt.plot(valid["experiment"], valid["best_so_far"], marker="o", label="best so far")
plt.xlabel("Experiment")
plt.ylabel("val_bpb lower is better")
plt.title("AutoResearch progress")
plt.legend()
plt.tight_layout()
plt.savefig(out_dir / "val_bpb_progress.png", dpi=160)
```

## Validation and anti-cheating checks

Run these checks periodically:

```bash
git diff --name-only HEAD~1..HEAD
```

The diff for candidate commits should normally include only `train.py` for the original repo.

```bash
git diff -- prepare.py pyproject.toml uv.lock README.md
```

This should be empty unless the setup phase intentionally changed documentation or program instructions.

Check logs for suspicious behavior:

```bash
grep -Ei "val_bpb|evaluate_bpb|TIME_BUDGET|EVAL_TOKENS|tokenizer|validation|shard" train.py prepare.py
```

Do not keep changes that bypass evaluation, reduce validation, fake output, or alter protected constants.

## Finalization

When the run ends, produce a final report under `results/autoresearch/<run_tag>/final_report.md` with:

- Run tag, branch, base commit, best commit.
- User task and exact metric direction.
- Baseline score, best score, absolute improvement, percent improvement if meaningful.
- Number of experiments run, kept, discarded, crashed.
- Hardware and environment summary.
- Best run log path and reproduction command.
- Summary of the retained code diff.
- Table of top experiments.
- Crash/failed-idea summary.
- Caveats: metric noise, not rerun, platform-specific results, or changed hardware.

Recommended final commands:

```bash
git status --short
git log --oneline --decorate --max-count=10
git diff <base_commit>..<best_commit> -- train.py > results/autoresearch/<run_tag>/best.diff
cp train.py results/autoresearch/<run_tag>/snapshots/best_train.py
python results/autoresearch/<run_tag>/plot_progress.py || true
```

If budget allows, rerun the best commit once to verify the result, especially if improvements are small.

## Red flags to fix before or during launch

- No NVIDIA GPU is available for the original repository.
- `uv sync` fails or Python version is incompatible.
- `uv run prepare.py` was not run and data/tokenizer are missing.
- Baseline does not produce a parseable `val_bpb`.
- Candidate edits modify `prepare.py`, `evaluate_bpb`, `TIME_BUDGET`, `EVAL_TOKENS`, data shards, tokenizer, or lockfiles.
- The agent commits `results.tsv`, logs, snapshots, or secrets.
- The run command streams huge logs into the LLM context instead of redirecting to a file.
- The run exceeds 10 minutes in the original benchmark and is not killed/logged as failure.
- The agent keeps changes that worsen the metric because they “seem promising.”
- The agent keeps complex changes for tiny gains without documenting the simplicity tradeoff.
- Multiple crashes occur without reverting to a known-good commit.
- The process is launched in the background without any actual decision-making agent attached.
- Biomni claims it will continue supervising when it has no persistent execution channel.
