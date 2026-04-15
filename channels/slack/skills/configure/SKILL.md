---
name: configure
description: Set up the Slack channel — save tokens and review access policy. Use when the user pastes Slack tokens, asks to configure Slack, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /slack:configure — Slack Channel Setup

Writes Slack tokens to `~/.claude/channels/slack/.env` and orients the
user on access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Tokens** — check `~/.claude/channels/slack/.env` for
   `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN`. Show set/not-set; if set,
   show first 10 chars masked (`xoxb-556870...`).

2. **Access** — read `~/.claude/channels/slack/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list user IDs
   - Monitored channels: count, with IDs and policies
   - Pending pairings: count, with codes and user IDs if any

3. **What next** — end with a concrete next step based on state:
   - No tokens → *"Run `/slack:configure bot <bot-token> app <app-token>`."*
   - Tokens set, nobody allowed → *"DM your bot on Slack. It replies with a
     code; approve with `/slack:access pair <code>`."*
   - Tokens set, someone allowed → *"Ready. DM your bot or add channels
     with `/slack:access channel add <channelId>`."*

**Push toward lockdown.** Once user IDs are captured via pairing, suggest
switching to `allowlist` policy.

### `bot <bot-token> app <app-token>` — save tokens

1. Parse the arguments to extract both tokens.
2. `mkdir -p ~/.claude/channels/slack`
3. Read existing `.env` if present; update/add the token lines,
   preserve other keys. Write back, no quotes around values.
4. `chmod 600 ~/.claude/channels/slack/.env`
5. Confirm, then show the no-args status.

### `clear` — remove tokens

Delete the token lines (or the file if those are the only lines).

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Token changes need a session restart.
  Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/slack:access` take effect immediately, no restart.
