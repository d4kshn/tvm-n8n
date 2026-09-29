# TVM dashboard → SharePoint Excel — n8n 2.x build kit

**Goal:** every ~15 min, find the newest `SLA_Flow_Stock_YYYY-MM-DD.html` in the source
library; if it's new since the last run, parse it, keep the **Overdue** tickets for your
owners, and rewrite the destination Excel **Table** so it mirrors exactly that set
(closed/resolved tickets fall out — "full sync").

---

## Coordinates to fill (⟨…⟩ = confirm/complete)

| | Value |
|---|---|
| Source site | `https://digitalrealty.sharepoint.com/sites/ServiceFabric` |
| Source library → folder | Shared Documents → `⟨Daily TVM Ticketing …⟩` |
| Source file pattern | `SLA_Flow_Stock_YYYY-MM-DD.html` |
| Dest site | `https://digitalrealty.sharepoint.com/sites/IPEngineering` |
| Dest workbook | `Network Engineering Overdue Tickets.xlsx` |
| Dest worksheet name | `⟨sheet name⟩` |
| Dest table name | `⟨table name⟩` |
| Table headers (row 1) | Overdue Tickets · Severity · Assignment · State · Summary |

**Credential (all Microsoft steps):** *Microsoft Entra Service Principal (App-Only)*.
The app registration needs Graph **application** permission **Sites.Selected**, authorized
on just these two sites (least privilege). The same credential is reused by the HTTP
Request node for the Graph clear call.

---

## Node pipeline

| # | Node | What it does |
|---|---|---|
| 1 | Schedule Trigger | every 15 min |
| 2 | Microsoft SharePoint | list the source folder's files |
| 3 | Code — "Pick newest + dedupe" | snippet A |
| 4 | Microsoft SharePoint | download the chosen file (binary); item id from step 3 |
| 5 | HTML (Extract) | Source = **Binary**; pull the rows as an array (see below) |
| 6 | Split Out | rows array → one item per row |
| 7 | Set — "Wrap row" *(safeguard)* | `<table>{{ $json.rowHtml }}</table>` |
| 8 | HTML (Extract) | Source = **JSON**; per-row field selectors (see below) |
| 9 | Filter | keep Overdue + owner ∈ list |
| 10 | Code — "Build rows + HYPERLINK" | snippet B |
| 11 | HTTP Request (Graph) | clear the table's data rows (snippet C) |
| 12 | Microsoft Excel (SharePoint) → Table: **Append** | map each column by name |

Notes:
- **Step 5 selector:** `table tbody tr` (or `table tr` if there's no `<tbody>` — header/summary
  rows will fall out at the Filter because they have no `a.tkt`). Return value = **HTML**,
  Return Array = **on**. Output field e.g. `rowHtml`.
- **Step 7** exists only because a bare `<tr>` can confuse the parser out of table context.
  If step 8's selectors return values without it, you can delete step 7.
- **Step 11 → 12 order:** clear first, then append. If step 12 runs per-item, keep the
  append as a single node that receives all rows (it appends them in one call).

---

## Step 8 — per-row selectors (one item = one `<tr>`)

| Field key | Selector | Return | Example |
|---|---|---|---|
| `ticket` | `td:nth-child(1) a.tkt` | text | `VUL0011607` |
| `url` | `td:nth-child(1) a.tkt` | attribute `href` | `https://servicenow.digitalrealty.com/nav_to.do?...` |
| `severity` | `td:nth-child(2)` | text | `Critical` |
| `state` | `td:nth-child(3)` | text | `Open` |
| `sla` | `td:nth-child(4)` | text | `Overdue` / `Within SLA` |
| `group` | `td:nth-child(5)` | text | `Network Engineering` |
| `owner` | `td:nth-child(6)` | text | `Julio Calderon` |

