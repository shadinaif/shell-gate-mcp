# shell-gate-mcp

**A gate, not a shell: your AI agent runs only the commands you defined, with keys it never sees.**

An MCP server that lets an AI agent run commands on your servers without ever holding SSH access to them. It works with any MCP client, such as Claude Code, Cursor or VS Code.

The agent asks shell-gate-mcp to run a named command with named arguments. shell-gate-mcp checks the request against a whitelist of commands, validates the arguments, logs the request, runs the command on the target server over SSH, and hands the output back through a job id that the agent follows until the job ends.

The commands are yours to define, for any server you can reach over SSH. The first setup it is built for manages an LXD host, so the examples in the docs use `lxc`.

## Status

Planning. There is no code yet: the repository holds the architecture docs that the code will be built from.

## Documentation

All decisions and rules live in [docs/architecture/](docs/architecture/README.md). If a decision is not written there, it has not been made.

## License

Licensed under the [Apache License 2.0](LICENSE). See [NOTICE](NOTICE) for the copyright notice.
