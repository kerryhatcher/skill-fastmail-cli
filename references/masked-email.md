# Masked Email

Masked email addresses are disposable aliases that forward to your real inbox. Useful for signups, services, or any situation where you want to keep your real address private.

Append a JSONL audit record to `~/.local/share/fastmail-cli-agent/actions.jsonl` for each masked-email create, enable, disable, or delete action.

## States

| State | Behaviour |
|-------|-----------|
| `pending` | Initial state. Auto-deleted 24h after creation if no mail is received. Not shown in Fastmail UI. |
| `enabled` | Active — mail is received normally. |
| `disabled` | Active address but mail goes straight to trash. |
| `deleted` | Inactive — mail sent to it is bounced. |

Once moved out of `pending`, an address cannot go back to `pending`.

## List Masked Emails

```bash
fastmail-cli masked list
```

```bash
# Compact view: id, email, state, description
fastmail-cli masked list | jq '.data[] | {id, email, state, description, forDomain}'

# Only enabled addresses
fastmail-cli masked list | jq '.data[] | select(.state == "enabled") | {id, email, description, lastMessageAt}'

# Only disabled addresses
fastmail-cli masked list | jq '.data[] | select(.state == "disabled") | {id, email, description}'

# Find by domain
fastmail-cli masked list | jq '.data[] | select(.forDomain | contains("example.com")) | {id, email, state}'

# Count by state
fastmail-cli masked list | jq '[.data[] | .state] | group_by(.) | map({state: .[0], count: length})'
```

## Create a Masked Email

```bash
# For a specific website (recommended)
fastmail-cli masked create \
  --domain "https://example.com" \
  --description "Example Site signup"

# With a custom prefix (address will start with your prefix)
fastmail-cli masked create \
  --prefix "shopping" \
  --description "Online shopping sites"

# Minimal creation (server generates a random address)
fastmail-cli masked create --description "Newsletter signup"
```

The created email address is returned in `.data.email`:

```bash
fastmail-cli masked create --domain "https://example.com" --description "Example Site" | jq '.data.email'
```

### Field guidelines

- `--domain`: The origin (`https://example.com`) of the site — no path, just scheme + domain
- `--description`: What this address is for — shown in Fastmail UI next to the alias
- `--prefix`: Optional; max 64 characters, only `a-z`, `0-9`, `_`

## Enable / Disable

```bash
# Re-enable a disabled address
fastmail-cli masked enable MASKED_EMAIL_ID

# Disable (mail goes to trash instead of inbox)
fastmail-cli masked disable MASKED_EMAIL_ID
```

Get the ID from `fastmail-cli masked list | jq '.data[] | {id, email}'`.

## Delete

```bash
# Prompts for confirmation
fastmail-cli masked delete MASKED_EMAIL_ID

# Skip confirmation
fastmail-cli masked delete MASKED_EMAIL_ID -y
```

Deleted addresses bounce all incoming mail. This cannot be undone.

## Workflow: Create and Use a Masked Email for a Signup

```bash
# 1. Create the alias
fastmail-cli masked create \
  --domain "https://somesite.com" \
  --description "Somesite account" | jq '{email: .data.email, state: .data.state}'

# 2. Use the returned email address during signup
# (address starts as "pending" — first received email activates it to "enabled")

# 3. Later, check last message received
fastmail-cli masked list | jq '.data[] | select(.forDomain | contains("somesite.com")) | {email, state, lastMessageAt}'

# 4. If the site starts spamming, disable it
fastmail-cli masked disable MASKED_EMAIL_ID
```
