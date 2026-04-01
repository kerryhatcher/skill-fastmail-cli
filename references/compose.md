# Compose: Send, Reply, Forward

## Safe Compose Pattern

Always follow this sequence before sending anything:

1. **Read the email** you're responding to (`fastmail-cli get EMAIL_ID`)
2. **Summarize to the user** — subject, sender, key content
3. **Draft the response** with the user
4. **Confirm recipients, subject, and body** before executing
5. **Send**

Never execute a send/reply/forward without explicit user confirmation of the final message.

Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for every send, reply, forward, or draft action.

## Identities

List available sending addresses before composing:

```bash
fastmail-cli list identities | jq '.data[] | {id, email}'
```

Use `--from EMAIL` to select the sending identity. If omitted, Fastmail uses the default identity.

Choose the identity that matches the context of the email — for example, use a domain-specific address when replying to email sent to that domain.

## Send a New Email

```bash
fastmail-cli send \
  --to "alice@example.com" \
  --subject "Hello" \
  --body "Message body here."
```

### Multiple recipients

```bash
# Comma-separated in --to
fastmail-cli send \
  --to "alice@example.com, bob@example.com" \
  --subject "Team update" \
  --body "Here's the latest..."
```

### With CC and BCC

```bash
fastmail-cli send \
  --to "alice@example.com" \
  --cc "bob@example.com" \
  --bcc "archive@example.com" \
  --subject "Proposal" \
  --body "Please see the attached proposal."
```

### From a specific identity

```bash
fastmail-cli send \
  --to "client@example.com" \
  --from "work@yourdomain.com" \
  --subject "Following up" \
  --body "Just checking in on the project timeline."
```

## Reply

Always `get` the email first to confirm context:

```bash
# Read the email
fastmail-cli get EMAIL_ID | jq '{subject: .data.subject, from: .data.from[0].email, body: (.data.bodyValues | to_entries[0].value.value)}'

# Reply to sender only
fastmail-cli reply EMAIL_ID --body "Thanks, I'll look into this."

# Reply-all (includes all original recipients)
fastmail-cli reply EMAIL_ID --body "Thanks everyone." --all

# Reply with additional CC
fastmail-cli reply EMAIL_ID --body "See you then." --cc "manager@example.com"

# Reply from a specific identity
fastmail-cli reply EMAIL_ID --body "Thanks." --from "work@yourdomain.com"
```

## Forward

```bash
# Read the email first
fastmail-cli get EMAIL_ID | jq '{subject: .data.subject, from: .data.from[0].email}'

# Forward
fastmail-cli forward EMAIL_ID \
  --to "colleague@example.com" \
  --body "FYI — see below."

# Forward to multiple recipients
fastmail-cli forward EMAIL_ID \
  --to "alice@example.com, bob@example.com" \
  --body "Sharing this for your awareness."

# Forward from a specific identity
fastmail-cli forward EMAIL_ID \
  --to "team@example.com" \
  --from "work@yourdomain.com" \
  --body "Forwarding for the team."
```

## Body Text Tips

- Body is plain text; newlines in the `--body` argument are preserved
- For multi-line bodies in shell, use a variable:

```bash
BODY="Hi Alice,

Thanks for reaching out. I'll follow up by end of week.

Best regards"

fastmail-cli reply EMAIL_ID --body "$BODY"
```

## Common Flags

| Flag | Description |
|------|-------------|
| `--to ADDR` | Recipient(s), comma-separated |
| `--from ADDR` | Sending identity (email address) |
| `--cc ADDR` | CC recipient(s) |
| `--bcc ADDR` | BCC recipient(s) |
| `--subject TEXT` | Email subject (`send` only) |
| `--body TEXT` | Message body |
| `--all` | Reply-all (`reply` only) |

## Verifying Success

```bash
fastmail-cli send --to "..." --subject "..." --body "..." | jq '{success: .success, message: .message}'
```

A successful send returns `"success": true`. On failure, `.error` contains the reason.

Refresh `~/.local/share/fastmail-cli-agent/inbox-cache.jsonl` after any action that changes mailbox contents.
