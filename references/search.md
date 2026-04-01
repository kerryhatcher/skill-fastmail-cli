# Email Search

## Basic Syntax

```bash
fastmail-cli search [FLAGS]
```

All flags are ANDed together. At least one flag is required.

## All Available Flags

| Flag | Type | Description |
|------|------|-------------|
| `--text TEXT` | string | Full-text search across subject, body, and headers |
| `--from EMAIL` | string | Sender address (partial match) |
| `--to EMAIL` | string | Recipient address (partial match) |
| `--cc EMAIL` | string | CC address |
| `--bcc EMAIL` | string | BCC address |
| `--subject TEXT` | string | Subject line (partial match) |
| `--body TEXT` | string | Body text only |
| `--mailbox NAME` | string | Restrict to a named mailbox |
| `--has-attachment` | flag | Only emails with attachments |
| `--min-size BYTES` | number | Minimum email size in bytes |
| `--max-size BYTES` | number | Maximum email size in bytes |
| `--after DATE` | ISO 8601 | Received after this date (inclusive) |
| `--before DATE` | ISO 8601 | Received before this date (exclusive) |
| `--unread` | flag | Only unread emails |
| `--flagged` | flag | Only flagged/starred emails |
| `--limit N` | number | Max results (default: 50) |

Dates use ISO 8601 format: `YYYY-MM-DD` (e.g. `2026-04-01`).

`fastmail-cli search` returns matches at `.data[]`.

## Common Patterns

### Find unread emails from a specific sender

```bash
fastmail-cli search --from "alice@example.com" --unread --limit 20
```

### Subject keyword search

```bash
fastmail-cli search --subject "invoice" --limit 10
```

### Full-text search across all fields

```bash
fastmail-cli search --text "budget proposal" --limit 10
```

### Date range — emails from this week

```bash
fastmail-cli search --after 2026-03-25 --before 2026-04-01 --limit 50
```

### Emails received today

```bash
# Replace with today's date
fastmail-cli search --after 2026-04-01 --limit 50
```

### Flagged emails needing follow-up

```bash
fastmail-cli search --flagged --limit 20
```

### Large emails (cleanup)

```bash
# Emails over 5MB
fastmail-cli search --min-size 5000000 --limit 20 | jq '.data[] | {id, subject, size, from: .from[0].email}'
```

### Emails with attachments from a sender

```bash
fastmail-cli search --from "contracts@example.com" --has-attachment --limit 20
```

### Search within a specific folder

```bash
fastmail-cli search --mailbox Sent --from "me@example.com" --after 2026-01-01 --limit 20
```

### Find a thread by subject

```bash
fastmail-cli search --subject "Project Alpha" --limit 10 | jq '.data[] | {id, threadId, subject, from: .from[0].email, receivedAt}'
```

## Combining Filters

Filters are ANDed — they all must match:

```bash
# Unread emails with attachments from a specific sender in the last month
fastmail-cli search \
  --from "contracts@example.com" \
  --has-attachment \
  --unread \
  --after 2026-03-01 \
  --limit 20
```

## Extracting Results with jq

```bash
# Summary: sender + subject
fastmail-cli search --unread --limit 20 | jq -r '.data[] | "\(.from[0].email): \(.subject)"'

# Just IDs (for piping to other commands)
fastmail-cli search --from "boss@example.com" | jq -r '.data[].id'

# IDs + subjects as a table
fastmail-cli search --flagged | jq -r '.data[] | [.id, .subject] | @tsv'

# Count results
fastmail-cli search --unread | jq '.data | length'

# Emails received today sorted by time
fastmail-cli search --after 2026-04-01 --limit 50 | jq '.data | sort_by(.receivedAt) | reverse | .[] | {id, subject, from: .from[0].email, receivedAt}'
```

## Notes

- Search is case-insensitive
- Partial email address matches work for `--from`, `--to`, etc. (e.g. `--from "example.com"` matches all senders at that domain)
- `--text` searches across all text fields; `--body` restricts to body only
- `--mailbox` uses the display name (e.g. `Inbox`, `Sent`, `Archive`) — get exact names from `fastmail-cli list mailboxes`
