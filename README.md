# obsidian-mcp

MCP server that gives AI agents full access to your Obsidian vault. Runs locally or in the cloud with always-on access via Obsidian Sync.

## How it works

obsidian-mcp reads and writes your vault's markdown files directly on disk. It exposes them over HTTP/SSE using the [Model Context Protocol](https://modelcontextprotocol.io), so any MCP-compatible client (Claude, Cursor, etc.) can search, read, write, and manage your notes.

**Two ways to run it:**

1. **Local mode** — on the same machine as your vault.
2. **Cloud mode** — on a VPS or Fly.io, paired with [Obsidian Headless Sync](https://github.com/Belphemur/obsidian-headless-sync-docker). Your vault stays synced via Obsidian Sync, and the server is always on — even when your Mac is asleep.

```
Cloud mode:

Your Mac (Obsidian + Sync)
        |  Obsidian Sync (E2E encrypted)
Docker on Fly.io / Railway:
  |- obsidian-headless-sync  ->  /vault/  (synced files)
  '- obsidian-mcp:3456       ->  /vault/  (reads/writes same files)
        |
AI Agent (MCP client)
```

## Local Setup

```bash
git clone https://github.com/madebydia/obsidian-mcp.git
cd obsidian-mcp
npm ci --ignore-scripts
npm run build
cp .env.example .env
```

Edit `.env`:

```env
VAULT_PATH=/Users/yourname/Documents/MyVault
PORT=3456
BIND_ADDRESS=127.0.0.1
DAILY_NOTE_FOLDER=Journal
AUTH_TOKEN=               # set this if exposing beyond localhost
```

Run:

```bash
npm start
curl http://localhost:3456/health
```

Point your MCP client to `http://localhost:3456/sse`.

## Remote Access to Local Mode

If your MCP client runs somewhere other than the machine hosting your vault, expose the local server through a private network or tunnel.

### Tailscale

Tailscale keeps the server private to your tailnet.

```bash
brew install tailscale
brew services start tailscale
sudo tailscale up
tailscale status
```

Set `BIND_ADDRESS=0.0.0.0` in `.env`, restart obsidian-mcp, then connect your MCP client to:

```text
http://<your-mac-hostname>.<tailnet>.ts.net:3456/sse
```

You can find the stable MagicDNS hostname in the Tailscale admin console.

### ngrok

ngrok is faster to test, but creates a public URL. Set `AUTH_TOKEN` before using it.

```bash
brew install ngrok
ngrok http 3456
```

Connect your MCP client to the generated URL:

```text
https://<ngrok-id>.ngrok-free.app/sse
```

If `AUTH_TOKEN` is set, include:

```text
Authorization: Bearer <your-token>
```

## Run Persistently on macOS

For a local Mac server, use `pm2` to keep obsidian-mcp running across terminal sessions and restarts.

```bash
brew install pm2
pm2 start dist/index.js --name obsidian-mcp --env production
pm2 save
pm2 startup
```

Useful commands:

```bash
pm2 logs obsidian-mcp
pm2 status
pm2 restart obsidian-mcp
```

## Cloud Setup

**Prerequisites:** [Obsidian Sync](https://obsidian.md/sync) subscription, Docker.

### 1. Get your Obsidian auth token

```bash
docker run --rm -it --entrypoint get-token \
  ghcr.io/belphemur/obsidian-headless-sync-docker:latest
```

This prompts for your Obsidian email, password, and MFA code. Save the token it prints.

### 2. Configure

```bash
cp .env.example .env
```

Fill in the cloud section:

```env
# Obsidian Sync
OBSIDIAN_AUTH_TOKEN=<token from step 1>
VAULT_NAME=MyVault                    # exact name from Obsidian Sync
VAULT_PASSWORD=                       # only if you use E2E encryption
DEVICE_NAME=obsidian-mcp-cloud

# MCP server
AUTH_TOKEN=<generate a strong token>  # REQUIRED for cloud deploys
PORT=3456
DAILY_NOTE_FOLDER=Journal
```

Generate an auth token:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### 3. Start

```bash
docker compose up -d
```

The sync container will download your vault (first sync may take a minute). obsidian-mcp waits automatically until the vault is ready, then starts accepting connections.

### 4. Verify

```bash
# Check health
curl http://localhost:3456/health
# -> {"status":"ok","noteCount":142,"lastModified":"2025-...","vaultExists":true,...}

# Connect MCP client to:
# http://your-host:3456/sse
```

### Deploy to Fly.io

```bash
fly launch --no-deploy
fly volumes create vault_data --size 1 --region sjc
fly secrets set \
  OBSIDIAN_AUTH_TOKEN=<your-token> \
  AUTH_TOKEN=$(openssl rand -hex 32) \
  VAULT_NAME=<your-vault-name>
fly deploy --dockerfile Dockerfile.fly
```

**Cost:** ~$3-5/month (shared-cpu-1x, 256MB, 1GB volume).

Your MCP endpoint: `https://<your-app>.fly.dev/sse`

### Deploy to Railway

Railway supports Docker Compose natively. Push this repo and set the environment variables in the Railway dashboard.

## Writes and Sync Behavior

When an AI agent writes a note via obsidian-mcp:

1. obsidian-mcp writes the `.md` file to the shared volume
2. The headless sync container detects the change and uploads it via Obsidian Sync
3. The note appears on your Mac, phone, and other devices within a few seconds

This is the same sync mechanism Obsidian uses — version history is preserved, conflicts are handled automatically.

## Tools (10)

| Tool | What it does |
|------|-------------|
| `list_notes` | List all markdown files, optionally filtered by folder |
| `read_note` | Read a note's content and frontmatter |
| `write_note` | Create or overwrite a note (auto-backup on overwrite) |
| `append_note` | Append text to an existing note |
| `edit_note` | Find and replace a unique text string within a note (auto-backup) |
| `delete_note` | Soft-delete to .trash/ (or permanent with flag) |
| `search_vault` | Full-text search across all notes |
| `list_tags` | List all tags and which notes use them |
| `get_daily_note` | Get today's or a specific date's daily note |
| `create_daily_note` | Create a daily note from a template |
| `get_sync_status` | Check for sync conflicts, recently modified files |

## Security

- **Path traversal protection** — all file operations are sandboxed to the vault directory
- **Bearer token auth** — set `AUTH_TOKEN` in `.env` (required for cloud deploys)
- **Obsidian Sync E2E encryption** — supported via `VAULT_PASSWORD`
- **Install scripts disabled** — `.npmrc` sets `ignore-scripts=true`; use `npm ci --ignore-scripts` for reproducible installs without package lifecycle execution

## Requirements

- Node.js 20+ (local mode)
- Docker (cloud mode)
- Obsidian Sync subscription (cloud mode only)

## License

MIT
