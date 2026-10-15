# figcraft-agent

Agent-facing wrapper around `figcraft-cli` for the KovaForge fork. This is a
KovaForge-only addition; the upstream CLI surface is unchanged.

## What it is

A small bash script (`figcraft-agent`) that exposes the most common agent
workflows as named subcommands with structured output (one JSON object per
line on stdout, errors on stderr). It resolves the right `figcraft-cli`
binary, manages the MCP server lifecycle, and writes render output to a
predictable directory.

## Why it lives in the KovaForge fork, not upstream

Upstream's `AGENTS.md` enforces a strict **clean-room** rule: no reading,
disassembling, or copying of the incumbent Figma's code, file formats, icons,
or images. Behaviour comes from public docs and black-box observation only.

A wrapper script alongside the CLI is **not** a clean-room concern. It calls
the same `figcraft-cli` that upstream ships, with no crate or surface
changes. That keeps upstream syncing a fast-forward (no rebase), and lets the
KovaForge fork add an agent-friendly surface without forking the
implementation.

## Subcommands

| Subcommand | Purpose |
|---|---|
| `render <file.figcraft> [-o out.png] [--scale N]` | Render to PNG. Default scale 2 (KovaForge export default). |
| `export <file.figcraft> --format svg\|pdf\|webp\|jpg [--out-dir DIR]` | Export layers in the chosen format. |
| `info <file.figcraft>` | Print file info (raw CLI output, parsed by caller). |
| `commands [--filter REGEX]` | List the full command registry for agent planning. |
| `mcp-start` | Start the headless MCP server in the background (stdio). |
| `mcp-attach <port>` | Probe a running figcraft app's control channel port. |
| `mcp-stop` | Stop the headless MCP server started by `mcp-start`. |

## Output

- **stdout**: one JSON object per line, machine-parseable.
- **stderr**: human-readable errors prefixed with `figcraft-agent: `.
- **File output**: `$FIGCRAFT_OUT` (default `$HOME/Pictures/figcraft`).
- **MCP state**: `$FIGCRAFT_OUT/agent-state/mcp-{pid,log}`.

## Environment

| Variable | Default | Purpose |
|---|---|---|
| `FIGCRAFT_OUT` | `$HOME/Pictures/figcraft` | Output directory and agent-state root. |
| `CARGO_TARGET_DIR` | workspace default | Override to use a per-agent target dir (see upstream AGENTS.md). |

## Conventions for agents

- **No retries on non-zero exit.** The wrapper fails fast; treat any non-zero
  exit as a blocking failure and stop the chain.
- **Absolute paths only.** Relative paths are rejected with exit code 2.
- **Read-only on input.** The wrapper never modifies a `.figcraft` file.
- **Output is for the agent, not the human.** When humans need to see results,
  the calling agent should attach the file or render an HTML summary.

## Build requirement

`figcraft-cli` must be built first. The wrapper resolves, in order:

1. `$WORKSPACE_ROOT/target/release/figcraft-cli`
2. `$WORKSPACE_ROOT/target/debug/figcraft-cli`
3. `figcraft-cli` on `PATH`

If none are found, `figcraft-agent` exits 127 with a build hint. Build it
once with:

```bash
cargo build --release -p figcraft-cli
```

## Example

```bash
# Render a Ridgeline example to the standard output dir at 2x
figcraft-agent render /home/mike/Work/KovaForge/figcraft/examples/ridgeline-app.figcraft

# Start the headless MCP server in the background (stdio)
figcraft-agent mcp-start
# {"ok":true,"reused":false,"pid":12345,"mode":"stdio","log":".../mcp.log"}

# Probe a running app's control channel port
figcraft-agent mcp-attach 47800
# {"ok":true,"port":47800}

# Stop the background server
figcraft-agent mcp-stop
```
