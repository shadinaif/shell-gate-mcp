# Planning tasks

The planning work left before implementation starts. A task is done when its decisions are written into the architecture docs and recorded in [decisions.md](decisions.md); then its box is ticked. Where another document marks something as **(open)**, the task for it is here.

## Security and access

### [ ] Approval flow

- How a human approves an `approve`-tier command: a prompt in a terminal on the laptop, a Telegram message, or something else.
- What the agent sees while a job is `pending_approval`, and whether it can do other work meanwhile.
- Whether an approval request expires, and what a rejection or an expiry looks like to the agent, including any new job states.
- What the audit log records about each approval: who decided, when, and through which channel.

### [ ] SSH identity on the target server

- A dedicated SSH key for the layer, separate from the laptop owner's own key.
- Which user on the target server the layer logs in as, and what that user can do. On an LXD host, membership in the `lxd` group is close to root.
- Whether to narrow it further on the server side, for example, on an LXD host, by limiting the user to one LXD project.

### [ ] Commands that run code inside an LXD instance

- `lxc exec` is arbitrary code execution inside the instance. Whether such commands exist at all, and if so, how narrowly they are limited.
- Depends on: **First set of commands**.

### [ ] MCP transport and authentication

- The port the MCP server listens on.
- How the bearer token is created, where it is stored in the VM, and how it is rotated.
- The firewall rule that admits only the VM's address, written down as a setup step.

## Commands

### [ ] First set of commands

- The operations the agent should have, each with its tier. For example: list instances, show info and read logs; start, stop and restart; create and restore snapshots.

### [ ] How script and validator files are found

- Named in `command.yaml` (`script: run.sh`, `validate: validate_args.py`), or found by fixed file names. Whichever is chosen applies to both.

### [ ] The helper guard's class name

- `AllowedCommands` or `UsedBy`.

### [ ] Command packs (namespaces)

A command pack is a named, shareable set of commands, such as `lxd`, `docker` or `systemd`. Its name is the namespace of its commands: `run_command(cmd="lxd.restart-instance", args={...})`. The LXD commands become the first pack. Names and the folder layout are decided in [commands.md](commands.md), and helpers in [validation.md](validation.md). To decide:

- **Pack metadata.** Whether a pack has its own file (name, version, description, author), and what it holds.
- **Distribution.** How a pack is published and installed: a git repository, a Python package, or a copied folder. How versions are pinned.
- **Trust.** Installing a pack means trusting its author, since its scripts run with the SSH user's full rights on the target server. How a pack is reviewed before it reaches the laptop, and whether the audit log records the pack's version next to the manifest hash.
- **Local overrides.** Whether the person running the server can change a pack's tiers or turn commands off without editing the pack, and whether overrides may only make a command stricter.
- **Conflicts.** Two packs with the same name are refused at startup; anything else that must be refused.
- **Target servers.** Whether a pack, or an installed copy of it, is tied to one target server. This is where managing several servers would be decided.
- **Startup checks.** What [failure-handling.md](failure-handling.md) must add for packs.

## Interface

### [ ] Reading part of a long output

- Separate tools such as `tail_output` and `search_output`, or arguments on `read_output`.

### [ ] Quick commands returning already finished

- Proposed: `run_command` waits a few seconds before returning, so a quick command comes back finished, with its output. How long it waits.

### [ ] Default limits

- The validator timeout (5 seconds proposed), how long `wait_job` blocks by default, and the most lines one page of output can hold.

### [ ] Checking SSH at startup

- Whether the server also refuses to start when the target server cannot be reached.

### [ ] A skill bundled with the server

- Proposed: a skill, packaged as a plugin for clients that support one (such as Claude Code), that ships the connection settings and teaches the agent the conventions, such as which command to use for what, snapshotting before an upgrade, and when to stop and ask. The skill guides; it never enforces anything.

## Operations and engineering

### [ ] Job and log storage

- Where the job folders and the audit log live on the laptop, how long they are kept, and how old jobs are cleaned up.

### [ ] Observing the agent

- How a human reads the audit log and the job folders day to day: plain files, a small viewer, or a report.
- Which patterns deserve attention, such as repeated denials, which show the agent trying something the whitelist does not allow.

### [ ] Test strategy

- Unit tests for the loader, the checks, validators and helpers, including the usage-list tests already decided in [validation.md](validation.md).
- How the layer is tested without a real target server, for example with a fake SSH runner.
- Scenario tests against a throwaway target server (for the LXD setup, a throwaway LXD inside a VM) that give the agent a task and check the end state.

### [ ] Project setup

- The Python version, the MCP SDK, the dependency tool, the test runner, and the lint configuration, with a line width of 120.
