---
name: fastmail-cli
description: Use when reading, searching, sending, or managing Fastmail email from the terminal. Covers inbox triage, email search, compose/reply/forward, thread reading, attachment handling, masked email management, and contacts. All operations use fastmail-cli bash commands with JSON output.
license: MIT
metadata:
  domain: productivity
  triggers: fastmail, email, inbox, send email, reply to email, forward email, search email, email triage, unread emails, email thread, masked email, contacts, JMAP, list emails, check email, mark as read, move to archive, move to trash, mark spam
  role: specialist
  scope: productivity
  output-format: json
  related-skills: find-skills
---

# fastmail-cli

CLI specialist for reading, searching, composing, and managing Fastmail email via `fastmail-cli`.

Default compose behavior: create Fastmail drafts first. Only send immediately when the user explicitly instructs you to send now.

## When to Use This Skill

- Reading or triaging an inbox
- Searching for emails by sender, subject, date, or content
- Sending, replying to, or forwarding email
- Downloading or extracting text from attachments
- Managing masked email addresses
- Looking up contacts

## Prerequisites

`fastmail-cli` must be authenticated. Check with:

```bash
fastmail-cli list mailboxes 2>&1 | head -3
```

Authentication uses one of:

**Config file** (`~/.config/fastmail-cli/config.toml`):
```toml
[core]
api_token = "fmu1-..."
```

**Environment variable:**
```bash
export FASTMAIL_API_TOKEN="fmu1-..."
```

Generate a token at: Fastmail Settings → Privacy & Security → API tokens.

> Contacts require a separate app password — see `references/contacts.md`.

## Local State

Keep a local JSONL cache and action log for every Fastmail session.

- State directory: `~/.local/share/fastmail-cli-agent/`
- Inbox cache: `~/.local/share/fastmail-cli-agent/inbox-cache.jsonl`
- Action log: `~/.local/share/fastmail-cli-agent/actions.jsonl`

Inbox cache rules:

- Create the state directory before first use.
- Refresh the inbox cache before inbox triage work and after any action that changes inbox contents.
- Store one JSON object per line with mailbox metadata needed for fast local review, for example: `id`, `threadId`, `receivedAt`, `from`, `subject`, `preview`, `mailboxIds`, `keywords`, `hasAttachment`, `size`.
- Rewrite the cache atomically via a temporary file and rename, not in-place.

Action log rules:

- Append one JSON object per line for every mutating operation: `send`, `reply`, `forward`, `move`, `mark-read`, `spam`, `trash`, masked email changes, and similar write actions.
- Include enough context to audit what happened: `timestamp`, `action`, `success`, `emailId`, `threadId`, `fromMailbox`, `toMailbox`, `subject`, `from`, `to`, and a short `note` when useful.
- Log failures as well as successes.
- Never log API tokens or full message bodies. Keep the log to metadata only.

## Quick Command Reference

| Task | Command |
|------|---------|
| List mailboxes + unread counts | `fastmail-cli list mailboxes` |
| List inbox emails | `fastmail-cli list emails --limit 20` |
| List emails in a folder | `fastmail-cli list emails --mailbox Sent --limit 10` |
| Get full email with body | `fastmail-cli get EMAIL_ID` |
| Search emails | `fastmail-cli search --from "alice@example.com" --unread` |
| List sending identities | `fastmail-cli list identities` |
| Draft a new email | `fastmail-cli send --to "..." --subject "..." --body "..." --draft` |
| Draft a reply | `fastmail-cli reply EMAIL_ID --body "..." --draft` |
| Draft a reply-all | `fastmail-cli reply EMAIL_ID --body "..." --all --draft` |
| Draft a forward | `fastmail-cli forward EMAIL_ID --to "..." --body "..." --draft` |
| Move to Archive/Trash | `fastmail-cli move EMAIL_ID --to Archive` |
| Mark as read | `fastmail-cli mark-read EMAIL_ID` |
| Mark as spam | `fastmail-cli spam EMAIL_ID -y` |
| Download attachments | `fastmail-cli download EMAIL_ID` |
| Extract attachment text | `fastmail-cli download EMAIL_ID --format json` |
| List masked emails | `fastmail-cli masked list` |
| Create masked email | `fastmail-cli masked create --domain "https://example.com" --description "..."` |
| Search contacts | `fastmail-cli contacts search "name"` |

## Response Shapes

The installed CLI does not use one uniform `data` shape. Match `jq` filters to the command you ran:

- `fastmail-cli list mailboxes` → `.data[]`
- `fastmail-cli list identities` → `.data[]`
- `fastmail-cli list emails ...` → `.data.emails[]`
- `fastmail-cli search ...` → `.data[]`
- `fastmail-cli get EMAIL_ID` → `.data`

