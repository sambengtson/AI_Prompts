# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of AI agent prompts for scaffolding full-stack applications on specific technology stacks. Currently only contains a **.NET** stack blueprint. There is no runnable application code — only markdown prompt files.

- `DOTNET_STACK_BLUEPRINT_PROMPT.md` — Master scaffolding prompt for .NET full-stack apps
- `dotnet/*.md` — Optional feature module prompts (Stripe, S3, SES, push notifications, etc.)

## How the Prompts Work Together

The master prompt defines a scaffolding workflow: ask clarifying questions → pull in relevant `dotnet/*.md` modules based on answers → replace placeholders → scaffold Infrastructure → Server → Web/Mobile → CI/CD → CLAUDE.md.

The master prompt produces projects with this architecture:
- **Server**: ASP.NET Core Web API with dual Lambda/Kestrel entry points, DynamoDB single-table (or Postgres/Aurora Serverless v2), JWT auth, interface+implementation service pattern
- **Web**: Blazor WebAssembly with custom `AuthStateProvider`, auto-refresh token flow, Bootstrap 5
- **Mobile**: .NET MAUI with MVVM (CommunityToolkit.Mvvm), Shell navigation, SecureStorage tokens
- **Infrastructure**: AWS CDK in C# with independent stacks (Secrets, Network, Database, Web) and dependent stacks (Api)

## Editing Guidelines

- Maintain placeholder syntax: `{ProjectName}` (PascalCase), `{projectname}` (lowercase), `{DomainEntity}`, `{DotNetVersion}` — the master prompt's find-and-replace workflow depends on these exact tokens
- Module files must be self-contained — each describes server additions, infrastructure additions, client additions, and NuGet packages for that feature
- The master prompt's Section 12 module table must stay in sync with the actual `dotnet/*.md` files
- Database choice (DynamoDB vs Postgres) affects Server models, services, Infrastructure stacks, and CDK dependencies — changes must be reflected across all relevant sections
- Code examples in prompts should be complete enough to copy-paste but minimal enough to adapt
