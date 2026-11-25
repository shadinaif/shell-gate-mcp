# Commands

A command is a folder. The server's code never lists commands: it reads them from disk, so adding a command means adding a folder, with no code change and no restart.

A command can run anything the SSH user can run on the target server. The examples here use `lxc`, for the first setup this project manages: an LXD host.

## Names and layout

A command's name is a dotted path, such as `lxd.restart-instance` or `lxd.snapshots.create`:

- **The first part is the namespace:** the command pack the command belongs to, such as `lxd`.
- **Every part in between is a group:** a folder that organises commands inside the namespace.
- **The last part is the command itself.**
- **The name maps one-to-one to folders:** `lxd.snapshots.create` lives in `commands/lxd/snapshots/create/`.

```
commands/
└── lxd/                          ← namespace
    ├── _helpers/                 ← shared Python code for this namespace's validators, never a command
    ├── list-instances/
    │   └── command.yaml          ← lxd.list-instances
    ├── restart-instance/
    │   ├── command.yaml          ← lxd.restart-instance
    │   └── validate_args.py
    ├── apt-upgrade/
    │   ├── command.yaml          ← lxd.apt-upgrade
    │   ├── run.sh
    │   └── validate_args.py
    └── snapshots/                ← group
        └── create/
            └── command.yaml      ← lxd.snapshots.create
```

### Name rules

- **Every command has a namespace,** so a name has at least one dot. `restart-instance` on its own is refused.
- **A dot is never the first or the last character.**
- **No two dots in a row:** `..` is refused.
- **No dot as the second character when the first is `_`:** `_.` is refused.
- **Every part starts with a lowercase letter** and contains only lowercase letters, digits and hyphens. A command name therefore never starts with `_`.

Together these make the pattern `^[a-z][a-z0-9-]*(\.[a-z][a-z0-9-]*)+$`. A name that matches it also meets the three dot rules, but the loader checks each rule on its own anyway, so a bad name fails with the exact rule it broke.

### Folder rules

- **A folder is either a command or a group, never both.** A command folder holds a `command.yaml`; a namespace or group folder holds only other folders. A folder with both is refused at startup.
- **Folders starting with `_` are never commands, groups or namespaces.** They are skipped when listing commands, and the name rules make them impossible to run.
- **`_helpers/` lives only at the root of a namespace,** such as `commands/lxd/_helpers/`. A `_helpers/` anywhere else is refused at startup. See [validation.md](validation.md).

## `command.yaml`

```yaml
description: Restart an LXD instance.
tier: auto
timeout_s: 120
args:
  type: object
  additionalProperties: false
  required: [instance]
  properties:
    instance:
      type: string
      pattern: '^[a-z][a-z0-9-]{0,62}$'
      description: Instance name, as shown by lxd.list-instances.
exec: [lxc, restart, '{instance}']
```

| Field | Rule |
|---|---|
| `description` | Required. What the agent sees in `list_commands()`. |
| `tier` | Required. `auto` runs at once; `approve` waits for a human. |
| `timeout_s` | Required. After this many seconds the job is killed and ends as `timed_out`. |
| `args` | Required. A JSON Schema for the arguments, with `additionalProperties: false` at every level. The loader refuses a schema that lacks it anywhere. |
| `exec` | A single command, as a list. Exactly one of `exec` and `script` is required. |
| `script` | A script file for multi-step commands. Exactly one of `exec` and `script` is required. |

There is no `never` tier: a command that must never run simply does not exist.

Whether the script and the validator are named in the manifest (`script: run.sh`, `validate: validate_args.py`) or found by fixed file names is **(open)**.

## `exec`: one command

`exec` lists the command and its arguments as separate elements. A placeholder such as `'{instance}'` is replaced by the validated argument of that name.

- **A placeholder fills a whole element, never part of one.** `'{instance}'` is allowed, `'{instance}.backup'` is not. Anything more complex goes in a script.
- **Every placeholder must name an argument that the schema defines.** The loader checks this.

### The SSH trap

SSH does not pass a list of arguments to the server. It joins them into one string, and the server's shell parses that string again. An argument such as `web; rm -rf /` would become two commands on the server. Two guards stop this:

1. **Strict patterns in the schemas,** so dangerous characters never pass validation.
2. **Every element is quoted with `shlex.quote`** before it goes into the SSH command, in case a pattern is ever too loose.

## Scripts: several steps

A script is for commands that need more than one `lxc` call, or a bit of logic:

```bash
set -euo pipefail

lxc snapshot "$ARG_INSTANCE" "pre-upgrade-$(date +%Y%m%d-%H%M)"
lxc exec "$ARG_INSTANCE" -- apt-get update
lxc exec "$ARG_INSTANCE" -- apt-get -y upgrade
```

**How it runs.** Nothing is installed on the server. The layer reads the script on the laptop and sends it over SSH to the server's `bash` on standard input:

```
ssh <server> "ARG_INSTANCE='web' bash -s" < commands/lxd/apt-upgrade/run.sh
```

**Rules:**

- **Arguments arrive as named environment variables:** `ARG_` plus the argument's name in uppercase, so `{"instance": "web"}` becomes `ARG_INSTANCE`. The prefix stops an argument from overwriting variables like `PATH`. Values are quoted with `shlex.quote` before they go into the SSH command.
- **Inside the script, every variable is double-quoted:** `"$ARG_INSTANCE"`, never `$ARG_INSTANCE`.
- **Every script starts with `set -euo pipefail`,** so the first failed step stops it. Without it, a failed snapshot would still go on to the upgrade.
- **The script's output streams into the job's output, and its exit code becomes the job's exit code.**
- **Scripts always run with `bash`.** Other interpreters wait until one is needed.
- **The script file must be a plain file name inside the command's own folder.** A path such as `../other/run.sh` is refused.

**Trust.** A script runs with the SSH user's full rights on the server, so scripts are trusted code: written and reviewed by a human, living on the laptop where the agent cannot edit them. The agent only chooses which command to run and supplies its arguments, and those are validated first.
