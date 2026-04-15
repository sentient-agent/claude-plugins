# Slack Channel for Claude Code

Custom MCP server that turns a Slack workspace into a real-time messaging channel for Claude Code. Messages sent in configured Slack channels or DMs are pushed directly into the Claude session. Claude responds via the `reply` tool — output in the terminal never reaches Slack.

## How it works

The server uses Slack's **Socket Mode** (WebSocket) to receive messages in real time without exposing a public URL. It connects using two tokens:

- `SLACK_BOT_TOKEN` (`xoxb-...`) — bot user token for sending messages, uploading files, adding reactions
- `SLACK_APP_TOKEN` (`xapp-...`) — app-level token for the Socket Mode WebSocket connection

On startup the server:
1. Reads tokens from `~/.claude/channels/slack/.env`
2. Loads access policy from `~/.claude/channels/slack/access.json`
3. Opens a Socket Mode WebSocket to Slack
4. Registers as an MCP server with the `claude/channel` experimental capability
5. Forwards allowed messages into the Claude session as `notifications/claude/channel`

### MCP tools exposed

| Tool | Purpose |
|---|---|
| `reply` | Send a message to a Slack channel/DM. Supports file attachments and threads. |
| `react` | Add an emoji reaction to a message. |
| `download_attachment` | Download a file attachment to the local inbox. |

### Access control

Managed via the `/slack:access` skill (run in the terminal, never from a Slack message).

- **DM policy**: `pairing` (new users must complete a 6-character code exchange) or `allowlist` (only pre-approved handles)
- **Channel subscriptions**: per-channel config with optional `requireMention` and per-channel `allowFrom`
- **State directory**: `~/.claude/channels/slack/`
  - `.env` — tokens (mode 0600)
  - `access.json` — policy config
  - `approved/` — pairing confirmations
  - `inbox/` — downloaded attachments

## Why this approach exists

Claude Code has a built-in plugin/channel system. Official channels (Telegram, iMessage) are distributed through the `claude-plugins-official` marketplace maintained by Anthropic. There is no Slack channel in the official marketplace — only an official Slack **MCP plugin** (read/search/send tools, no real-time channel capability).

To run a custom channel, Claude Code requires:

1. The MCP server must declare `claude/channel` as an experimental capability
2. The server must be listed in the `--channels` or `--dangerously-load-development-channels` CLI flags at launch
3. The plugin must be registered in `~/.claude/plugins/installed_plugins.json`
4. The plugin must be enabled in `~/.claude/settings.json` under `enabledPlugins`
5. **The plugin name must exist in the marketplace manifest** (`marketplace.json`) of the marketplace it claims to belong to

Requirement 5 is the critical constraint. Claude Code validates that the plugin name exists in the marketplace before loading it. Since we can't add entries to Anthropic's marketplace manifest, we exploit the fact that `slack` already exists there (as the official Slack MCP plugin). By injecting our channel code into the plugin cache at a **higher version number**, our code takes priority over the official plugin.

## Architecture

```
Source code (this directory)
  ~/Documents/Development/claude-plugins/channels/slack/

Wrapper script (aliased as `claude`)
  ~/bin/claude-launch

Plugin cache (where Claude Code reads from)
  ~/.claude/plugins/cache/claude-plugins-official/slack/1.0.1/

Plugin registry
  ~/.claude/plugins/installed_plugins.json

Plugin enable/disable state
  ~/.claude/settings.json → enabledPlugins

Channel runtime state
  ~/.claude/channels/slack/
```

### The wrapper: `~/bin/claude-launch`

The user's shell aliases `claude` to `~/bin/claude-launch`. The wrapper must run **unconditionally on every launch** because Claude Code's marketplace sync actively fights our setup. On every launch, Claude Code:
- Re-downloads the official Slack plugin to `slack/1.0.0`
- Overwrites `installed_plugins.json` to point back to the official version
- Orphans our `slack/1.0.1` directory

The wrapper counteracts this by running these steps **before** Claude Code starts:

