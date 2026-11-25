# Failure handling

## The rule: fail hard, nothing silent

Every failure is loud and explicit. Nothing is skipped, defaulted, truncated or passed without saying so. When something is ambiguous, the layer refuses and explains why. A bookkeeping mistake that denies a command in real use is acceptable; a mistake that lets something through quietly is not.

## At startup: refuse to start

The server checks every definition before it accepts a single request. If anything is invalid, it refuses to start.

What it checks:

1. Every folder in `commands/`, except those starting with `_`, follows the name and folder rules in [commands.md](commands.md): each part of its path is a valid name, every command sits inside a namespace, and no folder is both a command and a group.
2. Every `command.yaml` loads, its `args` schema is valid JSON Schema with `additionalProperties: false` at every level, its `tier` is known, and it has exactly one of `exec` and `script`.
3. Every `exec` placeholder fills a whole element and names an argument that the schema defines.
4. Every script and validator a command uses exists inside that command's own folder.
5. Every `_helpers/` sits at the root of a namespace. Every helper in it declares a guard instance and has its matching YAML file, and every name listed in each YAML file is a real command in the helper's own namespace.
6. The audit log and the job folders are writable.

When checks fail, the server **reports every problem it found**, not just the first, and exits with a non-zero code, so everything can be fixed in one pass.

Edits made to `commands/` while the server is running are checked again on every call, and a broken command fails loudly on its own.

Whether startup also checks the SSH connection to the target server is **(open)**.

## At runtime

| Where | What must never happen | What happens instead |
|---|---|---|
| Audit log | The log write fails and the job runs anyway | No log, no run: the request is refused |
| Broken `command.yaml` | `list_commands()` quietly drops the command | It is listed as broken, with the loader's error, and `run_command` refuses it with that error |
| Output limits | Output is cut off and the agent does not know | Every page says `truncated: true` and how much is missing |
| Timeout | The job is reported as a plain failure | Its own state, `timed_out`, with the limit that was reached |
| SSH drops mid-job | The job looks finished, or stays `running` forever | State `lost`, with the SSH error, saying the remote command may still be running |
| No exit code | Treated as success | Treated as failure |
| Unknown job id | An empty result | An explicit error: `no such job 'j-999'` |
| Validator crashes or times out | The request goes through | The request is denied, with the crash or timeout as the reason |
| Validator exits non-zero with no stderr | A denial with no reason | Denied with `validator exited <code> without giving a reason` |
| Helper used by an unlisted command | The helper runs anyway | The guard raises, naming the command, the helper and the YAML file to fix |
| `LAYER_COMMAND` missing | The guard assumes something | The guard raises |
| Unknown or misspelled argument | It is ignored | The request is rejected, naming the argument |
| Nested object sent as a JSON string | The string is parsed | The request is rejected with an explicit error |
