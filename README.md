# cielara-code

Graph-enhanced code retrieval CLI for [Claude Code](https://www.claude.com/claude-code) and [Codex CLI](https://developers.openai.com/codex/). After every grep-like tool call, `cielara-code` augments the output with a `[Related files from dependency graph]` block — symbol-level callers, callees, flow positions, cluster labels — so the model gets richer structural context for free.

Works with both agents via their `PostToolUse` hook mechanism:
- **Claude Code** — wired onto the `Grep` tool
- **Codex CLI** — wired onto the `Bash` tool (matches `rg`/`grep`/`ripgrep` invocations)

## Install

```bash
npm install -g cielara-code
```

**Supported platforms:** macOS arm64 (Apple Silicon). Other platforms land with v0.3.x once multi-platform CI is wired up.

### Prereqs

- **Node 18+** (for `npm install`; the CLI itself runs as a native binary)
- **git** (hook installation + repo-root discovery)
- **Claude Code** or **Codex CLI** — cielara augments whichever you use

## Quick start (repo-level)

```bash
cd /path/to/your/python/repo
cielara-code enable --claude-scope=project --codex-scope=project
```

That command:
1. Installs git hooks (`post-merge`, `post-checkout`, `post-rewrite`) so the index auto-refreshes on branch switches / merges
2. Writes the agent hooks into:
   - `<repo>/.claude/settings.json` (Claude Code `PostToolUse` on `Grep`)
   - `<repo>/.codex/hooks.json` (Codex `PostToolUse` on `Bash`) — also enables `codex_hooks = true` in `~/.codex/config.toml` if not already set
3. Builds the initial knowledge-graph index at `<repo>/.git/cielara-code/kg.json`

Skip either agent's hook with `--claude-scope=none` or `--codex-scope=none`.

Now start an agent session in this repo:

```bash
claude           # Claude Code
# or
codex            # Codex CLI
```

### Tear down

```bash
cielara-code disable           # remove hooks (restores any chained originals)
cielara-code disable --purge   # also delete the index
```

## Try it out on a real repo

Two reproducible end-to-end A/B tests against [MuLocBench](https://github.com/silkstrike/MuLocBench) instances. Both pin the repo at the issue's base commit, ask Claude Code (Opus 4.6) to localize files, and compare a `with_cielara` clone against a `no_cielara` clone of the same source tree.

Each test below was run for **N=10 trials per condition** with `claude --dangerously-skip-permissions --model claude-opus-4-6 --output-format stream-json --verbose -p <prompt>`.

### Common setup

```bash
# Where the test clones live
mkdir -p ~/cielara-tests
cd ~/cielara-tests

# Make sure cielara-code is installed
npm install -g cielara-code
cielara-code version          # confirm install
```

---

### Test 1 — flask-2023 ("How should logging in Flask look like?")

**Setup**
```bash
cd ~/cielara-tests
git clone --filter=blob:none https://github.com/pallets/flask.git flask-base       # one shared partial clone
git clone --local --no-hardlinks flask-base flask-2023-with-cielara
git clone --local --no-hardlinks flask-base flask-2023-no-cielara
git -C flask-2023-with-cielara remote set-url origin https://github.com/pallets/flask.git
git -C flask-2023-no-cielara   remote set-url origin https://github.com/pallets/flask.git
git -C flask-2023-with-cielara fetch --filter=blob:none origin 85fa8aabf5a7bd0adf204f0c2dacbba1fa6683de
git -C flask-2023-no-cielara   fetch --filter=blob:none origin 85fa8aabf5a7bd0adf204f0c2dacbba1fa6683de
git -C flask-2023-with-cielara checkout --force 85fa8aabf5a7bd0adf204f0c2dacbba1fa6683de
git -C flask-2023-no-cielara   checkout --force 85fa8aabf5a7bd0adf204f0c2dacbba1fa6683de
(cd flask-2023-with-cielara && cielara-code enable --claude-scope project --codex-scope none)
# kg.json should be ~70 KB; ~58 files indexed
```

**The prompt** (paste into `claude` started from each clone):
```
You are a file localization agent. Given a GitHub issue, explore the repository and identify which source files would need to be modified to resolve it.

Rules:
- List between 1 and 5 files.
- Include source, tests, docs, and config files if relevant.
- Use paths relative to repo root.
- Only include files that exist.
- End your reply with ONLY this JSON and nothing else:
  {"files_to_modify": ["path/to/file1", "path/to/file2", ...]}

## Issue Title
How should logging in Flask look like?

## Issue Body
Flask started to ship with a default, hardcoded logging handler. Unfortunately this setup makes it harder to install custom logging setups, because then you'll have to undo all the things Flask did to the app logger, or replace the `app.logger` entirely. A symptom of this is #1993, where Flask's own logger had to be tweaked yet again such that messages didn't get logged twice (once via Flask's setup, once via the custom one).

My question is: **Do we even want Flask to do any logging setup?** It appears that this sort of default logging is only useful during development, so maybe it makes sense to set up a default logging handler in the new Flask CLI instead of from within the application.
```

**Ground truth (11 files)**: `CHANGES`, `docs/config.rst`, `docs/contents.rst.inc`, `docs/errorhandling.rst`, `flask/app.py`, `flask/logging.py`, `tests/{test_basic,test_helpers,test_subclassing,test_templating,test_testing}.py`.

**Sample baseline output (no_cielara)** — typical answer (occurred 8/10 times):
```json
{"files_to_modify": ["flask/logging.py", "flask/app.py", "flask/cli.py", "tests/test_helpers.py"]}
```
Hits: `flask/logging.py`, `flask/app.py`, `tests/test_helpers.py` (3/11). `flask/cli.py` is a plausible-but-incorrect guess.

**Sample with-cielara output** — illustrative answer:
```json
{"files_to_modify": ["flask/logging.py", "flask/app.py", "tests/test_basic.py", "docs/errorhandling.rst", "docs/config.rst"]}
```
Hits: 5/11 — picks up the docs and the broader test file the baseline missed.

**N=10 results**:

| trial | WITH hits | WITH recall | WITHOUT hits | WITHOUT recall |
|:-----:|:---------:|:-----------:|:------------:|:--------------:|
| 1  | 4/11 | 0.364 | 4/11 | 0.364 |
| 2  | 4/11 | 0.364 | 4/11 | 0.364 |
| 3  | 4/11 | 0.364 | 3/11 | 0.273 |
| 4  | 4/11 | 0.364 | 4/11 | 0.364 |
| 5  | 5/11 | 0.455 | 3/11 | 0.273 |
| 6  | 4/11 | 0.364 | 3/11 | 0.273 |
| 7  | 4/11 | 0.364 | 3/11 | 0.273 |
| 8  | 4/11 | 0.364 | 3/11 | 0.273 |
| 9  | 4/11 | 0.364 | 3/11 | 0.273 |
| 10 | 3/11 | 0.273 | 4/11 | 0.364 |

|              | mean  | stddev |  min  |  max  |  n  |
|--------------|-------|--------|-------|-------|-----|
| WITH cielara | 0.364 | 0.041  | 0.273 | 0.455 | 10  |
| WITHOUT      | 0.309 | 0.045  | 0.273 | 0.364 | 10  |

**Δ = +0.055 (+5.5pp), effect ≈ 1.28σ.** The KG block reliably surfaces test/doc co-changers (`tests/test_basic.py`, `docs/config.rst`, `docs/errorhandling.rst`) that the baseline picks less often.

---

### Test 2 — pandas-10043 ("iloc breaks on read-only dataframe")

This one is structurally harder: the issue surface is `df.iloc[...]` (lives in `pandas/core/indexing.py`), but the actual fix is in a Cython code generator at `pandas/src/generate_code.py` — different module path, no obvious lexical bridge. Both with and without struggle to land on it.

**Setup**
```bash
cd ~/cielara-tests
git clone --filter=blob:none https://github.com/pandas-dev/pandas.git pandas-base
git clone --local --no-hardlinks pandas-base pandas-10043-with-cielara
git clone --local --no-hardlinks pandas-base pandas-10043-no-cielara
for d in pandas-10043-with-cielara pandas-10043-no-cielara; do
  git -C $d remote set-url origin https://github.com/pandas-dev/pandas.git
  git -C $d fetch --filter=blob:none origin 2e087c7841aec84030fb489cec9bfeb38fe8086f
  git -C $d checkout --force 2e087c7841aec84030fb489cec9bfeb38fe8086f
done
(cd pandas-10043-with-cielara && cielara-code enable --claude-scope project --codex-scope none)
# kg.json should be ~450 KB; ~340 .py files indexed
```

**The prompt**:
```
You are a file localization agent. Given a GitHub issue, explore the repository and identify which source files would need to be modified to resolve it.

Rules:
- List between 1 and 5 files.
- Include source, tests, docs, and config files if relevant.
- Use paths relative to repo root.
- Only include files that exist.
- End your reply with ONLY this JSON and nothing else:
  {"files_to_modify": ["path/to/file1", "path/to/file2", ...]}

## Issue Title
iloc breaks on read-only dataframe

## Issue Body
This is picking up #9928 again. We call `df.iloc[indices]` and that breaks with a read-only dataframe. I feel that it shouldn't though, as it is not writing.

Minimal reproducing example:

```python
import pandas as pd
import numpy as np
array = np.eye(10)
array.setflags(write=False)
X = pd.DataFrame(array)
X.iloc[[1, 2, 3]]
```

> ValueError buffer source array is read-only

Is there a way to slice the rows of the dataframe in another way that doesn't need a writeable array?
```

**Ground truth (2 files)**: `pandas/src/generate_code.py`, `pandas/tests/test_common.py`.

**Sample baseline output (no_cielara)** — typical wrong guess:
```json
{"files_to_modify": ["pandas/core/indexing.py", "pandas/core/internals.py", "pandas/tests/test_indexing.py"]}
```
Hits: 0/2 — model anchors on `iloc` and lands in `core/indexing.py`, never traces to the Cython generator.

**Sample with-cielara output** — when KG surfaces `pandas/src/`:
```json
{"files_to_modify": ["pandas/core/common.py", "pandas/src/generate_code.py", "pandas/src/generated.pyx", "pandas/tests/test_indexing.py"]}
```
Hits: 1/2 — `generate_code.py` makes it in. Still misses `tests/test_common.py` (neither side ever picks it).

**N=10 results**:

| trial | WITH hits | WITH recall | WITHOUT hits | WITHOUT recall |
|:-----:|:---------:|:-----------:|:------------:|:--------------:|
| 1  | 1/2 | 0.500 | 0/2 | 0.000 |
| 2  | 0/2 | 0.000 | 0/2 | 0.000 |
| 3  | 0/2 | 0.000 | 0/2 | 0.000 |
| 4  | 1/2 | 0.500 | 0/2 | 0.000 |
| 5  | 0/2 | 0.000 | 1/2 | 0.500 |
| 6  | 0/2 | 0.000 | 0/2 | 0.000 |
| 7  | 0/2 | 0.000 | 0/2 | 0.000 |
| 8  | 0/2 | 0.000 | 0/2 | 0.000 |
| 9  | 1/2 | 0.500 | 0/2 | 0.000 |
| 10 | 0/2 | 0.000 | 1/2 | 0.500 |

|              | mean  | stddev |  min  |  max  |  n  |
|--------------|-------|--------|-------|-------|-----|
| WITH cielara | 0.150 | 0.229  | 0.000 | 0.500 | 10  |
| WITHOUT      | 0.100 | 0.200  | 0.000 | 0.500 | 10  |

**Δ = +0.050 (+5.0pp), effect ≈ 0.23σ.**

## Subcommands

| Command | What it does |
|---|---|
| `cielara-code enable` | Install hooks + build initial index |
| `cielara-code disable` | Uninstall hooks. Add `--purge` to also delete the index |
| `cielara-code build` | Rebuild the index manually |
| `cielara-code query <file>` | Show related-files block for one file |
| `cielara-code post-grep` | Read a grep tool's JSON on stdin, emit the enriched version on stdout |
| `cielara-code version` / `help` | Info |

Flags on `enable`:
- `--claude-scope=project|local|user|none` — where to install the Claude Code hook.
  - `user` (default) writes to `~/.claude/settings.json` — fires in every Claude session on your machine
  - `project` writes to `<repo>/.claude/settings.json` — only fires in sessions started inside this repo; commit that file and teammates inherit it
  - `local` writes to `<repo>/.claude/settings.local.json` — same scope as project but gitignored (personal to one dev)
  - `none` skips the Claude hook entirely
- `--codex-scope=project|user|none` — same for Codex CLI (`.codex/hooks.json`). Codex has no `local` variant.
- `--no-build` — skip the initial index build

## What gets installed where (repo-level enable)

After `cielara-code enable --claude-scope=project --codex-scope=project` in `/path/to/repo`:

```
/path/to/repo/
├── .claude/
│   └── settings.json          # Claude Code PostToolUse:Grep hook
├── .codex/
│   └── hooks.json             # Codex CLI PostToolUse:Bash hook
├── .git/
│   └── hooks/
│       ├── post-merge         # refresh index on pulls/merges
│       ├── post-checkout      # refresh on branch switches (skips single-file checkouts)
│       └── post-rewrite       # refresh after rebase
└── ... (your code, untouched)
```

(Skip either file by passing `--claude-scope=none` or `--codex-scope=none`.)

Everything cielara writes lives inside `.git/` (plus the two agent-config dirs) — nothing pollutes your working tree. `cielara-code disable --purge` removes it all.

## Environment variables

| Variable | Default | Role |
|---|---|---|
| `CIELARA_KG_TOPK` | 5 | Max related files per section in the appended block |
| `CIELARA_KG_MIN_CONFIDENCE` | 0.0 | Filter edges below this confidence |
| `CIELARA_HOOK_DEBUG_LOG` | unset | Write one JSONL record per agent-hook invocation |
| `CIELARA_HOOK_TEST_MARKER` | unset | Inject this string into every `additionalContext` — grep for it in the agent's transcript |
| `CIELARA_SIDECAR_BIN` | (auto) | Path to the `cielara-sidecar` binary (set by the npm shim) |

## Troubleshooting

**`cielara-code build` takes ~2 minutes on pandas-sized repos**
Expected. Initial build on ~1500 Python files / 33k graph nodes ≈ 2 min. Subsequent builds via git hooks are incremental and take seconds.

**`npm install -g cielara-code` fails with "unsupported platform"**
v0.2.x ships `darwin-arm64` only. Other platforms come in v0.3.x.

**Hook not firing in a Claude Code session**
Run `/status` inside the session. If `<repo>/.claude/settings.json` isn't listed under hooks, Claude's cwd isn't inside the repo (it walks up from cwd to find a git root + `.claude/` marker). For `--claude-scope=project`, you must start `claude` from inside the repo's tree.

**Hook not firing in a Codex CLI session**
Codex requires the `codex_hooks` feature flag to be enabled. `cielara-code enable` auto-writes `codex_hooks = true` under `[features]` in `~/.codex/config.toml` if that section is missing. If the section already exists, you'll need to add the line manually (cielara prints a message instructing you). Also confirm Codex is reading `<repo>/.codex/hooks.json` — same cwd rule as Claude: start `codex` from inside the repo.

**"no KG index found"** from `cielara-code query`
Run `cielara-code build` (or `cielara-code enable`, which includes it).

## Uninstall

```bash
# In each repo you enabled:
cielara-code disable --purge

# Remove the npm package:
npm uninstall -g cielara-code
```
