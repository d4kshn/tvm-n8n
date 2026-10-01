# Weekly reports

Two upload-based workflows that turn a week of `tvm-overdue` daily Excel files into a report.
No credentials needed for the first; the second uses Azure OpenAI.

## Files
- `tvm-weekly-report.workflow.json` — per-owner change table (Start / End / Net / New overdue /
  Cleared), comparing the first and last day of the uploaded week. Downloads an Excel file.
- `tvm-weekly-ai-report.workflow.json` — parses every day of the week, computes the per-owner
  stats, and has **Azure OpenAI** write a narrative summary; downloads a styled HTML report.

## How they work
Both are form-upload workflows: open the form URL, upload the week's dated Excel files, and the
report downloads back. Files are ordered by the yyyy-mm-dd date in each filename.

## Setup
- Both: run via the form URL (the response is a Form Ending node), not "Execute step".
- AI report only: on the "Azure OpenAI Chat Model" node, select your Azure AI Foundry
  credential, Project, and chat deployment (e.g. gpt-4o). The numbers are computed in n8n; the
  model only writes the prose.

## Notes
- Expect files produced by the tvm-overdue workflows (columns "Overdue Tickets" and
  "Assignment"), named with the source dashboard's date.
