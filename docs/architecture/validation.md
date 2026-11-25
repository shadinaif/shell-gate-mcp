# Validation

## The checks, in order

Every `run_command(cmd=..., args={...})` passes these checks in this order. The first failure denies the request, with its reason, and the denial is logged.

1. **Name.** `cmd` follows every name rule in [commands.md](commands.md), checked one by one so a refusal names the rule that was broken.
2. **Folder.** The name maps to its folder, `lxd.snapshots.create` to `commands/lxd/snapshots/create/`, which must exist and hold a `command.yaml`. A valid name contains no `/` and no part made of dots, so the path can never leave `commands/`: tricks like `cmd="../../etc"` fail the name check.
3. **Manifest.** `command.yaml` loads and is valid, as described in [commands.md](commands.md).
4. **Schema.** `args` is validated against the manifest's JSON Schema.
5. **Validator.** The command's `validate_args.py`, if it has one, accepts the arguments.
6. **Tier.** An `approve`-tier command waits for approval.
7. **Log, then run.** The request is written to the audit log, and only then does the job start.

## Argument validators: `validate_args.py`

A JSON Schema checks types and patterns. Rules that need real logic, such as "any instance except `db-primary`", go in a Python validator instead of a filtering language in YAML.

```python
"""Refuse upgrade targets that the argument schema cannot rule out."""
import json
import sys

from _helpers.instances import is_protected


def main():
    args = json.load(sys.stdin)
    if is_protected(args['instance']):
        print(f"instance '{args['instance']}' is protected and cannot be upgraded", file=sys.stderr)
        return 1
    return 0


if __name__ == '__main__':
    sys.exit(main())
```

### The contract

- **Optional, and one per command.** Each command has its own validator; validators are not shared. Shared logic goes in the namespace's `_helpers/`.
- **Runs after the schema check,** so it always receives well-formed arguments and checks only what the schema cannot express.
- **Runs as a separate process on the laptop,** never imported into the server. Each call runs the current file, so edits apply on the next call without a restart, and a buggy validator cannot crash the server or leave state behind.
- **Input:** the arguments as JSON on standard input.
- **Output:** exit code 0 accepts. Any other exit code denies, and what the validator printed to stderr becomes the denial reason that the agent sees and the log records.
- **Timeout.** A validator that runs too long is killed, and the request is denied. The limit is **(open)**; 5 seconds is proposed.
- **Fail closed.** A crash, a timeout, or a non-zero exit all deny the request. A non-zero exit with nothing on stderr is denied with `validator exited <code> without giving a reason`, pointing at the validator.
- **Accept or reject, never modify.** A validator cannot change the arguments. The arguments the agent sent are the ones that are validated, logged and run.
- **No live checks on the server, for now.** Validators decide from the arguments and from local data only. They do not query the target server. The command itself reports what is missing when it runs: for example, `lxc` reports a missing instance.

### What a validator gets

- **`LAYER_COMMAND`** in its environment: the full name of the command being validated, for example `lxd.apt-upgrade`. Code that needs to know the command reads `os.environ['LAYER_COMMAND']`, never works it out from file paths.
- **`PYTHONPATH` pointing at its own namespace's folder,** such as `commands/lxd/`, so shared code is imported as a package: `from _helpers.instances import is_protected`. Importing through the `_helpers` package means a helper named after a standard module, such as `json.py`, can never shadow the real one.

## Shared helpers: `_helpers/`

Each namespace has its own helpers, at the root of the namespace. A command pack is therefore self-contained: copying its folder brings everything its validators need.

```
commands/
├── lxd/
│   ├── _helpers/
│   │   ├── __init__.py
│   │   ├── instances.py
│   │   ├── instances.yaml       ← the commands that use instances.py
│   │   ├── snapshots.py
│   │   └── snapshots.yaml
│   └── apt-upgrade/
└── docker/
    ├── _helpers/
    └── restart-container/
```