## Core Triage Workflow

```bash
# 0. Ensure local state exists
mkdir -p ~/.local/share/fastmail-cli-agent

# 1. Check unread count
fastmail-cli list mailboxes | jq '.data[] | select(.role == "inbox") | {name, unreadEmails, totalEmails}'

# 2. Refresh the local inbox cache (newest first)
fastmail-cli list emails --limit 200 | jq -c '.data.emails[] | {id, threadId, receivedAt, from, subject, preview, mailboxIds, keywords, hasAttachment, size}' > ~/.local/share/fastmail-cli-agent/inbox-cache.jsonl.tmp && mv ~/.local/share/fastmail-cli-agent/inbox-cache.jsonl.tmp ~/.local/share/fastmail-cli-agent/inbox-cache.jsonl

# 3. List recent emails from the cache
jq -c '{id, subject, from: (.from[0].email // null), receivedAt, hasAttachment}' ~/.local/share/fastmail-cli-agent/inbox-cache.jsonl

# 4. Read a specific email
fastmail-cli get EMAIL_ID | jq -r '.data.bodyValues | to_entries[0].value.value'

# 5. Act on it and append an audit entry
fastmail-cli reply EMAIL_ID --body "Thanks, I'll look into this." --draft
fastmail-cli move EMAIL_ID --to Archive
fastmail-cli mark-read EMAIL_ID
```

## jq Cheat Sheet

All `fastmail-cli` commands return JSON. Use `jq` to extract what you need:

```bash
# Unread count for inbox
fastmail-cli list mailboxes | jq '.data[] | select(.role == "inbox") | .unreadEmails'

# Email list: id + subject + sender
fastmail-cli list emails | jq '.data.emails[] | {id, subject, from: .from[0].email}'

# Email list: unread only (client-side filter)
fastmail-cli list emails | jq '.data.emails[] | select(.keywords | has("$seen") | not) | {id, subject}'

# Plain text body of an email
fastmail-cli get EMAIL_ID | jq -r '.data.bodyValues | to_entries[0].value.value'

# Subject + sender from search results
fastmail-cli search --unread | jq -r '.data[] | "\(.from[0].email): \(.subject)"'

# Check if a command succeeded
fastmail-cli get EMAIL_ID | jq '.success'

# Extract all email IDs from a search
fastmail-cli search --from "boss@example.com" | jq -r '.data[].id'
```

## Output Format

Every command returns:
```json
{
  "success": true,
  "data": { ... } | [ ... ],
  "message": "optional status message",
  "error": "error message if success is false"
}
```

`data` may be an object or an array depending on the command. Always check `success` before acting on `data`.

## Reference Guide

Load detailed guidance based on the task at hand:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Inbox triage | `references/triage.md` | listing, reading, moving, marking, threading |
| Search | `references/search.md` | filtering by sender, subject, date, flags, size |
| Compose | `references/compose.md` | send, reply, forward, identity selection |
| Attachments | `references/attachments.md` | download, text extraction, image resizing |
| Masked email | `references/masked-email.md` | create, list, enable, disable, delete aliases |
| Contacts | `references/contacts.md` | CardDAV setup, list and search contacts |

## Constraints

### MUST DO
- Run `fastmail-cli get EMAIL_ID` to read the full body before composing a reply or forward
- Use `--limit` on all `list` and `search` calls to avoid fetching excessive data
- Pipe output through `jq` to show the user only the relevant fields
- Maintain `~/.local/share/fastmail-cli-agent/inbox-cache.jsonl` as a JSONL cache of inbox metadata
- Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for every mutating Fastmail action
- Use `--from` to explicitly select the correct sending identity when multiple exist
- Default to `--draft` for `send`, `reply`, and `forward`; only omit `--draft` when the user explicitly asks to send immediately
- Confirm subject, recipients, and body with the user before executing any send/reply/forward/draft
- Check `.success` in the response before treating the operation as complete

### MUST NOT DO
- Send, reply, or forward without user confirmation of the message content
- Omit `--draft` on compose actions unless the user explicitly asked to send immediately
- Run `spam EMAIL_ID -y` or `move ... --to Trash` on multiple emails without explicit user approval per-email or a clearly scoped batch
- Store or log API tokens in command output
- Store full message bodies in the inbox cache or action log unless the user explicitly asks for that behavior
- Run `fastmail-cli auth TOKEN` in a terminal with shell history enabled — prefer the config file or env var
- Fetch full email bodies for large batches — get metadata first, then fetch bodies only for emails the user wants to read
