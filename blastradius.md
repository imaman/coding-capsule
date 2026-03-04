# Risk Profile for Claude Code in Docker

We analyzed the security posture of running Claude Code inside a Docker container. The container runs as non-root user node, has no Docker socket, no SSH keys, no AWS credentials, and no git push access. Linux capabilities are at Docker defaults with CapEff=0x0.

Three bind mounts from the host create the attack surface:

- The source repo (read-write, with read-only overlays on `.claude/` and `.mcp.json`)
- `~/.claude/` config directory (read-only staged copy, with read-write overlays on `projects/` and `history.jsonl` for session persistence)
- `~/.claude.json` (read-only)

The agent also has unrestricted outbound network access.

## Risk Table

| Risk | Impact | Likelihood |
|---|---|---|
| API key theft | High — plaintext Anthropic key readable inside container, trivial exfiltration | Easy |
| Source code exfiltration | High — full read access + outbound network | Easy |
| Conversation history leak | Medium — session data (`projects/`, `history.jsonl`) is read-write; everything else in `~/.claude/` is readable via the staged copy | Easy |
| Conversation history destruction | Medium — session data is read-write and not source-controlled; deletion or corruption is not easily reversible | Easy |

## Mitigated Risks

**Config tampering → host privilege escalation.** Previously the most critical risk. The `~/.claude/` directory was bind-mounted read-write, allowing code inside the container to poison `settings.json` with broad permission allow-lists (e.g., `Bash(*)`), register malicious hooks, or add rogue MCP servers. These changes would take effect the next time Claude Code ran on the host, which has access to SSH keys, AWS credentials, and the full home directory.

Now mitigated: `~/.claude/` is mounted as a read-only staged copy. Only `projects/` and `history.jsonl` are mounted back read-write from the host for session persistence. All config files (settings, credentials, commands, plugins — including any future config surfaces) are kernel-enforced immutable.

The same approach protects project-level config: the repo's `.claude/` directory and `.mcp.json` are mounted as read-only staged copies, preventing tampering with project-level hooks, permission allow-lists, MCP server configs, and custom agent definitions.

## Discussion

API key theft and source code exfiltration are high-impact but cannot be mitigated architecturally — the agent needs to read source code to do its job, and the API key is required for it to function. These are accepted risks inherent to the tool's purpose. Conversation history leak and destruction are consequences of session data being mounted read-write for persistence — the agent needs write access to save sessions, which also means it can exfiltrate or destroy them. This data is not source-controlled, so destruction is not easily reversible.

Several other risks were considered and excluded from the table:

- **Git remote history corruption:** the container has no SSH keys and no push access; confirmed via `ssh -T git@github.com` returning Permission denied.
- **Destruction of unpushed local work:** the repo is mounted read-write, so uncommitted changes, local-only branches, and stashes could be destroyed. However, this only affects work that hasn't been pushed, making the blast radius small for teams that push frequently.
- **Build cache corruption:** the `.build-raptor/` cache is writable but corrupting it only forces rebuilds — no lasting damage.

## Trade-offs

- Claude Code inside the container cannot modify project-level config (`.claude/settings.json`, `.claude/CLAUDE.md`, `.mcp.json`, etc.). Project instructions and settings should be set up outside the container.
- Non-session data written to `~/.claude/` inside the container (e.g., telemetry, debug logs, cache) is discarded when the container exits.
