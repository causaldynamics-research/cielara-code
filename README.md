# cielara-code

Graph-enhanced code retrieval CLI for [Claude Code](https://www.claude.com/claude-code). After every Grep tool call, `cielara-code` augments the output with a `[Related files from dependency graph]` block — symbol-level callers, callees, flow positions, cluster labels — so the model gets richer structural context for free.

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
cielara-code enable --claude-scope=project
```

That command:
1. Installs git hooks (`post-merge`, `post-checkout`, `post-rewrite`) so the index auto-refreshes on branch switches / merges
2. Writes a Claude Code `PostToolUse` hook into `<repo>/.claude/settings.json`
3. Builds the initial knowledge-graph index at `<repo>/.git/cielara-code/kg.json`

Now start a Claude Code session in this repo:

```bash
claude
```

Every time the model runs a Grep, cielara appends a block like:

```
[Related files from dependency graph]
  pandas/core/reshape/merge.py (cluster: PandasCoreAlgorithms):
    Callees: MergeError @ pandas/errors/__init__.py [0.9],
             take @ pandas/core/algorithms.py [0.9],
             FrozenList.difference @ pandas/core/indexes/frozen.py [0.9]
    Flow: corrwith (step 115/131), describe_numeric_1d (step 117/128)
```

The model sees this as additional context and can follow high-confidence callers/callees or stay within a cluster.

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
cielara-code enable --claude-scope=project
# … builds ~33k-node index in ~2 min

claude       # start a session inside pandas
```

**Prompt that shows the hook's value** (copy-paste into the Claude session):

> For `pandas/core/reshape/merge.py`, return exactly this JSON:
> `{"cluster_label":"...","flow_participation":[{"flow_name":"...","step":N,"total_steps":M}],"top_3_callees_with_confidence":[{"file":"...","symbol":"...","confidence":0.X}]}`.
> Use Grep at most 2 times. For fields you genuinely cannot determine from Grep output alone, use `null` — do not guess. No prose.

**With cielara enabled**, Claude returns:
```json
{"cluster_label":"PandasCoreAlgorithms",
 "flow_participation":[{"flow_name":"corrwith","step":115,"total_steps":131}],
 "top_3_callees_with_confidence":[
   {"file":"pandas/errors/__init__.py","symbol":"MergeError","confidence":0.9},
   {"file":"pandas/core/algorithms.py","symbol":"take","confidence":0.9},
   {"file":"pandas/core/indexes/frozen.py","symbol":"FrozenList.difference","confidence":0.9}
 ]}
```

**Without cielara** (run `cielara-code disable` first, then ask again), Claude correctly refuses those fields:
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
cielara-code enable --claude-scope=project
claude
```

Try:
> For `requests/sessions.py`, list its 3 most connected peers (highest-confidence callers or callees) with the specific function/method that links them. Return JSON: `{"peers":[{"file":"...","symbol":"...","confidence":0.X,"direction":"caller|callee"}]}`. Max 2 Grep calls. Null if you can't determine.

### A more realistic engineering question

In any indexed repo:
> I'm about to change the signature of `<pick-a-function>` in `<pick-a-file>`. What are the top 4 caller symbols (function/method names) across the repo? Return JSON `{"callers":[{"file":"...","symbol":"..."}]}`. Max 2 Greps. Null if you can't determine the symbol.

Without cielara, grep finds files that *import* the target but can't tell you which function inside each file actually calls it — you'd need to open each and scan. With cielara, symbol-level callers are precomputed and surfaced on the first grep.

## Verify the hook is firing

Set the debug log env var before starting the session:

```bash
export CIELARA_HOOK_DEBUG_LOG=/tmp/cielara-hook.log
claude
# ... do a grep in the session ...
```

In another terminal:
```bash
tail -f /tmp/cielara-hook.log
```

Each line is a JSON record per hook invocation:
```json
{"ts":"...","agent":"claude","sessionId":"abc123","toolName":"Grep","cwd":"...","hookInvoked":true,"kgApplied":true,"additionalContextLen":514,"testMarkerInjected":false}
```

For end-to-end marker proof (shows the model actually received the output):
```bash
export CIELARA_HOOK_TEST_MARKER="CIELARA-PING-$(uuidgen | head -c 8)"
echo "Will look for: $CIELARA_HOOK_TEST_MARKER"
claude -p "Grep for 'DataFrame' in pandas/core/frame.py. Then output 'ok'."
```
Then:
```bash
grep "$CIELARA_HOOK_TEST_MARKER" ~/.claude/projects/**/*.jsonl
```
A match confirms Claude received the cielara output.

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
- `--claude-scope=project|local|user|none` — where to install the Claude hook. Default `user` (global across every Claude session); `project` scopes to this repo only (teammates who clone and commit `.claude/settings.json` get it automatically).
- `--codex-scope=project|user|none` — same for Codex CLI
- `--no-build` — skip the initial index build

## What gets installed where (repo-level enable)

After `cielara-code enable --claude-scope=project` in `/path/to/repo`:

```
/path/to/repo/
├── .claude/
│   └── settings.json          # PostToolUse:Grep hook wired to cielara-code
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

Everything cielara writes lives inside `.git/` — nothing pollutes your working tree. `cielara-code disable --purge` removes it all.

## Environment variables

| Variable | Default | Role |
|---|---|---|
| `CIELARA_KG_TOPK` | 5 | Max related files per section in the appended block |
| `CIELARA_KG_MIN_CONFIDENCE` | 0.0 | Filter edges below this confidence |
| `CIELARA_HOOK_DEBUG_LOG` | unset | Write one JSONL record per agent-hook invocation |
| `CIELARA_HOOK_TEST_MARKER` | unset | Inject this string into every `additionalContext` — grep for it in Claude's transcript |
| `CIELARA_SIDECAR_BIN` | (auto) | Path to the `cielara-sidecar` binary (set by the npm shim) |

## Troubleshooting

**`cielara-code build` takes ~2 minutes on pandas-sized repos**
Expected. Initial build on ~1500 Python files / 33k graph nodes ≈ 2 min. Subsequent builds via git hooks are incremental and take seconds.

**`npm install -g cielara-code` fails with "unsupported platform"**
v0.2.x ships `darwin-arm64` only. Other platforms come in v0.3.x.

**Hook not firing in a Claude session**
Run `/status` inside the session. If `<repo>/.claude/settings.json` isn't listed under hooks, Claude Code's cwd isn't inside the repo (it walks up from cwd to find a git root + `.claude/` marker). For `--claude-scope=project`, you must start `claude` from inside the repo's tree.

**"no KG index found"** from `cielara-code query`
Run `cielara-code build` (or `cielara-code enable`, which includes it).

## Uninstall

```bash
# In each repo you enabled:
cielara-code disable --purge

# Remove the npm package:
npm uninstall -g cielara-code
```

## Release info

Current: **v0.2.3** (Apple Silicon macOS only).

Release tarballs live under [Releases](https://github.com/causaldynamics-research/cielara-code/releases). Each `cli-v<version>` tag includes:
- `cielara-code-<version>-darwin-arm64.tar.gz` — extracted binaries end up under `dist/` (TS binary + Python-sidecar binary + `PROVENANCE.json` + `LICENSE`)
- `*.sha256` — integrity verification

The npm package's `postinstall` fetches the right tarball automatically; you don't need to download manually unless you're doing offline setup.
