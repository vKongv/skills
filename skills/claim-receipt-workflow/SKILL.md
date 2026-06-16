---
name: claim-receipt-workflow
description: Use when sourcing, verifying, uploading, correcting, or auditing claim receipts against a Google Sheet and Drive folder. Enforces preflight checks, local audit artifacts, receipt-quality standards, safe Sheet/Drive mutations, and resumable AGENTS.md state.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [claims, receipts, google-workspace, drive, sheets, audit]
    related_skills: [google-workspace]
---

# Claim Receipt Workflow

## Overview

Use this skill for end-to-end claim receipt projects where an agent must match claim rows in a spreadsheet to receipt evidence, save a local audit trail, upload verified receipts to Drive, update the sheet, and handle later accountant corrections safely.

This is a generic playbook. Do **not** hardcode personal emails, client domains, account IDs, spreadsheet IDs, OAuth tokens, or Drive file IDs into the skill. Keep project-specific facts in the workspace `AGENTS.md` and audit JSON files.

Core principles:

- Progressive preflight: discover what you can, ask only when blocked, but stop before mutations until required details and approval are present.
- Evidence first: save evidence locally before upload; verify vendor/date/amount/context before marking `FOUND`.
- Sheet safety: resolve current rows immediately before writing; never trust stale row numbers.
- Mutation discipline: upload/update only with user approval or explicit request; delete/trash Drive files only with explicit delete approval.
- Auditability: every claim run produces local JSON artifacts and updates `AGENTS.md` so future agents can resume.

## When to Use

Use this skill when the user asks to:

- source receipts/invoices/proofs of payment for claim rows;
- upload receipts to Google Drive and link them in a Google Sheet;
- correct accountant feedback where wrong or non-receipt files were linked;
- add or remove claim rows;
- audit claim amount formatting or Drive URLs;
- build a resumable receipt-claim workspace.

Do **not** use this skill for:

- one-off arithmetic only;
- general email management unrelated to claim evidence;
- accounting policy decisions that require the user's finance team;
- submitting/sending emails unless the user explicitly asks.

## Required Claim Packet

Before final Drive/Sheet mutations, obtain or discover:

```text
Claim Sheet
- spreadsheet URL or spreadsheet ID
- tab name
- header row and data start row
- columns for item number, date, description, category, amount, Drive URL/status

Drive Destination
- root folder URL/ID
- desired folder structure
- upload filename convention
- whether filenames should include or omit visible item numbers

Evidence Sources
- local folders/files provided by user
- Gmail/Workspace accounts to search, if any
- whether existing old evidence folders may be reused or must be ignored

Tooling/Auth
- gog installed and authenticated for required accounts
- Python PDF extraction available
- image/vision handling available for screenshots

Mutation Scope
- explicit permission for Drive uploads
- explicit permission for Sheet edits
- separate explicit permission for deleting/trashing old Drive files
```

Use a **progressive gate with hard stops before mutations**:

- You may read/search/download/save local evidence with partial context.
- Stop before Drive upload, Sheet update, row delete, or Drive delete unless the needed IDs/columns/rows and user approval are clear.

## Preflight Dependency Checks

Prefer `gog` for Google Workspace operations. Load environment first when the user has configured keyring/env secrets:

```bash
set -a; . "$HOME/.hermes/.env"; set +a
```

Check commands before starting heavy work:

```bash
which gog
/opt/homebrew/bin/gog --version || gog --version
```

Check authentication and access after the user provides accounts/resources:

```bash
/opt/homebrew/bin/gog auth status --account <google_account> --json --no-input
/opt/homebrew/bin/gog --account <google_account> drive ls --max 1 --json --no-input
/opt/homebrew/bin/gog --account <google_account> sheets get <spreadsheet_id> '<tab>!A1:Z5' --json --no-input
```

If a `gog` command fails or syntax drifts, run the relevant help before guessing:

```bash
/opt/homebrew/bin/gog --help
/opt/homebrew/bin/gog drive --help
/opt/homebrew/bin/gog sheets --help
```

Check PDF extraction:

```bash
python3 - <<'PY'
import pypdf
print('pypdf OK')
PY
```

For image receipts/screenshots, use a vision tool to read visible receipt facts. Do not infer details from filenames alone.

## Workspace and AGENTS.md Setup

Create or reuse a dedicated workspace:

```text
<workspace>/
  AGENTS.md
  raw/<operation-name>/
  evidence/item-<item-number-or-row-key>/
  outputs/
```