1. **Copies source files** from this directory into the plugin cache at `~/.claude/plugins/cache/claude-plugins-official/slack/1.0.1/`
2. **Deletes all competing versions** — removes every directory under `slack/` except `1.0.1` (nukes the official `1.0.0` that marketplace sync re-downloads)
3. **Fixes `installed_plugins.json`** — overwrites the `slack@claude-plugins-official` entry to point to our `1.0.1` (undoes marketplace sync's revert to `1.0.0`)
4. **Removes orphan markers** — deletes `.orphaned_at` file if Claude Code left one
5. **Launches Claude Code** with:
   - `--channels plugin:telegram@claude-plugins-official plugin:imessage@claude-plugins-official` — loads production channels
   - `--dangerously-load-development-channels plugin:slack@claude-plugins-official` — loads this channel, bypassing the allowlist check (marks it as `dev: true` internally, which skips the channel allowlist validation)

All four pre-launch steps run unconditionally (not gated on file staleness) because the marketplace sync clobbers our setup on every launch.

The `--dangerously-load-development-channels` flag is an undocumented internal flag. It does not appear in `--help` output. It triggers an interactive confirmation dialog on first use per session ("I am using this for local development"). It sets `dev: true` on the channel entry, which bypasses the `allowedChannelPlugins` allowlist check in the channel registration logic.

### Why version 1.0.1?

The official Slack MCP plugin from the marketplace is version `1.0.0`. Claude Code loads the highest version in the cache for a given plugin name. By using `1.0.1`, our channel code wins the version race. The wrapper deletes the official `1.0.0` on every launch to prevent it from competing.

### The marketplace sync problem

Claude Code syncs with the `claude-plugins-official` marketplace on startup. This sync:
1. Sees `slack` in the marketplace manifest at version `1.0.0`
2. Downloads/restores the official plugin to `slack/1.0.0` in cache
3. Updates `installed_plugins.json` to point to `1.0.0`
4. Orphans any unrecognized versions (like our `1.0.1`)

This means **the wrapper's pre-launch fixup is not optional** — it's a race against the marketplace sync that happens every session. The wrapper wins because it runs before `exec claude`, and Claude Code reads the registry at process start.

### How Claude Code decides to load a channel

Extracted from the binary (function `m4_` in the minified source):

```
1. Server declares experimental capability "claude/channel"?     → skip if no
2. Channels feature available (feature flag)?                     → skip if no
3. User authenticated with claude.ai?                             → skip if no
4. Org policy allows channels? (team/enterprise only)             → skip if no
5. Server in --channels or --dangerously-load-development list?   → skip if no
6. For plugins: marketplace matches installed plugin's source?    → skip if mismatch
7. For non-dev plugins: on the approved channels allowlist?       → skip if no
8. All checks pass                                                → register
```

Step 7 is bypassed for dev channels (loaded via `--dangerously-load-development-channels`), which is why that flag is required for this custom channel.

## Files in this directory

| File | Purpose |
|---|---|
| `server.ts` | MCP server implementation (~700 lines). Socket Mode client, access control, message routing, tool handlers. |
| `.claude-plugin/plugin.json` | Plugin metadata. **Name must be `slack`** — matches the marketplace entry. Version doesn't matter here (the cache directory version wins). |
| `.mcp.json` | MCP server config. Tells Claude Code how to start the server (`bun run start`). **Server key must be `slack`** to match the `--channels` flag. |
| `package.json` | Dependencies: `@modelcontextprotocol/sdk`, `@slack/web-api`, `@slack/socket-mode`. Start script runs `bun install` then `bun server.ts`. |
| `skills/access/SKILL.md` | `/slack:access` skill — approve pairings, manage allowlists, add/remove channels. |
| `skills/configure/SKILL.md` | `/slack:configure` skill — set up tokens, check status. |

## Configuration files (outside this directory)

### `~/.claude/plugins/installed_plugins.json`

Must contain an entry for `slack@claude-plugins-official` pointing to the cache:

```json
"slack@claude-plugins-official": [
  {
    "scope": "user",
    "installPath": "/Users/alucavi/.claude/plugins/cache/claude-plugins-official/slack/1.0.1",
    "version": "1.0.1",
    "installedAt": "2026-04-15T05:48:00.000Z",
    "lastUpdated": "2026-04-15T05:48:00.000Z"
  }
]
```

### `~/.claude/settings.json`

Must have the plugin enabled:

```json
"enabledPlugins": {
  "slack@claude-plugins-official": true
}
```

### `~/.claude/channels/slack/.env`

```
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
```

### `~/.claude/channels/slack/access.json`

Managed by `/slack:access`. Example:

```json
{
  "dmPolicy": "pairing",
  "allowFrom": [],
  "channels": {
    "C0A3RLT557G": { "requireMention": false, "allowFrom": [] }
  },
  "pending": {}
}
```

## Maintenance

### After Claude Code updates

Claude Code auto-updates frequently. Updates may:

- **Wipe the plugin cache** — the wrapper handles this automatically. It copies files unconditionally on every launch.
- **Overwrite `installed_plugins.json`** — the wrapper handles this automatically. It rewrites the `slack@claude-plugins-official` entry on every launch.
- **Re-download the official Slack plugin** — the wrapper handles this automatically. It deletes competing versions on every launch.
- **Overwrite `settings.json`** — unlikely but possible. Check that `slack@claude-plugins-official` is still in `enabledPlugins`. The wrapper does NOT fix this file.

### If the official Slack plugin version exceeds 1.0.1

The official Slack MCP plugin (`slackapi/slack-mcp-plugin`) is currently at `1.0.0`. If Anthropic updates it to `1.0.2` or higher:

1. The new version will be cached at `slack/<new-version>/`
2. Claude Code will load the higher version, which is the official HTTP-based MCP plugin (not our channel)
3. **Fix**: bump our version in the wrapper's `PLUGIN_DST` path to be higher than the new official version (e.g., `1.0.3`), update `installed_plugins.json` to match, and delete the official version's cache directory

### Diagnosing issues

**"plugin not installed" at startup:**
```bash
# Check the plugin list for errors
claude plugin list --json 2>&1 | python3 -m json.tool

# Common causes:
# - installed_plugins.json points to wrong version/path
# - Cache directory was wiped and wrapper hasn't re-copied yet
# - Official slack plugin was re-installed at a higher version
```

**Server not starting (no bun process):**
```bash
# Check for orphan markers
ls ~/.claude/plugins/cache/claude-plugins-official/slack/1.0.1/.orphaned_at

# Check if the right version is in cache
ls ~/.claude/plugins/cache/claude-plugins-official/slack/

# Check enabledPlugins in settings
cat ~/.claude/settings.json

# Test the server directly
cd ~/.claude/plugins/cache/claude-plugins-official/slack/1.0.1
bun run start
```

**Messages not arriving:**
```bash
# Check for competing server processes (zombie sessions)
ps aux | grep "bun.*server.ts" | grep -v grep

# Each session gets its own server. Multiple sessions = multiple Socket Mode
# connections to the same Slack app. Slack delivers each message to only ONE
# connection. Kill stale sessions if messages are going to the wrong one.
```

**Version collision after update:**
```bash
# Check what versions exist in cache
ls ~/.claude/plugins/cache/claude-plugins-official/slack/

# If there's a newer official version, delete it and bump ours
rm -rf ~/.claude/plugins/cache/claude-plugins-official/slack/1.0.0
# Then update PLUGIN_DST in ~/bin/claude-launch and installed_plugins.json
```

## Caveats

1. **This is a hack.** We impersonate the official `slack` plugin name to pass marketplace validation. This works today but could break if Anthropic changes how plugin validation works.

2. **Version racing.** If the official Slack plugin updates past our version, it silently takes over. There's no warning — the channel just stops working and the official MCP tools appear instead.

3. **Single consumer.** Slack Socket Mode delivers each message to one WebSocket connection. If multiple Claude sessions are running, messages go to whichever session's server received the Socket Mode event. There's no fan-out.

4. **`--dangerously-load-development-channels` is undocumented.** The flag exists in the binary but not in `--help`. It could be removed or changed in any update.

5. **The wrapper must run on every launch.** If Claude is started without the wrapper (e.g., from an IDE extension, the web app, or directly via the binary), the channel won't load.

6. **Registry edits are fragile.** `installed_plugins.json` and `settings.json` can be overwritten by Claude Code during plugin install/uninstall/update operations. Manual edits to these files are not persistent across all operations.
