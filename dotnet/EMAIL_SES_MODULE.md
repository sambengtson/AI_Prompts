# Module: Transactional Email (SES)

Adds email sending capability via Amazon Simple Email Service.

---

## Server Additions

### EmailService

Core operations:
- Send HTML email to a single recipient
- Send templated email (password reset, welcome, notifications, etc.)
- Build HTML email body from templates with variable substitution

### Email Templates

Create HTML email templates as embedded resources or string constants. Common templates:
- **Password reset** — Contains a reset link with token
- **Welcome** — Sent after signup
- Add domain-specific templates as needed

### NuGet Package

```xml
<PackageReference Include="AWSSDK.SimpleEmail" />
```

---

## Infrastructure Additions

### ApiStack Update

- Register `IAmazonSimpleEmailService` as a singleton in `Startup.cs`
- Grant SES send permissions to the Lambda execution role:
  - `ses:SendEmail`, `ses:SendRawEmail`

Key decisions:
- **Verified identity**: Verify a domain (recommended for production) or individual email addresses (sufficient for dev). SES starts in sandbox mode — request production access when ready
- **Template management**: Store templates in code (simplest), in S3, or use SES templates (`CreateTemplate` API). Code-based templates are easiest to version control
- **From address**: Use a no-reply address (e.g., `noreply@{domain}`) or a branded address

---

## Configuration

Add to `AppSettings`:
- `SesFromEmail` — The verified sender email address
- Populated from `SES_FROM_EMAIL` environment variable (set in CDK ApiStack)

---

## File Additions

```
Server/src/Server/
  Services/
    IEmailService.cs
    EmailService.cs
```
