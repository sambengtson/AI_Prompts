# Module: Report / PDF Generation

Adds the ability to generate downloadable reports (PDF or other formats) and store them in S3.

**Prerequisite**: This module requires the S3 Storage module (`modules/S3_STORAGE_MODULE.md`).

---

## Server Additions

### ReportService

Core operations:
- Generate a report from domain data (query data, format into document)
- Store generated report in S3 with a structured key path (e.g., `reports/{userId}/{reportId}.pdf`)
- Generate presigned download URL for a stored report
- Optionally embed a verification code in the report for authenticity checking

Key decisions:
- **PDF library**: Choose based on requirements:
  - `QuestPDF` — Modern, fluent API, open-source (community license). Good for structured layouts
  - `iTextSharp` / `iText7` — Full-featured, AGPL or commercial license
  - `PdfSharpCore` — Lightweight, MIT license
- **Generation trigger**: On-demand (user requests a report via API) vs scheduled (EventBridge triggers periodic generation — see `modules/SCHEDULED_TASKS_MODULE.md`)
- **Lambda memory**: PDF generation can be memory-intensive. Consider increasing Lambda memory to 1024MB or higher if generating complex reports
- **Template approach**: Code-driven layout (QuestPDF fluent API) vs HTML-to-PDF conversion

### ReportsController

- Route: `/api/reports`
- `POST /generate` — Trigger report generation for the authenticated user
- `GET /` — List available reports for the authenticated user
- `GET /{reportId}/download` — Get presigned S3 URL to download a specific report
- `DELETE /{reportId}` — Delete a report

### NuGet Package (choose one)

```xml
<!-- QuestPDF (recommended) -->
<PackageReference Include="QuestPDF" />

<!-- OR PdfSharpCore -->
<PackageReference Include="PdfSharpCore" />
```

---

## DynamoDB Key Pattern (if using DynamoDB)

| Entity | PK | SK |
|--------|----|----|
| Report | `USER#{userId}` | `REPORT#{year}#{generatedAt:O}` |

---

## Infrastructure Considerations

- Ensure S3 Storage module is included for report file storage
- If reports are generated on a schedule, also include the Scheduled Tasks module
- Consider Lambda memory increase in `ApiStack` if PDF generation is resource-intensive

---

## File Additions

```
Server/src/Server/
  Controllers/
    ReportsController.cs
  Services/
    IReportService.cs
    ReportService.cs
  Models/
    GeneratedReport.cs
```