Always create or update `AGENTS.md` with:

- project purpose and current status;
- spreadsheet ID, tab, header/data rows, and column mapping;
- Drive root folder and folder convention;
- filename convention;
- approved mutation scope;
- required accounts described by role, not secrets;
- what has been completed and how to resume;
- warnings, exclusions, and finance/accountant decisions.

Never store tokens, refresh tokens, passwords, API keys, or OAuth secrets in `AGENTS.md` or summaries.

## Evidence Quality Rules

Classify each row as `FOUND`, `UNSURE`, or `NOT_FOUND`.

### FOUND

Use `FOUND` only when evidence verifies:

- vendor matches the claim row;
- document is an actual receipt, invoice, payment proof, statement, or verified screenshot accepted for the claim;
- amount and currency match exactly or plausibly map to the card-posted claim amount;
- date semantics match: invoice/receipt/payment date, service period, and/or card-posting date are explainable;
- bill-to/account/entity context is plausible for the claim.

### UNSURE

Use `UNSURE` when:

- only email body text or HTML was found;
- a hosted receipt link expired or requires dashboard login;
- one key fact is missing;
- amount mismatch is explainable but not proven;
- evidence is related but not strong enough for final submission.

Email `.txt` / `.html` evidence is supporting evidence, **not final receipt evidence by default**. Save it locally, mark `UNSURE`, and ask the user whether to accept it, provide the actual receipt, leave blank, or mark `NOT_FOUND`. Upload/link email text only if the user explicitly approves that item or class of items.

### NOT_FOUND

Use `NOT_FOUND` when:

- no valid evidence exists after reasonable searches;
- evidence belongs to another company/entity;
- only wrong, duplicate, irrelevant, or zero-amount receipts exist;
- vendor dashboard/email cannot recover a receipt and the user has not provided one.

## Row Resolution Rules

Rows shift after deletions/insertions. Before every Sheet write, re-read the sheet and resolve targets from current visible values.

Resolution priority:

1. Visible item number / No. column if user says `#NN` and it is unique.
2. Exact date + description + amount.
3. Description + amount + nearby date.
4. If multiple matches remain, stop and ask.

Treat `#NN` as the visible claim item number by default, **not** a physical sheet row number.

Never update a cell solely because an earlier snapshot said that physical row used to contain the item.

## Local Evidence and JSON Artifacts

Save every source file locally before upload. Do not upload directly from `Downloads` without copying into the workspace evidence folder.

Required minimal item result shape, extra fields allowed:

```json
{
  "item_number": "123",
  "current_sheet_row": 456,
  "date": "DD/MM/YYYY",
  "description": "<claim description>",
  "category": "<category>",
  "amount_claimed": "RM123.45",
  "status": "FOUND | UNSURE | NOT_FOUND",
  "evidence_files": [
    {
      "local_path": "evidence/item-123/receipt.pdf",
      "type": "invoice | receipt | screenshot | email_text | statement",
      "vendor": "<vendor>",
      "document_id": "<invoice_or_receipt_id>",
      "document_date": "YYYY-MM-DD",
      "amount_original": "USD 10.00",
      "amount_claimed": "RM43.10",
      "verification_notes": "Why this matches"
    }
  ],
  "drive_urls": [],
  "warnings": []
}
```

Each operation should write at least:

```text
raw/<operation>/
  preflight.json
  sheet_snapshot_before.json
  row_targets.json
  evidence_verification.json
  upload_manifest.json
  upload_results.json
  sheet_updates.json
  sheet_snapshot_after.json
  verification_summary.json
```

For deletion/trashing operations also write:

```text
raw/<operation>/deleted_old_drive_files.json
```

## Sourcing Strategy

1. Read current sheet target rows and normalize item/date/description/category/amount/current URL.
2. Search user-provided local files/folders first when given.
3. Search Gmail/Drive accounts by vendor, amount, invoice ID, sender, subject, card last-4, and date window.
4. Download actual attachments or generated PDFs when available.
5. For hosted links, try to fetch fresh links from email or ask user to retrieve from vendor dashboard.
6. Save all discovered evidence locally with notes even if `UNSURE`.
7. Prefer actual receipt/invoice/payment proof over email text.

Use the sheet date as card-posting date for matching, but document invoice/receipt/payment/service dates separately.

## Subagent Strategy

Use parallel subagents for bulk sourcing and verification only. The parent/orchestrator owns shared mutations.

Good subagent work:

