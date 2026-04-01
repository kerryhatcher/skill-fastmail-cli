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

## Quick Command Reference

| Task | Command |
|------|---------|
| List mailboxes + unread counts | `fastmail-cli list mailboxes` |
| List inbox emails | `fastmail-cli list emails --limit 20` |
| List emails in a folder | `fastmail-cli list emails --mailbox Sent --limit 10` |
| Get full email with body | `fastmail-cli get EMAIL_ID` |
| Search emails | `fastmail-cli search --from "alice@example.com" --unread` |
| List sending identities | `fastmail-cli list identities` |
| Send a new email | `fastmail-cli send --to "..." --subject "..." --body "..."` |
| Reply to an email | `fastmail-cli reply EMAIL_ID --body "..."` |
| Reply-all | `fastmail-cli reply EMAIL_ID --body "..." --all` |
| Forward an email | `fastmail-cli forward EMAIL_ID --to "..." --body "..."` |
| Move to Archive/Trash | `fastmail-cli move EMAIL_ID --to Archive` |
| Mark as read | `fastmail-cli mark-read EMAIL_ID` |
| Mark as spam | `fastmail-cli spam EMAIL_ID -y` |
| Download attachments | `fastmail-cli download EMAIL_ID` |
| Extract attachment text | `fastmail-cli download EMAIL_ID --format json` |
| List masked emails | `fastmail-cli masked list` |
| Create masked email | `fastmail-cli masked create --domain "https://example.com" --description "..."` |
| Search contacts | `fastmail-cli contacts search "name"` |

## Core Triage Workflow

```bash
# 1. Check unread count
fastmail-cli list mailboxes | jq '.data[] | select(.role == "inbox") | {name, unreadEmails, totalEmails}'

# 2. List recent emails (newest first)
fastmail-cli list emails --limit 20 | jq '.data.emails[] | {id, subject, from: .from[0].email, receivedAt, hasAttachment}'

# 3. Read a specific email
fastmail-cli get EMAIL_ID | jq -r '.data.bodyValues | to_entries[0].value.value'

# 4. Act on it
fastmail-cli reply EMAIL_ID --body "Thanks, I'll look into this."
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
fastmail-cli search --unread | jq '.data.emails[] | "\(.from[0].email): \(.subject)"'

# Check if a command succeeded
fastmail-cli get EMAIL_ID | jq '.success'

# Extract all email IDs from a search
fastmail-cli search --from "boss@example.com" | jq -r '.data.emails[].id'
```

## Output Format

Every command returns:
```json
{
  "success": true,
  "data": { ... },
  "message": "optional status message",
  "error": "error message if success is false"
}
```

Always check `success` before acting on `data`.

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
- Use `--from` to explicitly select the correct sending identity when multiple exist
- Confirm subject, recipients, and body with the user before executing any send/reply/forward
- Check `.success` in the response before treating the operation as complete

### MUST NOT DO
- Send, reply, or forward without user confirmation of the message content
- Run `spam EMAIL_ID -y` or `move ... --to Trash` on multiple emails without explicit user approval per-email or a clearly scoped batch
- Store or log API tokens in command output
- Run `fastmail-cli auth TOKEN` in a terminal with shell history enabled — prefer the config file or env var
- Fetch full email bodies for large batches — get metadata first, then fetch bodies only for emails the user wants to read
