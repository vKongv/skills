# Claim Receipt Workflow Design Notes

These notes capture user-approved decisions from a long claims-receipt workflow. They are examples and design rationale for the generic `claim-receipt-workflow` skill, not hardcoded project state.

## User-Approved Workflow Decisions

- Scope should be end-to-end: sourcing receipts, verifying evidence, uploading to Drive, linking Sheet rows, correcting accountant feedback, deleting wrong files only when approved, and auditing final state.
- Skill should be user-local and generic: do not hardcode personal email addresses or one company's specific account values into the skill.
- Start with a progressive preflight gate:
  - discovery can start with partial information;
  - mutations require complete Sheet/Drive context and approval;
  - check dependencies such as Google Workspace CLI/auth and PDF extraction before deep work.
- Actual receipts/invoices/screenshots are required by default for final claim evidence.
- Email `.txt`/`.html` evidence can be saved locally and used for investigation, but should be marked `UNSURE` and not uploaded/linked as final proof unless the user explicitly approves.
- If the user later provides actual receipts for email-text items, replace the Sheet links after verification. Ask whether to delete/trash old Drive files; delete only after explicit approval.
- Small direct batches can be executed immediately after verification. Large/bulk batches or destructive operations should be previewed unless the user gave clear bulk approval.
- New upload filenames should omit `#NN` by default.
- Filename date should use invoice/receipt date priority, not sheet/card-posting date by default:
  1. invoice date / receipt issue date;
  2. receipt date;
  3. payment/paid date;
  4. sheet/card-posting date as fallback.
- Upload invoice + receipt together when both exist, unless they are exact duplicates.
- Always save evidence locally before uploading.
- Every non-trivial workspace should have an `AGENTS.md` with claim context, sheet/folder IDs, progress, pending items, and resume instructions.
- Amount values should be numeric; `RM` should be display-only formatting. Do lightweight checks normally and full amount-column fixes when adding/removing rows, writing amounts, or with approval.
- Treat `#NN` in user messages as visible claim item number by default, not physical sheet row.
- Color-code/accountant-feedback handling is optional and should only run when the user explicitly mentions colors/color codes/accountant markings.
- For bulk sourcing, use subagents for search/verification but centralize final Drive/Sheet/delete mutations in the parent/orchestrator.

## Useful Patterns from the Session

### Replacing Email/Text Evidence

A robust replacement loop:

1. Read current Sheet rows and old Drive URLs.
2. Resolve target rows by visible item/date/description/amount, not stale row numbers.
3. Verify provided actual receipts using PDF extraction or image analysis.
4. Copy evidence into the workspace.
5. Upload new files using no-`#NN` filenames.
6. Batch-update Sheet URL cells with new Drive URLs.
7. Only after successful Sheet update, trash old non-receipt Drive files if the user explicitly requested deletion.
8. Verify row URL cells, new Drive metadata, and old file trash status.
9. Save `verification.json`, `uploads.json`, `sheet_updates.json`, `deleted_old_drive_files.json`, and `verification_summary.json`.

### Row Shift Recovery

When a script fails because a row no longer matches expected values, do not force the old row. Re-read the current sheet and resolve by stable visible values:

```text
date + description + amount
or visible item number + description/amount
```

This protects against prior row deletions/insertions.

### Accountant Feedback Corrections

If an accountant rejects evidence because it is not an actual receipt:

- Identify the affected rows and current Drive links.
- Inspect Drive filenames/extensions for `.txt`, `.html`, or `verification-summary` style files.
- Search/request actual receipt/invoice files.
- Replace links with actual evidence.
- Trash the old bad files only if the user explicitly asks.

### Google Workspace / `gog` Workflows

For Google Workspace tasks on the user's Mac, workflows often use `gog`; check for the CLI, auth, and required environment before running commands. Do not hardcode account emails in the skill; ask/discover the account per project.

## Do Not Persist as General Rules

- Do not encode one project's Sheet ID, Drive folder IDs, account emails, invoice numbers, or row numbers into the skill.
- Do not treat a particular color code as globally meaningful; color legends are project-specific.
- Do not assume all future claim sheets use the same columns; discover or ask for mapping.
