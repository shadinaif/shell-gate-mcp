# MCP interface

## One server

The layer is a single MCP server. It has one process to start, stop and watch, one whitelist, one audit log and one token, and the job tools can see every job because all jobs live in the same place. Splitting it into several servers would add no security, since they would all run on the same laptop with the same SSH key.

A second server makes sense only for a second environment with different stakes, such as production next to staging. Each would then have its own SSH key, token, `commands/` and log.

Any MCP client that supports the streamable HTTP transport connects to it. In Claude Code, for example:

```bash
claude mcp add --transport http shell-gate http://<laptop address>:<port>/mcp --header "Authorization: Bearer <token>"
```

## Arguments

- **Always named.** MCP has no positional arguments: every tool call sends its arguments as a JSON object, so every value carries its name. Examples in these docs are always written with named arguments, `wait_job(job_id="j-123", cursor=40)`, to match what is actually sent.
- **Closed.** Every input schema sets `additionalProperties: false` at every level. A misspelled or unknown argument is rejected, never ignored.
- **Typed.** Every argument has a type, and constrained values have a pattern or bounds. Anything that does not match is rejected before any code runs.
- **Keyword-only in the Python code too.** Tool functions are declared as `def wait_job(*, job_id: str, cursor: int = 0)`, so internal calls are as strict as calls from the agent.
- **Objects are allowed, free-form objects are not.** An argument can be a JSON object, but always a typed model that lists every allowed key. A bare `dict` would accept any key and open a hole in the whitelist. Nesting stays shallow, one or two levels, and every field has a `description`, because that is what the agent reads to know what to send.
- **A nested object sent as a JSON string is rejected** with an explicit error, not parsed. Some clients occasionally send `"limits": "{\"cpu\": 2}"` instead of an object; the server stays strict.

### One schema per tool

A tool has exactly one input schema, so there are no overloads. Instead:

- **Different behaviours are different tools**, each with a short schema.
- **Modifiers of one behaviour are optional arguments**, and the server rejects combinations that make no sense with an explicit error.
- **A tagged union is allowed only nested inside one argument**, never as the whole input schema, because client support for unions at the top level is uneven.

## Tools

### Discovery

| Tool | Returns |
|---|---|
| `list_commands()` | Every command's name, description and tier. Broken commands are listed as broken, with the loader's error. |
| `describe_command(cmd=...)` | That command's full argument schema |

### Starting a command

`run_command(cmd=..., args={...})` runs every check in [validation.md](validation.md) and replies with:

```json
{"job_id": "j-0142", "decision": "allowed"}
```

`decision` is `allowed`, `pending` (waiting for approval) or `denied`. A denial always carries the reason, so the agent knows why and does not retry blindly:

```json
{"decision": "denied", "reason": "command 'lxd.delete-instance' does not exist"}
```

Whether `run_command` waits a few seconds so that quick commands come back already finished is **(open)**.

### Following a job

| Tool | Returns |
|---|---|
| `job_status(job_id=...)` | The job's state, exit code and duration |
| `read_output(job_id=..., cursor=..., max_lines=...)` | The output lines after `cursor`, the next cursor, and whether the job is done |
| `wait_job(job_id=..., cursor=..., timeout_s=...)` | Blocks until new output arrives, the job ends or the timeout passes, then returns like `read_output` |
| `cancel_job(job_id=...)` | Stops a running job |
| `list_jobs(status=...)` | Recent jobs, so the agent can pick a job back up after losing track of it |

How the agent reads only part of a long output (the tail, or lines matching a pattern) is **(open)**: separate tools, or arguments on `read_output`.

A typical run:

```
run_command(cmd="lxd.restart-instance", args={"instance": "web"})  → {job_id: "j-0142", decision: "allowed"}
wait_job(job_id="j-0142", cursor=0)                            → {status: "running", output: [...], cursor: 12}
wait_job(job_id="j-0142", cursor=12)                           → {status: "succeeded", exit_code: 0, output: [...], cursor: 47}
```

### Why jobs and not one call

The model sees a tool's result only when the call returns. A long command in a single call would leave the agent blind until it ended and would run into tool-call timeouts. With a job id, the agent starts the command, then reads its output in steps, while the command is still running.

Jobs belong to the layer, not to the MCP connection. A job keeps running if the agent's connection drops, and the agent finds it again with `list_jobs`. Only `cancel_job` stops it.

## Job states

| State | Meaning |
|---|---|
| `pending_approval` | Waiting for a human to approve an `approve`-tier command |
| `running` | Running on the target server |
| `succeeded` | Finished with exit code 0 |
| `failed` | Finished with a non-zero exit code, or without one |
| `cancelled` | Stopped by `cancel_job` |
| `timed_out` | Killed after reaching the command's `timeout_s` |
| `lost` | The SSH connection dropped mid-job. The remote command may still be running, and the state says so. |

A denied request never becomes a running job, but it is still written to the audit log.

## What is kept on the laptop

- **One folder per job,** with `request.json` (the call as the agent sent it), `output.jsonl` (every output line, with its stream and sequence number) and `result.json` (the final state, exit code and duration).
- **One append-only audit log,** with one line per request: time, command, arguments, decision, final state and the hash of the `command.yaml` that was used. If a command is edited later, the log still shows exactly which definition ran.

The agent can read job output through the tools, but it can never edit or delete the job folders or the audit log.

## Permissions in the client

An MCP client's per-tool permissions, such as Claude Code's, see `run_command` as one tool, whatever command it is asked to run. The layer's tiers are the real gate, and they are the stronger one: they run on the laptop, outside the VM.