- Gmail searches by vendor/month/account;
- local folder inspection;
- PDF text extraction and semantic verification;
- screenshot/image receipt description;
- per-item JSON result files;
- `FOUND` / `UNSURE` / `NOT_FOUND` classification.

Keep centralized in the parent:

- Drive folder creation;
- upload manifest finalization;
- Drive uploads when possible;
- Sheet URL writes;
- row insertion/deletion;
- Drive deletion/trashing;
- amount-column audits;
- final `AGENTS.md` updates.

If subagents are allowed to upload/update, give them narrow batches, require exact JSON results, and verify their side effects from the parent before reporting success.

## Upload and Naming Convention

Default new-upload filename convention:

```text
YYMMDD - Vendor - Short Description - DocumentID - RMxx.xx.ext
```

Do **not** include `#NN` by default.

Filename date priority:

1. invoice date / receipt issue date;
2. receipt date;
3. payment/paid date;
4. sheet/card-posting date only as fallback.

If one row needs multiple evidence files, upload all files needed to prove the row. Put multiple Drive URLs in the Sheet URL cell separated by newlines.

When both invoice and receipt exist, upload both unless they are exact duplicates.

If each file has a distinct document ID, no numeric suffix is needed. If not, add `-01`, `-02`, etc. before the extension.

Sanitize filenames:

- no slashes, colons, control characters, or excessive length;
- preserve original extension (`.pdf`, `.png`, `.jpg`, etc.);
- use the claim row amount for the final `RMxx.xx` segment;
- verify uploaded Drive metadata and assert the filename contains no `#` unless the user changed the convention.

## Drive Folder and Upload Commands

Use project-specific folder conventions from the claim packet, for example:

```text
<drive root>/<year>/<category>/<vendor>/
```

Helpful `gog` recipes:

```bash
# Search for a folder under a parent. Put the parent clause in the raw query.
/opt/homebrew/bin/gog --account <google_account> drive search \
  "name = '<folder_name>' and mimeType = 'application/vnd.google-apps.folder' and trashed = false and '<parent_folder_id>' in parents" \
  --raw-query --max 10 --json --no-input

# Create a folder.
/opt/homebrew/bin/gog --account <google_account> drive mkdir '<folder_name>' \
  --parent <parent_folder_id> --json --no-input

# Upload a receipt.
/opt/homebrew/bin/gog --account <google_account> drive upload '<local_path>' \
  --name '<upload_filename>' --parent <folder_id> --json --no-input

# Verify metadata.
/opt/homebrew/bin/gog --account <google_account> drive get <file_id> --json --no-input
```

## Sheet Update Rules

Before writing:

1. Re-read the sheet.
2. Resolve current physical rows from visible item number/date/description/amount.
3. Build `sheet_updates.json`.
4. For small direct batches, execute after verification. For large/bulk batches, preview first unless user already gave clear bulk approval.

Suggested threshold:

- 1–5 rows: verify, upload, update, then report.
- 6+ rows or any destructive operation: preview first unless already explicitly approved.

Batch update recipe:

```json
[
  {"range":"<tab>!G35", "values":[["https://drive.google.com/file/d/.../view"]]},
  {"range":"<tab>!G36", "values":[["https://drive.google.com/file/d/.../view\nhttps://drive.google.com/file/d/.../view"]]}
]
```

```bash
/opt/homebrew/bin/gog --account <google_account> sheets batch-update \
  --data-json @sheet_updates.json \
  <spreadsheet_id> --json --no-input
```

After writing, re-read the affected rows and verify the URL cell exactly matches the new URLs.

## Deletion / Trash Rules

Drive deletion/trashing is destructive. Do it only when the user explicitly says to delete/trash old files.

Policy:

- Replacing a Sheet URL after upload is allowed with upload/update approval.
- Deleting/trashing previous Drive files requires explicit delete approval.
- If the user asks to delete old wrong uploads, resolve file IDs from the old Sheet URLs, fetch metadata for audit, then move them to trash.
- Prefer trash over permanent delete unless the user explicitly asks for permanent deletion.

Command:

```bash
/opt/homebrew/bin/gog --account <google_account> drive delete <file_id> \
  --force --json --no-input
```

Save `deleted_old_drive_files.json` with file ID, prior URL, metadata before delete, response, and status.

## Adding or Removing Rows

### Adding rows

Requires explicit user instruction.

- Determine insertion location from current sheet structure.
- Use the next sensible visible item number unless the user specifies one.
- Write numeric amount values, not `RM` text.
- Apply existing row formatting and amount number format.
- Link Drive URLs only after evidence is verified/uploaded.
- Verify formulas/subtotals after insertion.

