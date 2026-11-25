# Topology

## The three machines

| Machine | Role | Holds |
|---|---|---|
| **VM** | Runs the AI agent's MCP client, for example Claude Code. A KVM virtual machine on the laptop. | Only the bearer token for the MCP server. No SSH keys. |
| **Laptop** | Runs the layer: the MCP server. | The SSH keys, `commands/`, the job folders and the audit log. |
| **Target server** | The server being managed, for example an LXD host and its instances. | Nothing from this project. Nothing is installed on it. |

The VM cannot reach the target server: the SSH keys live on the laptop and never enter the VM. The VM can reach the laptop over the private virtual network every VM shares with the machine it runs on. For example, the VM at `10.50.130.123` reaches the laptop at its gateway address `10.50.130.1`.

## How a request travels

```
VM                                  Laptop                                       Target server
MCP client  ──MCP over HTTP──▶  layer: check name, args, validator, tier
                                       log the request
                                       run over SSH (laptop's keys)  ──ssh──▶   the command
            ◀──────────────────── output, followed by job id ◀──────────────────────
```

1. The agent calls `run_command(cmd=..., args={...})` on the MCP server.
2. The layer runs every check in [validation.md](validation.md), writes the request to the audit log, and starts a job.
3. The job runs the command on the target server over SSH, with the laptop's keys.
4. The output is stored in the job's folder on the laptop. The agent follows it with the job tools in [mcp-interface.md](mcp-interface.md).

## Network rules

- The MCP server listens on the laptop's address on the VM network, never on an address reachable from outside the laptop.
- The laptop's firewall allows the MCP server's port only from the VM's address, so nothing else on the local network can reach it.
- Every MCP request carries a bearer token. A request without a valid token is refused.
- Only the laptop talks to the target server, the same way it does today. The agent has no network path to the server.

## The off switch

Stopping the MCP server on the laptop cuts the agent off completely. There are no credentials in the VM to revoke.

## What the server trusts

The target server trusts whatever arrives with the laptop's SSH key. That is safe only while the layer is the one using that key on the agent's behalf, which is why every check happens in the layer before anything is sent over SSH.

## Where the code lives and where it runs

The code is written in this repository, inside the VM. It runs on the laptop. Changes reach the laptop only when the laptop's owner pulls them after reviewing them. That review step is what keeps rule 4 in [README.md](README.md) true: The agent can edit this repository, but it can never write to the laptop's copy of `commands/` directly.
