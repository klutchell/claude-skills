---
name: sprites
description: >
  Manage temporary dev environments via Sprites (Fly.io persistent Linux VMs).
  Trigger on "sprite", "dev environment", "temporary VM", "disposable VM",
  "cloud computer", "sandbox environment", "spin up a machine", "remote environment".
---

# Sprites Dev Environments

Sprites are persistent, hardware-isolated Linux VMs with durable filesystems. Use them for temporary dev environments, build/test isolation, or running services you don't want on your local machine.

## Lifecycle Workflow

### 1. Create
```
mcp__sprites__create_sprite(name: "my-env")
```
Creates a VM. Name it descriptively — it becomes part of the public URL.

### 2. Setup
```
mcp__sprites__exec(sprite_name: "my-env", command: "apt-get update && apt-get install -y git nodejs")
```
Each `exec` call is a **separate shell session** — no env vars or working directory carry over between calls. Chain commands with `&&` or use `cd /dir && command`.

### 3. Checkpoint
```
mcp__sprites__checkpoint_create(sprite_name: "my-env", name: "after-setup")
```
Snapshots the entire filesystem. Checkpoints are nearly free — **take them aggressively** before risky operations, dependency installs, or config changes.

### 4. Services (for long-running processes)
```
mcp__sprites__service_create(sprite_name: "my-env", name: "web", command: "cd /app && npm start")
```
Services survive hibernation. Use them for anything that needs to keep running (web servers, databases, watchers). TTY sessions from `exec` do NOT survive hibernation.

### 5. Destroy
```
mcp__sprites__destroy_sprite(name: "my-env")
```
Clean up when done. The VM and all its data are deleted.

## MCP Tools Reference

| Tool | Purpose |
|------|---------|
| **Sprite Lifecycle** | |
| `create_sprite` | Create a new VM |
| `destroy_sprite` | Delete a VM and all data |
| `list_sprites` | List all active sprites |
| **Run Commands** | |
| `exec` | Run a command (separate session each call) |
| `exec_list` | List running sessions |
| `exec_kill` | Kill a running session |
| **Checkpoints** | |
| `checkpoint_create` | Snapshot filesystem state |
| `checkpoint_list` | List available checkpoints |
| `checkpoint_get` | Get checkpoint details |
| `checkpoint_restore` | Roll back to a checkpoint |
| **Services** | |
| `service_create` | Create a managed long-running process |
| `service_start` | Start a stopped service |
| `service_stop` | Stop a running service |
| `service_get` | Get service status and config |
| `service_list` | List all services on a sprite |
| `service_logs` | View service output logs |
| **Network** | |
| `policy_network_get` | View network access policy |
| `policy_network_update` | Update network access policy |

All tools are prefixed `mcp__sprites__` when calling.

## CLI Fallback

These operations aren't available via MCP — use Bash:

```bash
# Interactive shell (not possible via MCP)
sprite console -s my-env

# Port forwarding to localhost
sprite proxy -s my-env -p 3000:3000,5432:5432

# File transfer (upload)
cat local-file.tar.gz | base64 | sprite run -s my-env -- bash -c 'base64 -d > /tmp/file.tar.gz'

# File transfer (download)
sprite run -s my-env -- base64 /app/output.json | base64 -d > local-output.json
```

## Key Behaviors

- **Filesystem persists, RAM doesn't** — processes stop on hibernate, but files survive
- **Services survive hibernation**, TTY sessions don't — use services for anything that must persist
- **Each MCP `exec` is isolated** — separate shell session, no shared state between calls
- **Public URL**: `https://<name>-<org>.sprites.dev/` routes to port 8080
- **Checkpoint aggressively** — they're cheap and save you from reinstalling everything
- **Network policy** defaults to restrictive — update it if the sprite needs outbound access
