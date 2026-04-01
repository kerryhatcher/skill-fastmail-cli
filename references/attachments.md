# Attachments

## Check for Attachments

Before downloading, check whether an email has attachments:

```bash
# In list output
fastmail-cli list emails --limit 20 | jq '.data.emails[] | select(.hasAttachment == true) | {id, subject, from: .from[0].email}'

# For a specific email
fastmail-cli get EMAIL_ID | jq '{hasAttachment: .data.hasAttachment}'
```

## Download Attachments

```bash
# Download to current directory
fastmail-cli download EMAIL_ID

# Download to a specific directory
fastmail-cli download EMAIL_ID --output ~/Downloads

# Download to a project folder
fastmail-cli download EMAIL_ID --output ./attachments
```

Downloaded files are saved with their original filenames.

## Text Extraction

Extract the text content of document attachments as JSON (useful for reading PDFs, DOCX, etc. without opening them):

```bash
fastmail-cli download EMAIL_ID --format json
```

Returns structured JSON with extracted text per attachment:

```json
{
  "success": true,
  "data": [
    {
      "name": "report.pdf",
      "contentType": "application/pdf",
      "size": 204800,
      "textContent": "Q1 Financial Report\n\nExecutive Summary..."
    }
  ]
}
```

```bash
# Extract and display text from all attachments
fastmail-cli download EMAIL_ID --format json | jq -r '.data[] | "=== \(.name) ===\n\(.textContent)"'

# Get just filenames and extracted text lengths
fastmail-cli download EMAIL_ID --format json | jq '.data[] | {name, size, textLength: (.textContent | length)}'
```

## Supported Text Extraction Formats

Text extraction via `--format json` supports 56 formats:

| Category | Formats |
|----------|---------|
| Documents | PDF, DOC, DOCX, ODT, RTF |
| Spreadsheets | XLS, XLSX, ODS, CSV, TSV |
| Presentations | PPT, PPTX |
| eBooks | EPUB, FB2 |
| Markup | HTML, XML, Markdown, RST, Org |
| Data | JSON, YAML, TOML |
| Email | EML, MSG |
| Archives | ZIP, TAR, GZ, 7z |
| Academic | BibTeX, LaTeX, Typst, Jupyter notebooks |

## Image Resizing

Limit image file sizes on download using `--max-size`:

```bash
# Resize images to max 500KB
fastmail-cli download EMAIL_ID --max-size 500K

# Resize images to max 1MB
fastmail-cli download EMAIL_ID --max-size 1M

# Download to a directory with resizing
fastmail-cli download EMAIL_ID --output ~/Downloads --max-size 500K
```

`--max-size` only affects image files. Non-image attachments are downloaded at their original size.

## Combined Download Options

```bash
# Extract text AND save to a directory
fastmail-cli download EMAIL_ID --format json --output ./docs

# Download with image resizing to a specific folder
fastmail-cli download EMAIL_ID --max-size 200K --output ./images
```

## Workflow: Read Attachment Content Without Saving

To read the text content of a document attachment inline (without writing to disk):

```bash
# Get full extracted text of first attachment
fastmail-cli download EMAIL_ID --format json | jq -r '.data[0].textContent'

# Summarize attachment names + sizes
fastmail-cli download EMAIL_ID --format json | jq '.data[] | {name, contentType, size}'
```
