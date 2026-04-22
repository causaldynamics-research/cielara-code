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

Two repos that take ~2 minutes or less to index.

### pandas

```bash
git clone --depth 50 https://github.com/pandas-dev/pandas.git
cd pandas
cielara-code enable --claude-scope=project --codex-scope=project
# … builds ~33k-node index in ~2 min

claude       # or `codex` — both see the enriched grep output
```

**Prompt that shows the hook's value** (copy-paste into your agent session):

> For `pandas/core/reshape/merge.py`, return exactly this JSON:
> `{"cluster_label":"...","flow_participation":[{"flow_name":"...","step":N,"total_steps":M}],"top_3_callees_with_confidence":[{"file":"...","symbol":"...","confidence":0.X}]}`.
> Use Grep at most 2 times. For fields you genuinely cannot determine from Grep output alone, use `null` — do not guess. No prose.

**With cielara enabled**, the agent returns:
```json
{"cluster_label":"PandasCoreAlgorithms",
 "flow_participation":[{"flow_name":"corrwith","step":115,"total_steps":131}],
 "top_3_callees_with_confidence":[
   {"file":"pandas/errors/__init__.py","symbol":"MergeError","confidence":0.9},
   {"file":"pandas/core/algorithms.py","symbol":"take","confidence":0.9},
   {"file":"pandas/core/indexes/frozen.py","symbol":"FrozenList.difference","confidence":0.9}
 ]}
```

**Without cielara** (run `cielara-code disable` first, then ask again), the agent correctly refuses those fields:
```json
{"cluster_label":null,"flow_participation":null,"top_3_callees_with_confidence":[
   {"file":"pandas/core/sorting.py","symbol":null,"confidence":null},
   {"file":"pandas/core/algorithms.py","symbol":null,"confidence":null},
   {"file":"pandas/core/common.py","symbol":null,"confidence":null}
]}
```
because cluster labels, flow positions, and per-edge confidence scores don't exist in the source code — only in the pre-computed knowledge graph that cielara ships.

### requests (small, fast to index)

```bash
git clone --depth 50 https://github.com/psf/requests.git
cd requests
cielara-code enable --claude-scope=project --codex-scope=project
claude           # or codex
```

Try:
> For `requests/sessions.py`, list its 3 most connected peers (highest-confidence callers or callees) with the specific function/method that links them. Return JSON: `{"peers":[{"file":"...","symbol":"...","confidence":0.X,"direction":"caller|callee"}]}`. Max 2 Grep calls. Null if you can't determine.

### A more realistic engineering question

In any indexed repo:
> I'm about to change the signature of `<pick-a-function>` in `<pick-a-file>`. What are the top 4 caller symbols (function/method names) across the repo? Return JSON `{"callers":[{"file":"...","symbol":"..."}]}`. Max 2 Greps. Null if you can't determine the symbol.

Without cielara, grep finds files that *import* the target but can't tell you which function inside each file actually calls it — you'd need to open each and scan. With cielara, symbol-level callers are precomputed and surfaced on the first grep.

## Verify the hook is firing

The most reliable single-command proof — run this from **inside your enabled repo**:

```bash
rm -f /tmp/cielara-hook.log
CIELARA_HOOK_DEBUG_LOG=/tmp/cielara-hook.log claude --dangerously-skip-permissions -p \
  'Use the Grep tool to search for any common symbol like "def" in any one file. Output only "ok".'
cat /tmp/cielara-hook.log
```

Passing the env inline (`VAR=val command...`) guarantees the subprocess inherits it. Using `-p` mode means `claude` exits once the turn is done, so the `cat` at the end reliably sees the log. Expected output is one JSON line:

```json
{"ts":"...","agent":"claude","sessionId":"...","toolName":"Grep","cwd":"...","hookInvoked":true,"kgApplied":true,"additionalContextLen":514,"testMarkerInjected":false}
```

If you prefer tailing a long interactive session:

```bash
# Terminal 1 — create the file first so tail -F has something to watch,
# then export and start claude/codex in the SAME shell:
touch /tmp/cielara-hook.log
export CIELARA_HOOK_DEBUG_LOG=/tmp/cielara-hook.log
claude           # or codex

# Terminal 2:
tail -F /tmp/cielara-hook.log     # -F (capital) follows the filename even if recreated
```

(Use `-F`, not `-f` — on macOS BSD `tail`, `-f` errors if the file doesn't exist yet.)

Each log line is a JSON record per hook invocation. The `agent` field tells you whether it was Claude or Codex:
```json
{"ts":"...","agent":"claude","sessionId":"abc123","toolName":"Grep","cwd":"...","hookInvoked":true,"kgApplied":true,"additionalContextLen":514,"testMarkerInjected":false}
{"ts":"...","agent":"codex","sessionId":"xyz789","toolName":"Bash","cwd":"...","hookInvoked":true,"kgApplied":true,"additionalContextLen":396,"testMarkerInjected":false}
```

### End-to-end marker proof (shows the model actually received the output)

```bash
export CIELARA_HOOK_TEST_MARKER="CIELARA-PING-$(uuidgen | head -c 8)"
echo "Will look for: $CIELARA_HOOK_TEST_MARKER"
claude --dangerously-skip-permissions -p \
  "Grep for 'DataFrame' in pandas/core/frame.py. Then output 'ok'."
grep -l "$CIELARA_HOOK_TEST_MARKER" ~/.claude/projects/*/*.jsonl | head -3
```
A match confirms Claude actually received the cielara output.

### If the log file never appears

1. **Are you inside an enabled repo?** Check `cat .claude/settings.json` — it should list a `PostToolUse` hook matching `Grep` that calls `cielara-code claude-hook`. If it's missing, re-run `cielara-code enable --claude-scope=project` from that repo.
2. **Is `/status` inside the claude session showing your repo's settings.json?** If not, you started `claude` from a directory outside the repo — `cd` into the repo first.
3. **Did you start `claude` BEFORE exporting the env var?** Claude's subprocesses inherit from the shell where claude started. Re-run after exporting.
4. **Did your grep actually happen?** The hook only fires when a Grep tool call completes. If the model chose to use Read/Bash instead, no hook runs. Explicitly instruct "use the Grep tool" in your prompt.

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
│   ├── hooks/
│   │   ├── post-merge         # refresh index on pulls/merges
│   │   ├── post-checkout      # refresh on branch switches (skips single-file checkouts)
│   │   └── post-rewrite       # refresh after rebase
│   └── cielara-code/
│       ├── kg.json            # knowledge-graph index (JSON, ~30-80 KB per repo)
│       ├── kg.pkl             # intermediate NetworkX graph (pickle)
│       └── meta.json          # build metadata
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
