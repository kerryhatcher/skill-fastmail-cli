# Compose: Send, Reply, Forward

Default compose behavior: save drafts in Fastmail with `--draft`. Only send immediately when the user explicitly instructs you to do so.

Draft support exists on all compose commands in this CLI:

- `send --draft`
- `reply --draft`
- `forward --draft`

If the user asks to create a draft, do exactly that. Do not convert a draft request into a send, and do not assume draft creation is unavailable without checking `--help`.

Use `references/email-writing.md` for message style. Drafts should sound like a real person replying in context, not a polished assistant.

## Safe Compose Pattern

Always follow this sequence before creating or sending anything:

1. **Read the email** you're responding to (`fastmail-cli get EMAIL_ID`)
2. **Summarize to the user** — subject, sender, key content
3. **Draft the response** with the user
4. **Confirm recipients, subject, and body** before executing
5. **Create a draft** with `--draft`
6. **Send immediately** only if the user explicitly asked for that

Never execute a send/reply/forward without explicit user confirmation of the final message. Unless the user explicitly says otherwise, execute compose actions as drafts.

When the user says things like "draft it", "create the draft", "make a reply draft", or "write the email", treat that as a draft request, not a send request.

When drafting copy:

- Prefer plain, natural wording over polished corporate phrasing
- Use contractions unless the user wants a formal tone
- Keep the tone specific to the relationship and context
- Avoid canned openings and closings that sound like generic assistant output
- Use concrete details from the thread instead of vague filler when you have them

Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for every send, reply, forward, or draft action.

## Identities

List available sending addresses before composing:

```bash
fastmail-cli list identities | jq '.data[] | {id, email}'
```

Use `--from EMAIL` to select the sending identity. If omitted, Fastmail uses the default identity.

Choose the identity that matches the context of the email — for example, use a domain-specific address when replying to email sent to that domain.

## Send a New Email

Default pattern:

```bash
fastmail-cli send \
  --to "alice@example.com" \
  --subject "Hello" \
  --body "Message body here." \
  --draft
```

### Multiple recipients

```bash
# Comma-separated in --to
fastmail-cli send \
  --to "alice@example.com, bob@example.com" \
  --subject "Team update" \
  --body "Here's the latest..." \
  --draft
```

### With CC and BCC

```bash
fastmail-cli send \
  --to "alice@example.com" \
  --cc "bob@example.com" \
  --bcc "archive@example.com" \
  --subject "Proposal" \
  --body "Please see the attached proposal." \
  --draft
```

### From a specific identity

```bash
fastmail-cli send \
  --to "client@example.com" \
  --from "work@yourdomain.com" \
  --subject "Following up" \
  --body "Just checking in on the project timeline." \
  --draft
```

If the user explicitly asks to send immediately, omit `--draft`.

Typical draft verification:

```bash
fastmail-cli send \
  --to "alice@example.com" \
  --subject "Hello" \
  --body "Message body here." \
  --draft | jq '{success, email_id: .data.email_id, status: .data.status}'
```

## Reply

Always `get` the email first to confirm context:

```bash
# Read the email
fastmail-cli get EMAIL_ID | jq '{subject: .data.subject, from: .data.from[0].email, body: (.data.bodyValues | to_entries[0].value.value)}'

# Draft reply to sender only
fastmail-cli reply EMAIL_ID --body "Thanks, I'll look into this." --draft

# Draft reply-all (includes all original recipients)
fastmail-cli reply EMAIL_ID --body "Thanks everyone." --all --draft

# Draft reply with additional CC
fastmail-cli reply EMAIL_ID --body "See you then." --cc "manager@example.com" --draft

# Draft reply from a specific identity
fastmail-cli reply EMAIL_ID --body "Thanks." --from "work@yourdomain.com" --draft
```

If the user explicitly asks to send immediately, omit `--draft`.

Typical reply-draft verification:

```bash
fastmail-cli reply EMAIL_ID \
  --body "Thanks, I'll look into this." \
  --draft | jq '{success, email_id: .data.email_id, in_reply_to: .data.in_reply_to, status: .data.status}'
```

Use `--all --draft` when the user wants to reply to the full thread instead of only the sender.

## Forward

```bash
# Read the email first
fastmail-cli get EMAIL_ID | jq '{subject: .data.subject, from: .data.from[0].email}'

# Draft forward
fastmail-cli forward EMAIL_ID \
  --to "colleague@example.com" \
  --body "FYI - see below." \
  --draft

# Draft forward to multiple recipients
fastmail-cli forward EMAIL_ID \
  --to "alice@example.com, bob@example.com" \
  --body "Sharing this for your awareness." \
  --draft

# Draft forward from a specific identity
fastmail-cli forward EMAIL_ID \
  --to "team@example.com" \
  --from "work@yourdomain.com" \
  --body "Forwarding for the team." \
  --draft
```

If the user explicitly asks to send immediately, omit `--draft`.

Typical forward-draft verification:

```bash
fastmail-cli forward EMAIL_ID \
  --to "colleague@example.com" \
  --body "FYI - see below." \
  --draft | jq '{success, email_id: .data.email_id, status: .data.status}'
```

## Body Text Tips

- Body is plain text; newlines in the `--body` argument are preserved
- Keep drafts concise unless the user asks for a longer message
- Match the sender's context and level of formality
- Read the draft once for AI tells before creating it: over-formal wording, repetitive sentence rhythm, empty filler, and generic transitions
- For multi-line bodies in shell, use a variable:

```bash
BODY="Hi Alice,

Thanks for reaching out. I'll follow up by end of week.

Best regards"

fastmail-cli reply EMAIL_ID --body "$BODY" --draft
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
| `--draft` | Save to Fastmail Drafts instead of sending |

## Verifying Success

```bash
fastmail-cli send --to "..." --subject "..." --body "..." --draft | jq '{success: .success, message: .message}'
```

A successful draft or send returns `"success": true`. Draft responses also include an email id and usually `"status": "draft"`.
On failure, `.error` contains the reason.

For compose actions, prefer surfacing the fields the user cares about:

```bash
fastmail-cli reply EMAIL_ID --body "..." --draft | jq '{success, email_id: .data.email_id, status: .data.status}'
```

Refresh `~/.local/share/fastmail-cli-agent/inbox-cache.jsonl` after any action that changes mailbox contents.
