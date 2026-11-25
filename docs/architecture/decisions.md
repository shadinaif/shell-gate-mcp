# Decisions

Each entry records what was decided, why, and what was rejected. New decisions are added at the end; a decision that is reversed stays here, marked as superseded, with a link to the entry that replaces it.

## 1. The layer runs on the laptop, not on the target server

*2026-11-24*

- **Decision:** the MCP server runs on the laptop, where the SSH keys already are, and runs commands on the target server over SSH.
- **Why:** the agent gets no network path to the server at all, and nothing has to be installed on the server.
- **Rejected:** running the layer on the target server, reached through an SSH tunnel from the laptop. The agent's requests would have landed on the server itself, which is not what was wanted.

## 2. MCP, not a command-line client run through Bash

*2026-11-24*

- **Decision:** the agent reaches the layer as an MCP server. A command returns a job id at once, and the agent follows the output with that id.
- **Why:** job ids give the agent the output of long commands in steps, while they run, without a single long tool call. MCP also gives typed tool schemas, and works from any MCP client, including those that have no shell.
- **Rejected:** a small CLI that the agent runs through its shell tool in the background. It would have given live output and pipes, but job ids cover the same need inside MCP.

## 3. One MCP server

*2026-11-24*

- **Decision:** one server exposes every tool.
- **Why:** one process, one whitelist, one log and one token, and shared job state for the job tools. Several servers on the same laptop with the same SSH key would add no security.
- **Rejected:** one server per group of tools. A second server is justified only by a second environment with its own key.

## 4. One `run_command` tool, with commands defined as folders

*2026-11-24*

- **Decision:** a single `run_command(cmd=..., args={...})` tool. Each command is a folder under `commands/` with a `command.yaml`.
- **Why:** adding or changing a command must never require changing the server's code.
- **Rejected:** one MCP tool per operation. It would have given each operation its own schema and its own permission rule in the client, but every new command would have meant a code change.
- **Trade-off accepted:** MCP clients' per-tool permissions now cover `run_command` as a whole. Discovery moves to `list_commands()` and `describe_command()`, and the layer's tiers are the gate.

## 5. Arguments are always named, typed and closed

*2026-11-24*

- **Decision:** every argument is named, every schema sets `additionalProperties: false` at every level, and object arguments are always typed models, never free-form.
- **Why:** a misspelled argument must fail rather than be ignored, and a free-form object would let the agent set keys the whitelist never allowed.

## 6. Validators run as separate processes and only accept or reject

*2026-11-24*

- **Decision:** `validate_args.py` runs as a subprocess on the laptop, with the arguments as JSON on standard input. Exit 0 accepts; anything else denies, with stderr as the reason. A validator never modifies arguments.
- **Why:** edits apply without a restart, a buggy validator cannot crash the server, and the log always shows exactly the arguments that ran.
- **Rejected:** importing validators into the server process; letting validators rewrite arguments.

## 7. No live server checks in validators, for now

*2026-11-24*

- **Decision:** validators decide from the arguments and local data only.
- **Why:** it keeps validators fast and simple. `lxc` already reports a missing instance when the command runs.

## 8. One validator per command, shared code in `_helpers/`

*2026-11-24*

- **Decision:** each command has its own validator. Reusable code lives in `commands/_helpers/` as an importable package.
- **Superseded in part by decision 16:** reusable code now lives in each namespace's own `_helpers/`, not in one shared folder.
- **Why:** each command's rules stay in that command's folder, while common logic is written once.

## 9. The command name reaches validators through `LAYER_COMMAND`

*2026-11-24*

- **Decision:** the layer sets `LAYER_COMMAND` in each validator's environment.
- **Rejected:** working the command out from `sys.argv`, `__main__` or the call stack. These work but are indirect; the layer already knows the command and can just say so.

## 10. Every helper keeps a checked list of the commands that use it

*2026-11-24*

- **Decision:** each helper declares a guard instance that checks `LAYER_COMMAND` against the helper's same-named YAML list and raises if the command is missing. Unit tests check that every helper declares one and that every list is exact.
- **Why:** when a helper is edited, the engineer must see exactly which commands the change affects. A comment would go out of date by mistake; a list the code checks cannot.

## 11. Fail hard, nothing silent

*2026-11-24*

- **Decision:** every failure is loud and explicit. A bookkeeping mistake that denies a command in real use is accepted as the price of never passing anything quietly.
- **Details:** [failure-handling.md](failure-handling.md).

## 12. Refuse to start if any definition is invalid

*2026-11-24*

- **Decision:** at startup the server checks every command, validator and helper list, and refuses to start if any is invalid, reporting every problem at once.
- **Rejected:** starting anyway and reporting broken commands, which would let one problem go unnoticed while everything else worked.

## 13. Every decision lives in this repository

*2026-11-24*

- **Decision:** all decisions and rules are written in `docs/architecture/`. Nothing lives only in a chat or an assistant's memory.
- **Why:** memory can be lost. The repository keeps the full record, with its history.

## 14. The engine works with any server reachable over SSH; LXD is the first setup

*2026-11-24*

- **Decision:** nothing in the engine is tied to LXD. A command runs whatever its definition says on the target server, and LXD is the first setup it is built and tested for. The docs use `lxc` in their examples for that reason.
- **Why:** the commands are data, so the same engine manages any server once its commands are written. Limiting it to LXD would add nothing and would narrow who can use it.

## 15. Command names are dotted paths that start with a namespace

*2026-11-24*

- **Decision:** a command name is a dotted path. The first part is the namespace, every part in between is a group folder, and the last part is the command: `lxd.snapshots.create` lives in `commands/lxd/snapshots/create/`. Every command has a namespace. A dot is never the first or the last character, two dots never follow each other, and `_.` is refused. Every part starts with a lowercase letter, so a command name never starts with `_`. A folder is either a command or a group, never both.
- **Why:** namespaces let command packs sit side by side without name clashes, and groups keep a large pack organised. Mapping names one-to-one to folders means the name alone says where a command lives.
- **Rejected:** names without a namespace; folders that are both a command and a group.

## 16. Each namespace has its own helpers

*2026-11-24*

- **Decision:** helpers live in `_helpers/` at the root of each namespace, such as `commands/lxd/_helpers/`, and nowhere else. A validator's `PYTHONPATH` points at its own namespace, and no namespace may use another's helpers; a unit test enforces it. Usage lists name commands by their full dotted names, all from the helper's own namespace. The guard class comes from shell-gate-mcp itself.
- **Why:** a command pack must be self-contained, so that copying its folder installs it. Full names match `LAYER_COMMAND` and the audit log, so a search finds every use.
- **Rejected:** one shared `_helpers/` for every namespace, which would make installing a pack mean merging its helpers into a shared folder where names can clash; per-namespace helpers plus a shared folder, which would break self-containment again for any pack that used it; usage lists with names relative to the namespace.
