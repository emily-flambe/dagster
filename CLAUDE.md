# Dagster Infrastructure Repository

## Project Context

This is an infrastructure documentation repo for a multi-project Dagster instance on Windows + WSL2.

- **Type**: Documentation-only (no application code, no tests)
- **Main artifact**: `README.md` - comprehensive setup and operations guide
- **Architecture**: Native WSL2 systemd services (migrated from Docker)
- **Config files**: Live in WSL2 filesystem (`/opt/dagster/`), NOT in this repo

## Tech Stack

| Layer | Technology |
|-------|------------|
| Host OS | Windows |
| Runtime | WSL2 Ubuntu with systemd |
| Orchestration | Dagster (webserver + daemon) |
| Database | PostgreSQL (native in WSL2) |
| Network | Tailscale VPN, port proxy |

## Critical Knowledge

1. **Config files are NOT in this repo** - `dagster.yaml`, `workspace.yaml`, and systemd services live in `/opt/dagster/` and `/etc/systemd/system/`

2. **WSL2 IP changes on reboot** - Port proxy must be updated:
   ```powershell
   # PowerShell Admin
   netsh interface portproxy delete v4tov4 listenport=3000 listenaddress=0.0.0.0
   netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=<NEW_WSL_IP>
   ```

3. **Environment variables require BOTH service files** - Update both `dagster-webserver.service` AND `dagster-daemon.service`, then `systemctl daemon-reload`

4. **Workspace changes require restart** - `sudo systemctl restart dagster-webserver dagster-daemon`

## Common Operations

```bash
# Service management (WSL2)
sudo systemctl start dagster-webserver dagster-daemon
sudo systemctl status dagster-webserver dagster-daemon
sudo journalctl -u dagster-webserver -f  # View logs

# PostgreSQL
sudo service postgresql start
sudo service postgresql status

# Get WSL2 IP
wsl -d Ubuntu hostname -I
```

## Workflow

- Single `master` branch, direct commits
- README.md is source of truth - keep it comprehensive and accurate
- Commit footer: `Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>`

## Documentation Standards

When editing README.md:
- Use clear hierarchical markdown with code blocks
- Include complete command examples with context
- Explain "why" for non-obvious decisions
- Keep troubleshooting sections up to date

## Knowledge Graph (Agent-MCP)

After significant changes (new features, architecture decisions, schema changes), save context to Agent-MCP using `update_project_context`. Use the key prefix `dagster/` (e.g., `dagster/architecture`).

Update existing entries when information changes. Create new keys for new topics. This ensures any agent in any session can retrieve project context via `ask_project_rag`.
