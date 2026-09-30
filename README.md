# TVM overdue tickets → SharePoint Excel (n8n)

An n8n 2.x workflow that keeps a SharePoint Excel table in sync with the daily TVM
vulnerability dashboard. Every 15 minutes it picks up the newest
`SLA_Flow_Stock_YYYY-MM-DD.html` from the source SharePoint library, keeps the **Overdue**
tickets assigned to a set of owners, and rewrites the destination Excel table so it mirrors
exactly that set (resolved/closed tickets fall off).

## Files
- `tvm-overdue-sync.workflow.json` — import into n8n (Workflows → ⋯ → Import from File)
- `docs/build-kit.md` — node-by-node reference: CSS selectors, Code snippets, Graph clear call

## How it works
Schedule → list source folder → pick newest file (skips if already processed) →
clear the destination table body → download + parse the HTML → keep Overdue rows for the
owner list → append rows back (ticket written as `=HYPERLINK(...)` so it stays clickable).

SharePoint/Excel operations use the Microsoft Graph API, so the workflow needs a single
credential and no per-node resource pickers.

## Prerequisites
- An n8n 2.x instance.
- A Microsoft Entra app registration with the Graph **application** permission
  **Sites.Selected**, authorized on both SharePoint sites (source + destination).

## Setup (after import)
1. **Credential** — create an *OAuth2 API* credential and attach it to the five HTTP nodes
   (List folder, Get table range, Clear table rows, Download file, Append to table):
   - Grant Type: Client Credentials
   - Access Token URL: `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`
   - Client ID / Secret: from the app registration
   - Scope: `https://graph.microsoft.com/.default`
2. **Config node** — fill in:
   - `srcSiteId` / `dstSiteId` — e.g. `GET /sites/<tenant>.sharepoint.com:/sites/<SiteName>?$select=id`
   - `srcFolderPath` — source folder in the site's default document library
   - `dstItemId` — the workbook's driveItem id
   - `dstSheet` / `dstTable` — worksheet name + Excel table name
   - `ownerList` — already populated

## Notes
- Filter = SLA `Overdue` AND owner in `ownerList` (all severities, no assignment-group gate).
- "Clear table rows" empties the table body each run for a true full sync; it continues on
  error for the already-empty case.
- The per-row selectors in "Extract fields" assume the dashboard's current column order.
- Run manually once (schedule off) to validate before enabling the timer.

## Related
- [`local-no-sharepoint/`](local-no-sharepoint/) — offline version that reads/writes local files (no SharePoint or Microsoft credentials), handy for testing the parse/filter logic before access is provisioned.
