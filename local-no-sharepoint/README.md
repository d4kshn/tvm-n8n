# Local variant (no SharePoint / no Microsoft access)

An offline version of the TVM overdue-tickets workflow that reads a **local** dashboard
HTML file and writes a **local** Excel file — no SharePoint, no Microsoft Graph, no
credentials. Useful for validating the parse/filter/output logic while Entra ID access is
being provisioned.

## File
- `tvm-overdue-local.workflow.json` — import into n8n (Workflows → ⋯ → Import from File)

## How it works
Manual trigger → read the local HTML → parse rows → keep Overdue rows for the owner list →
build columns → write a local `.xlsx` (create, or overwrite if it exists).

## Setup
1. Since n8n 2.0 the file nodes are restricted to `~/.n8n-files` (Docker:
   `/home/node/.n8n-files`). Put your HTML there and set the output path there too, or set
   `N8N_RESTRICT_FILE_ACCESS_TO=/your/dir` (semicolon-separated for several) and restart n8n.
2. In the **Config** node set `inputHtmlPath`, `outputXlsxPath`, and the `ownerList`.
3. Click **Execute workflow**.

## Notes
- Columns: Overdue Tickets, Ticket URL, Severity, Assignment, State, Summary. Ticket + URL
  are plain text here — a clickable `=HYPERLINK` needs the online/Graph version.
- Regenerates the whole file each run (same full-sync behaviour).
- For the production SharePoint/Graph version, see the repository root.
