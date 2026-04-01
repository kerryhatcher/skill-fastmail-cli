# Contacts

## Prerequisites

Contact search uses CardDAV, which requires an **app password** — API tokens do not work for CardDAV.

Generate an app password at: Fastmail Settings → Privacy & Security → App passwords

Set credentials via environment variables or config file:

**Environment variables:**
```bash
export FASTMAIL_USERNAME="you@fastmail.com"
export FASTMAIL_APP_PASSWORD="your-app-password"
```

**Config file** (`~/.config/fastmail-cli/config.toml`):
```toml
[contacts]
username = "you@fastmail.com"
app_password = "your-app-password"
```

> The `fastmail-cli auth` command only sets the API token. The `[contacts]` section must be added manually.

## List All Contacts

```bash
fastmail-cli contacts list
```

```bash
# Compact view: name + email
fastmail-cli contacts list | jq '.data[] | {name: .fullName, emails: [.emails[]?.value]}'

# Count total contacts
fastmail-cli contacts list | jq '.data | length'

# Find contacts with a phone number
fastmail-cli contacts list | jq '.data[] | select(.phones | length > 0) | {name: .fullName, phones: [.phones[]?.value]}'
```

## Search Contacts

Search by name, email address, or organization:

```bash
fastmail-cli contacts search "alice"
fastmail-cli contacts search "example.com"
fastmail-cli contacts search "Acme Corp"
```

```bash
# Get email address for a contact
fastmail-cli contacts search "alice" | jq '.data[] | {name: .fullName, emails: [.emails[]?.value]}'

# First email address only (for use in compose)
fastmail-cli contacts search "alice" | jq -r '.data[0].emails[0].value'
```

## Workflow: Look Up a Contact Before Composing

```bash
# 1. Find the contact
fastmail-cli contacts search "Bob Smith" | jq '.data[] | {name: .fullName, emails: [.emails[]?.value]}'

# 2. Use the address in send/reply
fastmail-cli send \
  --to "bob.smith@example.com" \
  --subject "Following up" \
  --body "Hi Bob, just checking in."
```

## Contact Data Structure

Contacts return fields like:

```json
{
  "fullName": "Alice Example",
  "emails": [{ "value": "alice@example.com", "type": "work" }],
  "phones": [{ "value": "+1-555-0100", "type": "mobile" }],
  "organization": "Example Corp",
  "notes": "Met at conference 2025"
}
```

Extract what you need with jq:

```bash
# All emails for a contact
fastmail-cli contacts search "alice" | jq '.data[0].emails[].value'

# Full contact card
fastmail-cli contacts search "alice" | jq '.data[0]'
```

## Troubleshooting

If contacts commands fail with an auth error:

1. Verify `FASTMAIL_USERNAME` and `FASTMAIL_APP_PASSWORD` are set
2. Confirm the app password was created specifically for CardDAV (not an API token)
3. Check the config file has a `[contacts]` section, not just `[core]`

```bash
# Verify env vars are set
echo "User: $FASTMAIL_USERNAME"
echo "Password set: $([ -n "$FASTMAIL_APP_PASSWORD" ] && echo yes || echo no)"
```
