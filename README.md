# Token-Saver

[![CI](https://github.com/ppgranger/token-saver/actions/workflows/ci.yml/badge.svg)](https://github.com/ppgranger/token-saver/actions/workflows/ci.yml)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://python.org)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)
[![Avg Savings](docs/assets/badge-savings.svg)](docs/processors/)

**Keep the signal. Stop rereading the same failures.**

Token-Saver compresses the terminal output your AI coding assistant reads —
`git diff`, `pytest`, `npm install`, `terraform plan`, `kubectl`, `docker` — to
leave more room for the code, instructions, and next decision. Compression runs
locally, without another model call or an output upload.

**New in v3: [Delta](#delta-changes-between-runs).** Run your tests,
edit, run again: see which failures are new, changed, or unchanged. Fresh
diagnostics keep their full details; repeated ones can become a short inventory,
with details available on demand. Delta is an experimental, opt-in feature in
3.0.0 for supported pytest and Ruff commands in Claude Code.

**36 specialized processors** cover git, pytest, jest, cargo, go, docker,
kubernetes, terraform, pulumi, helm, ansible, aws, gcloud, and more. Their rules
reduce progress logs and boilerplate, with failure fixtures and optional quality
contracts checking important diagnostics.

Ordinary compression supports **Claude Code** and **Antigravity CLI**. It runs
automatically after installation. Results depend on the command and output;
already concise or unsupported output may stay unchanged.

**Why developers use Token-Saver:**

- 💸 **Less output to process** — many benchmark scenarios save 60-99% of estimated output tokens; billing impact depends on your usage and provider.
- 🔁 **See what changed** — opt-in Delta compares repeated pytest/Ruff runs and keeps retained details one command away.
- 🪟 **Bigger effective context** — fit more real work into the same context window.
- 🎯 **Tested preservation** — failure fixtures check important diagnostics; add quality contracts for your own workflows.
- 🔌 **Install once, forget it** — works automatically in the background, no prompts to change.
- 🛡️ **Local compression** — pure regex/parsing with no network calls in the compression path.

---

## Table of Contents

- [Before & After](#before--after)
- [Quick Start](#quick-start)
- [Why Token-Saver Exists](#why-token-saver-exists)
- [Who It's For](#who-its-for)
- [What Gets Compressed — Real Examples](#what-gets-compressed--real-examples)
- [How It Compares](#how-it-compares)
- [How It Works](#how-it-works)
- [Precision Guarantees](#precision-guarantees)
- [Installation](#installation)
- [CLI Reference](#cli-reference)
- [Delta: Changes Between Runs](#delta-changes-between-runs)
- [Compression Quality Gates](#compression-quality-gates)
- [Processors](#processors)
- [Configuration](#configuration)
- [Tuning Recipes](#tuning-recipes)
- [Custom Processors](#custom-processors)
- [Savings Tracking](#savings-tracking)
- [Security & Privacy](#security--privacy)
- [Performance](#performance)
- [FAQ](#faq)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)
- [Project Structure](#project-structure)
- [Tests](#tests)
- [Contributing](#contributing)

---

## Before & After

| Command | Raw Output | Compressed | Savings |
|---------|-----------|------------|---------|
| `git diff` (5 files, 20 context lines each) | 2,270 tokens | 546 tokens | **76%** |
| `git diff --stat` (25 files) | 526 tokens | 21 tokens | **96%** |
| `git log --oneline` (50 entries) | 568 tokens | 117 tokens | **79%** |
| `pytest` (500 tests, 2 failures) | 6,744 tokens | 307 tokens | **95%** |
| `pytest` (200 passed, 30 warnings) | 3,543 tokens | 80 tokens | **98%** |
| `jest` (50 suites, all passing) | 570 tokens | 32 tokens | **94%** |
| `npm install` (220 packages) | 3,843 tokens | 4 tokens | **99.9%** |
| `pip install -r requirements.txt` (30 packages) | 1,797 tokens | 88 tokens | **95%** |
| `cargo build` (120 crates) | 934 tokens | 21 tokens | **98%** |
| `docker build` (20 steps) | 1,682 tokens | 207 tokens | **88%** |
| `curl` download (100 progress lines) | 2,122 tokens | 0 tokens | **100%** |
| `ruff check` (110 violations, 4 rules) | 1,475 tokens | 186 tokens | **87%** |
| `eslint` (55 violations, 2 rules) | 1,292 tokens | 144 tokens | **89%** |
| `tree` (350+ lines) | 1,840 tokens | 136 tokens | **93%** |
| `find . -name '*.py'` (205 results) | 1,294 tokens | 217 tokens | **75%** |

These are locked to [`tests/compression_baselines.json`](tests/compression_baselines.json) and gated by CI ([`tests/test_compression_ratchet.py`](tests/test_compression_ratchet.py)) — if a change makes any of them compress worse, the build fails. Run `python3 scripts/audit_compression.py` to see the full scenario set, or `token-saver benchmark '<command>'` to measure savings on your own workloads.

Not every command compresses, and that's deliberate. `cat large_file.py`, `git status -s`, and `tsc` with 7 type errors all sit in the same baseline file at **0%** — they're already dense, or the model needs them verbatim. Token-Saver returns the original bytes rather than shaving a few tokens at the cost of correctness.

## Quick Start

```
/plugin marketplace add ppgranger/token-saver
/plugin install token-saver
```

(Also available on Anthropic's official community marketplace — see [Installation](#installation) for both options.)

That's the whole setup. Ask Claude to run anything — *"run the tests"*, *"show me the diff"*, *"what changed in the last 10 commits"* — and the output is compressed before it reaches the model. Then:

```bash
token-saver stats
```

Want to see it work before installing it?

```bash
git clone https://github.com/ppgranger/token-saver.git && cd token-saver
python3 bin/token-saver benchmark 'git log --oneline -50'
python3 bin/token-saver benchmark 'git diff' --show-removed
python3 bin/token-saver explain 'docker compose logs | grep error'
```

## Why Token-Saver Exists

Verbose tool output competes with your code and instructions for context. A
`pytest` run with 200 passing tests or an install with hundreds of progress
lines often repeats information that can be represented much more compactly.
After an edit, another run can repeat the same failures too.

Token-Saver reduces that repetition using local parsing. Delta extends this to
consecutive runs, while keeping a current diagnostic inventory and retrievable
details. Whether this improves task completion, response time, or billing needs
measurement in the agent workflow that uses it.

There are three common ways to attack this:

1. **Summarize with another LLM.** Accurate-ish, but it costs a second inference per command, adds seconds of latency, and is non-deterministic — the same `pytest` run can be summarized two different ways.
2. **Truncate blindly.** Free and instant, but it's exactly how you lose the one stack-trace line that mattered.
3. **Parse the format you already know.** `git diff` has a grammar. `pytest` has a summary line. `npm install` has a progress phase and a result phase. Format-aware parsing can remove repetitive output while retaining tested diagnostics, deterministically and without another model call.

Token-Saver is the third approach, applied to 36 command families. It sits
between the CLI and your AI assistant, applying format-specific rules whose
preservation behavior you can test against your own captured output.

## Who It's For

- **Developers paying per token** on Claude Code, whether via API credits or a plan with usage limits. Tool output is often the largest single category of input tokens in an agentic session.
- **Teams running agents in CI**, where a long pipeline command (`terraform plan`, `pytest`, `gradle build`) is piped straight into a model.
- **Anyone hitting context limits** mid-task — the "conversation is too long" wall usually arrives right after a big `npm install` or a wide `git diff`.
- **Monorepo and infrastructure work**, where a single `terraform plan` or `kubectl describe` is thousands of tokens of mostly-unchanged resource attributes.
- **Privacy-sensitive environments** that can't route output through a second model or a hosted service. Token-Saver is pure local Python with no network calls in the compression path.

It's probably **not** for you if your agent mostly reads and writes source files and rarely runs shell commands — file reads go through Claude Code's own Read tool, which Token-Saver doesn't intercept.

## What Gets Compressed — Real Examples

Everything below is actual Token-Saver output, produced by the same scenario corpus the CI ratchet uses.

### `pytest` — 533 lines → 26 lines (95%)

Passing tests collapse to a count. Every failure keeps its full traceback, assertion diff, file, and line number.

```
[500 tests passed]
=================================== FAILURES ===================================
__________________________ test_compression_ratio ______________________________

    def test_compression_ratio():
        engine = CompressionEngine()
        result = engine.compress('git status', large_output)
>       assert len(result[0]) < len(large_output) * 0.5
E       AssertionError: assert 1500 < 1000
...
FAILED tests/test_engine.py::test_compression_ratio - AssertionError: assert 1500 < 1000
FAILED tests/test_processors.py::test_diff_context_trim - AssertionError: assert 42 < 10
========================= 2 failed, 510 passed ==
```

### `ruff check` — 111 lines → 18 lines (87%)

Violations are grouped by rule with a count and a couple of concrete examples, so the model learns the *pattern* to fix instead of reading 45 near-identical lines.

```
110 issues across 4 rules:
  E501: 45 occurrences in 45 files
    src/module/e501_00.py:10:5: E501 Line too long
    src/module/e501_01.py:11:6: E501 Line too long
    ... (43 more)
  F401: 30 occurrences in 30 files
    src/module/f401_00.py:10:5: F401 imported but unused
    ... (28 more)
Found 110 errors.
```

### `npm install` — 526 lines → 1 line (99.9%)

```
Build succeeded.
```

Nothing actionable was lost: the raw output was 220 `added <pkg>@<ver>` lines and a progress bar. Had the install failed, the error and the failing package would survive verbatim — see [Precision Guarantees](#precision-guarantees).

### `cargo build` — 122 lines → 2 lines (98%)

```
[121 crates compiled]
    Finished dev [unoptimized + debuginfo] target(s) in 45.23s
```

### `git diff` — context trimmed, changes untouched (76%)

Unified diffs keep every `+`/`-` line and every `@@` hunk header, with surrounding unchanged context trimmed to `max_diff_context_lines` (default 3):

```
diff --git a/src/module_0.py b/src/module_0.py
@@ -10,30 +10,32 @@ def some_function_0():
     # This is context line 9 that hasn't changed and takes up tokens
-    old_value = compute_something(0)
-    return old_value
+    new_value = compute_something_better(0)
+    cached = cache.get(new_value)
+    if cached:
+        return cached
+    return new_value
     # This is trailing context line 0 that is unchanged and wastes tokens
```

### `tree` — 312 lines → 28 lines (93%)

The directory shape is preserved down to the truncation point, with an explicit marker and the original totals, so the model knows what it *isn't* seeing:

```
.
├── src/
│   ├── engine.py
│   ├── config.py
│   ├── processors/
│   │   ├── git.py
...
... (287 lines truncated)
20 directories, 310 files
```

### `env` / `printenv` — secrets redacted before the model sees them

Variables whose names look sensitive (`SECRET`, `PASSWORD`, `CREDENTIAL`, `API_KEY`, `GITHUB_TOKEN`, and bare `KEY`/`TOKEN`/`AUTH`/`PWD` at letter boundaries — so `PATH`, `AUTHOR`, and `MONKEY` are left alone) have their values replaced. Token-Saver returns this redacted output **even when it isn't smaller** than the input: a redacted result is never traded back for the raw one to satisfy a compression threshold.

## How It Compares

| | Token-Saver | LLM summarizer | Blind truncation | Response caching |
|---|---|---|---|---|
| Extra inference cost | None | One call per command | None | None |
| Additional work | Python startup + parsing | Another model call | Truncation | Cache lookup |
| Deterministic | Yes | No | Yes | Yes |
| Diagnostic preservation | Tested fixtures + configurable quality contracts | Depends on prompt/model | No semantic checks | Depends on cached content |
| Works offline | Yes | Needs a model | Yes | Usually not |
| Understands `git diff` | Yes | Sort of | No | N/A |

For a sourced comparison with RTK and Context Mode, including integration caveats, see the [full comparison](docs/comparison.md).

## How It Works

### Architecture

**Built around explicit SOLID boundaries.** Processors parse output, a shared
policy decides command eligibility, adapters execute commands, and dedicated
modules own storage and presentation. Delta comparison and statistics formatting
are pure functions; engine dependencies and telemetry writers can be supplied
independently. The [architecture guide](docs/architecture.md) documents the
contracts, compatibility guarantees, tests, and remaining couplings.

```
CLI command  -->  Specialized processor  -->  Compressed output
                        |
                  36 processors
                  (git, test, cargo, go, build,
                   lint, package_list, python_install,
                   maven_gradle, bun, network, docker,
                   kubectl, terraform, pulumi, cdktf,
                   nix, mise, env, search, system_info,
                   gh, db_query, cloud_cli, ansible,
                   helm, syslog, ssh, jq_yq, just, act,
                   structured_log, file_listing,
                   file_content, generic)
```

The engine (`CompressionEngine`) maintains a priority-ordered chain of processors.
The first processor that can handle the command (`can_handle()`) produces the
compressed output. `GenericProcessor` serves as a fallback and always matches last.

When a specialized processor doesn't achieve the minimum compression ratio,
the engine tries the generic processor as a fallback before returning uncompressed output.

After the specialized processor runs, a lightweight cleanup pass (`clean()`)
strips residual ANSI codes and collapses consecutive blank lines.

### The Compression Pipeline, Step by Step

1. **Eligibility.** The hook first decides whether the command is safe to wrap at all — see [Security & Privacy](#security--privacy). Redirections, `sudo`, editors, unquoted newlines, background `&`, and shell-construct fragments are all left alone.
2. **Size gates.** Output shorter than `min_input_length` is returned untouched. Output longer than `max_output_bytes` (default 10 MB) is hard-capped with an explicit note before compression, so a pathological command can't be held in memory in full.
3. **Routing.** Processors are tried in priority order; the first `can_handle()` match wins.
4. **Failure routing.** If the command exited non-zero and the matched processor hasn't opted into `handles_failure`, the engine deliberately routes to the generic processor instead. A specialized processor's heuristics are tuned for the *success* shape of a tool's output; failure text often isn't recognized, and an unrecognized line is exactly what a loop drops silently.
5. **Chaining.** A processor may declare `chain_to` to hand its result to a second one, bounded by `max_chain_depth`.
6. **Cleanup.** ANSI escape stripping and blank-line collapsing.
7. **Critical-line recovery.** The engine diffs the compressed output against the original for error-shaped lines that vanished, and re-appends up to `recover_critical_lines` of them under an explicit `[token-saver] N error line(s) recovered` marker. This is a backstop covering all 36 processors — and every future one — rather than an audit that has to be repeated per processor.
8. **Ratio gate.** If the result isn't meaningfully smaller, the original is returned. **Exception:** if the processor redacted secrets, the redacted output always wins, regardless of size.
9. **Tracking.** Sizes — never content — go to a local SQLite database.

### Platform Integration

The two platforms use different mechanisms:

**Claude Code** (PreToolUse hook):

```
1. Claude wants to run `git status`
2. PreToolUse hook intercepts the command
3. Rewrites to: python3 wrap.py 'git status'
4. wrap.py executes the original command
5. Compresses the output
6. Claude receives the compressed version
```

Claude Code's PreToolUse hook cannot modify output after execution.
The only way to reduce tokens is to rewrite the command to go through a wrapper
that executes, compresses, and returns the result.

Because that rewrite is a source-to-source shell transformation, the rewritten
command is syntax-checked (`sh -n -c`) before it runs. If the rewrite wouldn't
parse, the **original** command runs uncompressed — a missed saving, never a
broken command.

**Antigravity CLI** (AfterTool hook):

```
1. Antigravity executes the command
2. AfterTool hook receives the raw output
3. Compresses the output
4. Replaces it via {"decision": "deny", "reason": "<compressed output>"}
```

Antigravity CLI allows direct output replacement through the deny/reason mechanism.

### Chained Commands

`git add -A && git commit -m wip && git push` is a single Bash tool call, but three
commands with three different output shapes. Token-Saver splits the chain, validates
each segment independently, injects unique boundary markers, and compresses each
segment with its own processor — so the `git push` output is handled by the git
processor even though it shares a shell with two other commands. If *any* segment is
ineligible, the whole chain runs untouched.

## Precision Guarantees

Compression is aggressive on noise, conservative on signal:

- Tiny outputs are never touched, and a result that doesn't actually get smaller is discarded in favor of the original (both thresholds are configurable — see [Configuration](#configuration))
- Failure fixtures verify specific errors, tracebacks and changed lines; grouping and bounded recovery can omit other details
- Source code files (`cat *.py`, `cat *.ts`, ...) pass through **unchanged** — the model needs exact content
- Secrets in `.env.production`, `.env.local`, and other `.env.*` variants are automatically **redacted** before reaching the model (`.env`, `.env.example`, and `.env.template` pass through unchanged, by design — see the note below)
- Processors remove or group progress bars, passing tests, installation logs, ANSI codes and other repetitive content according to their documented rules
- Generic truncation includes an omission marker; format-specific processors can also summarize or omit lines
- 1300+ tests including precision-specific tests that verify every critical piece of data survives compression, and a compression-ratchet suite that fails CI if a real-world scenario's compression ratio regresses

Two of these are enforced structurally rather than by convention:

- **`TestFailureHandling`** runs a realistic *failing*-command fixture through **every** processor, at **every** exit code (`0`, `1`, and unknown), and asserts the failure reason is still present. A processor that claims `handles_failure = True` has to earn it; one that doesn't gets routed around.
- **Critical-line recovery** (step 7 above) restores recognized error lines within a configurable cap, except after redaction where restoring raw lines could expose secrets.

These are protections on tested output formats, not a universal lossless guarantee.
Use [quality gates](#compression-quality-gates) to assert the exact diagnostics
your workflow requires.

> **Note on `.env`:** `.env`, `.env.example`, and `.env.template` are intentionally left untouched (not redacted, not compressed) — these are the files you're most likely actively editing, where exact values matter. Other `.env.*` variants (`.env.production`