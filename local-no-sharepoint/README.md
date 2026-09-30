# Interim variants (no SharePoint / no Microsoft access)

Two ways to run the TVM overdue-tickets workflow without SharePoint or Microsoft Graph
credentials — useful for validating the parse/filter/output logic while Entra ID access is
being provisioned. Both produce the same columns and filtering; they differ only in how the
HTML gets in and the Excel comes out.

## Which one?
- **`tvm-overdue-local.workflow.json`** — reads a local HTML file and writes a local `.xlsx`.
  Use when n8n runs on the **same machine** as your files (the file nodes act on the n8n
  host, and since n8n 2.0 are restricted to `~/.n8n-files`).
- **`tvm-overdue-form-upload.workflow.json`** — you upload the HTML in your **browser** and
  the filtered Excel downloads back. Use when n8n runs on a **server** and the files are on
  your own PC (no server file access or `N8N_RESTRICT_FILE_ACCESS_TO` needed).

## Common setup
- In the **Config** node, set the `ownerList` (prefilled with the current owners).
- Columns: Overdue Tickets, Ticket URL, Severity, Assignment, State, Summary. Ticket + URL
  are plain text here — a clickable `=HYPERLINK` needs the production SharePoint/Graph
  version in the repo root.

## Local file version
1. Put your HTML inside `~/.n8n-files` (Docker: `/home/node/.n8n-files`), or set
   `N8N_RESTRICT_FILE_ACCESS_TO=/your/dir` and restart n8n.
2. Set `inputHtmlPath` / `outputXlsxPath` in the Config node.
3. Click **Execute workflow**.

## Form-upload version
1. Open the "On form submission" node and copy its form URL (Test URL while editing;
   Production URL once the workflow is Active).
2. In your browser, open it, pick the dashboard `.html`, and submit.
3. The Excel downloads automatically.

For the production SharePoint/Graph version, see the repository root.
