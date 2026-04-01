# Inbox Triage

## List Mailboxes

Get all folders with unread and total counts:

```bash
fastmail-cli list mailboxes
```

```bash
# Inbox unread count only
fastmail-cli list mailboxes | jq '.data[] | select(.role == "inbox") | .unreadEmails'

# All mailboxes: name + unread
fastmail-cli list mailboxes | jq '.data[] | {name, role, unreadEmails, totalEmails}'

# Mailboxes with unread emails
fastmail-cli list mailboxes | jq '.data[] | select(.unreadEmails > 0) | {name, unreadEmails}'
```

Common mailbox names: `Inbox`, `Sent`, `Drafts`, `Trash`, `Spam`, `Archive`

## List Emails

`fastmail-cli list emails` returns email rows at `.data.emails[]`.

```bash
# INBOX, most recent 20
fastmail-cli list emails --limit 20

# A different folder
fastmail-cli list emails --mailbox Archive --limit 10
fastmail-cli list emails --mailbox Sent --limit 10

# Compact view: id, subject, sender, date
fastmail-cli list emails --limit 20 | jq '.data.emails[] | {id, subject, from: .from[0].email, receivedAt, hasAttachment}'

# Show only emails with attachments
fastmail-cli list emails --limit 50 | jq '.data.emails[] | select(.hasAttachment == true) | {id, subject, from: .from[0].email}'
```

> The default limit is 50. Always pass `--limit` to control how many are fetched.

## Get Full Email (with Body)

```bash
fastmail-cli get EMAIL_ID
```

```bash
# Plain text body
fastmail-cli get EMAIL_ID | jq -r '.data.bodyValues | to_entries[0].value.value'

# Subject + sender + body
fastmail-cli get EMAIL_ID | jq '{subject: .data.subject, from: .data.from[0].email, body: (.data.bodyValues | to_entries[0].value.value)}'

# Check if it has attachments
fastmail-cli get EMAIL_ID | jq '.data.hasAttachment'

# All recipients
fastmail-cli get EMAIL_ID | jq '{to: [.data.to[]?.email], cc: [.data.cc[]?.email]}'
```

## Thread Reading

Emails share a `threadId` when they belong to the same conversation. To read a full thread:

`fastmail-cli search` returns matches at `.data[]`.

```bash
# Step 1: get threadId from an email
THREAD_ID=$(fastmail-cli get EMAIL_ID | jq -r '.data.threadId')

# Step 2: search all emails in that thread
fastmail-cli search --text "" | jq --arg tid "$THREAD_ID" '.data[] | select(.threadId == $tid) | {id, subject, from: .from[0].email, receivedAt}'
```

Or list emails and filter client-side if you have the thread ID:

```bash
fastmail-cli list emails --limit 100 | jq --arg tid "THREAD_ID" '.data.emails[] | select(.threadId == $tid)'
```

## Mark as Read / Unread

```bash
# Mark a single email as read
fastmail-cli mark-read EMAIL_ID

# Mark as unread
fastmail-cli mark-read EMAIL_ID --unread
```

Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for each read/unread change.

## Move Emails

```bash
# Archive
fastmail-cli move EMAIL_ID --to Archive

# Trash
fastmail-cli move EMAIL_ID --to Trash

# Any named folder
fastmail-cli move EMAIL_ID --to "Work"
```

> Use exact mailbox names as returned by `fastmail-cli list mailboxes`.

Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for each move, and refresh `~/.local/share/fastmail-cli-agent/inbox-cache.jsonl` after moves that affect inbox contents.

## Mark as Spam

```bash
# Prompts for confirmation
fastmail-cli spam EMAIL_ID

# Skip confirmation (use with caution)
fastmail-cli spam EMAIL_ID -y
```

Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for each spam action.

## Batch Triage Pattern

When triaging multiple emails, work through them one at a time:

```bash
# 1. Get a list of candidate IDs
fastmail-cli list emails --limit 20 | jq -r '.data.emails[].id'

# 2. For each ID, read + decide
fastmail-cli get EMAIL_ID | jq -r '.data.bodyValues | to_entries[0].value.value'

# 3. Dispatch
fastmail-cli mark-read EMAIL_ID
fastmail-cli move EMAIL_ID --to Archive
# or
fastmail-cli reply EMAIL_ID --body "..."
```

Always confirm destructive actions (spam, trash) with the user before running them for each email.