(Cells 7–10 are due date / age / SLA-days / days-overdue — not needed for the current table,
but they're there if you ever add columns.)

---

## Step 9 — Filter

Keep the row when **all** of these are true:
- `sla` **equals** `Overdue`
- `owner` **is in** your owner list

Optional extra conditions (add if you want them — see the two open questions):
- `group` equals `Network Engineering`
- `severity` equals `Critical`

Put the owner list in a **Set** node near the top (e.g. field `ownerList` =
`["Pedro Faria", "Julio Calderon"]`) and reference it in the Filter with
`{{ $('Config').first().json.ownerList.includes($json.owner) }}`.

---

## Snippet A — pick the newest file, skip if already processed
Code node · Mode: **Run Once for All Items**

```js
// Input: the file list from the SharePoint "list folder" node.
// Rename `f.name` below if your list node calls the field something else.
const files = $input.all().map(i => i.json);

const re = /SLA_Flow_Stock_(\d{4}-\d{2}-\d{2})\.html$/i;

const dated = files
  .map(f => {
    const m = String(f.name ?? '').match(re);
    return m ? { file: f, date: m[1] } : null;
  })
  .filter(Boolean)
  .sort((a, b) => (a.date < b.date ? 1 : -1)); // newest first

if (dated.length === 0) return [];

const newest = dated[0];
const store = $getWorkflowStaticData('global');
if (store.lastProcessedFile === newest.file.name) {
  return []; // nothing new since last run — stop here
}
store.lastProcessedFile = newest.file.name;
return [{ json: newest.file }];
```

---

## Snippet B — build the 5 columns with a clickable ticket
Code node · Mode: **Run Once for All Items**

```js
// Input: filtered rows from the Filter node.
// The keys read here (ticket, url, severity, state, owner) must match the
// field keys your step-8 HTML node produced — rename if needed.
return $input.all().map(item => {
  const j = item.json;
  const url = String(j.url ?? '').replace(/"/g, '""');   // escape quotes for the formula
  const ticket = String(j.ticket ?? '').trim();
  return {
    json: {
      'Overdue Tickets': `=HYPERLINK("${url}","${ticket}")`,
      'Severity': String(j.severity ?? '').trim(),
      'Assignment': String(j.owner ?? '').trim(),
      'State': String(j.state ?? '').trim(),
      'Summary': '',
    },
  };
});
```

The output keys match your table headers exactly, so the **Append** node's
"map each column by name" lines them up. A value starting with `=` is stored as a
**formula**, so the ticket renders as a clickable hyperlink. If your instance ever writes
it as literal text, switch to writing the range's `formulas` property via Graph instead.

---

## Snippet C — clear the table body before appending
HTTP Request node(s) · same Entra Service Principal credential (Graph)

Two requests:

**1) Get the data range**
```
GET https://graph.microsoft.com/v1.0/sites/{siteId}/drive/items/{itemId}/workbook/tables('{table}')/dataBodyRange?$select=address,rowCount
```

**2) If `rowCount` > 0, delete those rows (shift up)**
```
POST https://graph.microsoft.com/v1.0/sites/{siteId}/drive/items/{itemId}/workbook/worksheets('{sheet}')/range(address='{localAddress}')/delete
Body (JSON): { "shift": "Up" }
```

- `{localAddress}` = the `address` from request 1 with the `SheetName!` prefix removed
  (e.g. `Network Engineering!A2:E250` → `A2:E250`).
- Guard request 2 behind an IF on `rowCount > 0` (an empty table has no `dataBodyRange`).
- `{siteId}` / `{itemId}`: grab once from the SharePoint node's output (the file's
  `parentReference.siteId` and `id`) or from Graph Explorer. `{table}` / `{sheet}` are your
  names from the coordinates table.

This keeps the formal Table and its formatting/filtering intact — it just empties the body
so the following Append writes a clean, current set.

---

## Still to confirm
1. **Summary** column — leave blank, or populate it from something?
2. **Filter scope** — owner names for the list; and whether to also gate on
   `group = Network Engineering` and/or `severity = Critical`.

Once those are set, this can be packaged as a single importable workflow `.json`.
