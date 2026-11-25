# Architecture

shell-gate-mcp sits between an AI agent and a server it manages over SSH. The agent never gets SSH access to the server: it asks this layer to run a whitelisted command, and the layer validates, logs, runs and reports.

Nothing in the engine is tied to one kind of server: a command runs whatever its definition says, on any host the layer can reach over SSH. The first setup manages an LXD host, so the examples in these docs use `lxc`.

These docs are the single record of every decision and rule. Nothing about the design lives only in someone's head, a chat, or an assistant's memory: if it is not written here, it has not been decided.

## Documents

| Document | What it covers |
|---|---|
| [topology.md](topology.md) | The three machines, how a request travels, where code lives and where it runs |
| [mcp-interface.md](mcp-interface.md) | The MCP tools the agent sees, their arguments, jobs and output |
| [commands.md](commands.md) | How a command is defined on disk: `command.yaml`, `exec`, scripts |
| [validation.md](validation.md) | The checks every request passes, argument validators, shared helpers and their usage lists |
| [failure-handling.md](failure-handling.md) | Fail hard, nothing silent: startup checks and runtime failure states |
| [decisions.md](decisions.md) | Every decision, with its reason and the alternatives that were rejected |
| [planning-tasks.md](planning-tasks.md) | The planning work left: everything proposed or still undecided |

## Ground rules

1. **The layer enforces; nothing else does.** Instructions to the agent (a skill, a prompt, a tool description) guide it but are never a security boundary. Every rule that matters is checked by the layer, so it holds even if the agent ignores every instruction.
2. **Fail hard, nothing silent.** Any invalid definition, broken validator, bookkeeping mistake or lost connection produces a loud, explicit error. Nothing is skipped, defaulted or passed quietly. See [failure-handling.md](failure-handling.md).
3. **Commands are data.** Adding or changing a command means adding or changing a folder under `commands/`. It never means changing the server's code.
4. **The agent never holds the keys.** The SSH keys stay on the laptop, and the agent can never write to the laptop's copy of `commands/`.
5. **Arguments are named, typed and closed.** Every argument has a name and a type, and anything unknown is rejected.
6. **No log, no run.** A request that cannot be written to the audit log first is refused.

## How to read these docs

What is written in the documents above is decided. Anything proposed but not yet confirmed has a task in [planning-tasks.md](planning-tasks.md), and the other documents mark it as **(open)** where it comes up.