- **Only at the root of a namespace.** A `_helpers/` inside a group or a command folder is refused at startup.
- **No sharing between namespaces.** A validator's `PYTHONPATH` reaches only its own namespace, so `_helpers` always means that namespace's helpers. Python cannot stop a validator from loading other files on purpose (see the last section), so a unit test also checks that no validator or helper reaches into another namespace.
- **The guard class comes from shell-gate-mcp itself,** not from a file in each pack, so every pack uses the same guard and no copy can drift.
- **Importable Python only.** Helpers hold data and pure functions, such as `PROTECTED` and `is_protected(name)`.
- **No side effects.** No SSH, no network, no writes. That keeps helpers fast, predictable and easy to test.
- **Never a command.** The `_` prefix keeps `_helpers` out of the command listing, and the name pattern makes it impossible to run.
- **Editing a helper changes every validator that imports it,** on the next call. Helpers therefore have their own unit tests.

### Usage lists: every helper knows who uses it

When someone edits a helper, they need to know exactly which commands the change affects. A comment listing them goes out of date without anyone noticing; a list the code checks cannot.

- **Every helper module declares a guard instance at module level,** for example `USED_BY = UsedBy(__file__)`.
- **Lists use full dotted names,** such as `lxd.apt-upgrade`: exactly what `LAYER_COMMAND` holds and what the audit log shows, so a search finds every use. Every name in a list belongs to the helper's own namespace.
- **The guard reads the YAML file with the same name next to the helper** (`instances.py` → `instances.yaml`), which lists the commands that use it.
- **When the helper is imported,** the guard compares `os.environ['LAYER_COMMAND']` with that list. If the command is not listed, it raises: `command 'lxd.apt-upgrade' uses lxd/_helpers/instances.py but is not listed in lxd/_helpers/instances.yaml`. The validator then fails, and the request is denied.
- **A missing `LAYER_COMMAND` also raises.** Unit tests of helpers set it explicitly.
- **Checking at import is enough.** A module's top-level code runs only on its first import in a process, and each process is one validator for one command.
- **Indirect use counts.** If `lxd.apt-upgrade` imports `middle`, which imports `instances`, then `lxd.apt-upgrade` must be listed in `instances.yaml` as well. The guard checks the running command, not the file that did the import, so each list shows the true reach of an edit.
- **The purpose is bookkeeping, not access control.** The list exists to keep the record of who depends on each helper accurate. A mistake in it denies a command loudly, which is intended: fail hard, nothing silent.

The guard class's name is **(open)**: `AllowedCommands` or `UsedBy`.

### Tests that keep the lists honest

1. **Every helper declares a guard.** A unit test parses each module in every `_helpers/` (except `__init__.py`) and fails if it does not create a guard instance at module level.
2. **Every list is exact.** A unit test parses every `validate_args.py` with `ast`, follows its imports through `_helpers/`, including indirect ones, and asserts that each helper's YAML lists exactly the commands that use it. A missing entry fails the test, and so does a stale one, for a command that no longer uses the helper. The runtime guard can only catch missing entries; this test catches both.
3. **Every name in a list is a real command in the helper's own namespace.**
4. **No namespace reaches into another.** A unit test parses every validator and helper and fails if one imports, adds to `sys.path`, or loads by path anything from another namespace.

## Python imports are not a sandbox

These facts were checked by experiment and shape the rules above:

- **`..` does not work in an import from a validator.** Relative imports follow package structure, not folders on disk. A validator runs as a plain script, belongs to no package, and every relative import fails with `attempted relative import with no known parent package`.
- **Any code can still load code from anywhere.** A validator can add any folder to `sys.path`, load any file by path with `importlib`, or skip imports entirely and use `open()`, `exec()` or `subprocess`.
- **So import rules are discipline, not containment.** Checks on imports catch mistakes, but not code written on purpose to get around them. Real containment would need operating-system tools, such as a separate low-privilege user with no network. Python cannot provide it.
- **The real guarantee is who can write the files.** Validators and helpers are trusted code written by a human, and the agent can never write to the laptop's copy of `commands/`.
- **An imported module can tell which validator is running,** through `LAYER_COMMAND`. It cannot reliably tell which file imported it: its top-level code runs only on the first import, so later importers are invisible. None of this is tamper-proof, and none of it needs to be, for trusted code.
