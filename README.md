# AI Stack Blueprints

Opinionated scaffolding prompts for AI coding agents. Feed a blueprint prompt to an AI agent and it will scaffold a complete, production-ready application — infrastructure, backend, frontend, and CI/CD.

## Available Stacks

### .NET Full-Stack on AWS

**`DOTNET_STACK_BLUEPRINT_PROMPT.md`** — Scaffolds a complete .NET application:

| Layer | Technology |
|-------|-----------|
| Backend | ASP.NET Core Web API on AWS Lambda |
| Web Client | Blazor WebAssembly (standalone) |
| Mobile Client | .NET MAUI (Android, iOS, macOS, Windows) |
| Database | DynamoDB (single-table, default) or PostgreSQL (Aurora Serverless v2) |
| Infrastructure | AWS CDK in C# |
| CI/CD | GitHub Actions |
| Auth | JWT with BCrypt, auto-refresh token flow |

### Feature Modules

Optional modules in `dotnet/` that the blueprint pulls in based on project requirements:

| Module | File |
|--------|------|
| Stripe Payments | `dotnet/STRIPE_MODULE.md` |
| S3 File Storage | `dotnet/S3_STORAGE_MODULE.md` |
| SES Email | `dotnet/EMAIL_SES_MODULE.md` |
| Push Notifications | `dotnet/PUSH_NOTIFICATIONS_MODULE.md` |
| Background Processing | `dotnet/BACKGROUND_PROCESSING_MODULE.md` |
| Shareable Links | `dotnet/SHARE_LINKS_MODULE.md` |
| Report/PDF Generation | `dotnet/REPORT_GENERATION_MODULE.md` |
| Geolocation | `dotnet/GEOLOCATION_MODULE.md` |
| Scheduled Tasks | `dotnet/SCHEDULED_TASKS_MODULE.md` |

## Usage

1. Open a new conversation with an AI coding agent (Claude, Cursor, etc.)
2. Paste or attach `DOTNET_STACK_BLUEPRINT_PROMPT.md` as context
3. The agent will ask clarifying questions about your project (domain entities, database choice, client platforms, integrations)
4. Based on your answers, tell it which `dotnet/*.md` module files to include
5. The agent scaffolds the full project in order: Infrastructure → Server → Web/Mobile → CI/CD
6. A `CLAUDE.md` is generated last, documenting the scaffolded project's actual architecture

## Adding a New Stack

To add a blueprint for another technology stack:

1. Create a top-level `{STACK}_STACK_BLUEPRINT_PROMPT.md` with the master scaffolding prompt
2. Create a `{stack}/` directory for optional feature modules
3. Follow the same pattern: clarifying questions → modular architecture → placeholder-driven code generation