### Removing rows

Requires explicit user instruction.

- Resolve current rows by visible item/date/description/amount.
- For one uniquely resolved row, direct deletion is acceptable if the user explicitly requested deletion.
- For multi-row deletion, preview exact rows first unless the user already approved that exact list.
- Delete bottom-up to avoid shifting.
- Verify search terms no longer appear.
- Re-check subtotal/formulas/amount formatting.

If `gog sheets` lacks row delete, use Google Sheets API `spreadsheets.batchUpdate` with `deleteDimension`. Zero-based, end-exclusive row indexes:

```json
{
  "requests": [
    {"deleteDimension":{"range":{"sheetId":0,"dimension":"ROWS","startIndex":23,"endIndex":24}}}
  ]
}
```

## Amount Column Handling

Amount cells must be numeric. `RM` should be display-only.

Normal workflow:

- Do a lightweight check of amount values.
- Do not rewrite the whole amount column unless needed.

Full audit/fix when:

- adding rows;
- removing rows;
- writing amounts;
- user explicitly asks for amount audit/fix.

Rules:

- Write `123.45` or `-706.66`, not `RM123.45` or `-RM706.66`.
- Apply number format like:

```text
"RM"#,##0.00;-"RM"#,##0.00
```

- Verify unformatted values are numeric.
- Verify subtotal formulas still work.

## Accountant Feedback / Color Markings

Ignore color-code workflows unless the user explicitly mentions colors, accountant markings, or color codes.

When triggered:

1. Read sheet formatting for the relevant range.
2. Build a color inventory with physical row, item number, date, description, category, amount, current URLs, and RGB/hex.
3. Ask/confirm what each color means unless the user already provided a legend.
4. Respect ignore instructions.
5. Save `color_inventory.json` under `raw/<operation>/`.
6. Re-resolve rows again before writing because rows may shift.

Do not rely on screenshots alone if API formatting is available.

## Mutation Approval Model

Discovery/local save is allowed once the user asks for receipt sourcing.

Require user approval or explicit request for:

- Drive upload;
- Sheet update;
- adding rows;
- removing rows.

Require separate explicit approval for:

- Drive file deletion/trashing;
- permanent deletion;
- broad/bulk destructive operations.

Execute small upload/update batches immediately when the user clearly says “upload”, “link”, “tag”, or “do the same”. Preview large batches or destructive changes unless already clearly approved.

## Verification Checklist

Before final response:

- [ ] Preflight checks completed or blockers reported.
- [ ] Workspace evidence copied locally before upload.
- [ ] `AGENTS.md` created/updated with project-specific state.
- [ ] Every `FOUND` item has vendor/date/amount/context verification notes.
- [ ] Email text/HTML-only evidence is `UNSURE` unless user approved it as final evidence.
- [ ] Sheet target rows were resolved from current sheet values immediately before writing.
- [ ] Upload filenames follow convention and contain no `#NN` unless requested.
- [ ] Drive metadata verifies uploaded filename and file ID.
- [ ] Sheet rows were re-read and URL cells match expected Drive URLs.
- [ ] Old Drive files were deleted/trashed only with explicit user approval.
- [ ] Amount cells remain numeric if rows/amounts were edited.
- [ ] JSON artifacts are saved under `raw/<operation>/`.
- [ ] Final chat summary cites real counts, rows/items, URLs, and audit path.

## Common Pitfalls

1. **Uploading email text as final receipt evidence.** Save locally and mark `UNSURE` unless the user explicitly accepts it.

2. **Using stale physical row numbers.** Always re-read and resolve current rows before writing.

3. **Treating `#NN` as a row number.** It usually means the visible claim item number.

4. **Hardcoding accounts or domains in the reusable skill.** Keep those in `AGENTS.md`, not the skill.

5. **Writing `RM` as text into amount cells.** Amounts must be numeric; `RM` belongs in number formatting.

6. **Deleting old Drive files without separate approval.** Replacing sheet links is not the same as deleting evidence.

7. **Letting subagents mutate shared Sheet/Drive state without verification.** Parallelize sourcing; centralize mutation.

8. **Assuming receipt date equals sheet date.** Sheet date may be card-posting date. Match using vendor/amount/date semantics and document the reasoning.

9. **Skipping local copies.** Always copy user-provided or downloaded files into workspace evidence before upload.

10. **Reporting success without verifying.** Re-read Sheet and Drive metadata after every mutation.
