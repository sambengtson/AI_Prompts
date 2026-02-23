# Module: Scheduled Tasks (EventBridge Scheduler)

Adds the ability to create deferred or recurring tasks via Amazon EventBridge Scheduler.

---

## Server Additions

### SchedulerService

Core operations:
- Create a one-time schedule (execute at a specific future datetime)
- Create a recurring schedule (cron or rate expression)
- Delete/cancel a scheduled task
- List active schedules for a user or resource

The scheduler invokes a separate Lambda function (Notification Lambda) rather than the main API Lambda. This keeps scheduled work isolated and avoids API Gateway overhead.

Key decisions:
- **One-time vs recurring**: One-time schedules are deleted after execution. Recurring schedules repeat on a cron/rate basis
- **Retry policy**: Configure `MaximumRetryAttempts` (default 0-185) and `MaximumEventAgeInSeconds` for failed invocations
- **Flexible time window**: Use `OFF` for exact timing, or set a window (e.g., 15 minutes) to allow the scheduler to optimize delivery
- **Payload**: The schedule passes a JSON payload to the target Lambda containing context needed to perform the work (user ID, resource ID, action type, etc.)

### NuGet Package

```xml
<PackageReference Include="AWSSDK.Scheduler" />
```

---

## Infrastructure Additions

### NotificationStack (new stack)

- **Notification Lambda**: 256MB memory, 30s timeout, standalone handler (not ASP.NET Core — just a simple `Function.cs` with `FunctionHandler` method)
  - Receives the scheduled event payload and performs the deferred work (send notification, generate report, etc.)
  - Environment variables: `TABLE_NAME`, relevant secret ARNs
  - IAM: DynamoDB read/write, plus permissions for the actual work (SES send, push notification API, etc.)
- **Scheduler group**: `{projectname}-scheduler-group-{stage}` — groups all schedules for this project
- **Scheduler execution role**: IAM role that EventBridge Scheduler assumes to invoke the Notification Lambda
  - Grants: `lambda:InvokeFunction` on the Notification Lambda ARN
- Depends on `ApiStack` (or at minimum, `DatabaseStack`)
- Expose: `SchedulerGroupName`, `SchedulerRoleArn`, `NotificationLambda`

### CDK Entry Point Update

Add `NotificationStack` as a dependent stack after `ApiStack`. Pass the Notification Lambda ARN and Scheduler role to the `ApiStack` (or pass them via environment variables).

### ApiStack Update

- Add Lambda environment variables: `SCHEDULER_GROUP_NAME`, `SCHEDULER_ROLE_ARN`, `NOTIFICATION_LAMBDA_ARN`
- Grant `scheduler:CreateSchedule`, `scheduler:DeleteSchedule`, `scheduler:GetSchedule` to the API Lambda execution role
- Grant `iam:PassRole` on the scheduler execution role ARN (required for creating schedules)

---

## DynamoDB Key Pattern (if tracking schedules in DynamoDB)

| Entity | PK | SK |
|--------|----|----|
| Schedule | `USER#{userId}` | `SCHEDULE#{scheduleId}` |

---

## File Additions

```
Server/src/Server/
  Services/
    ISchedulerService.cs
    SchedulerService.cs

Infrastructure/
  Stacks/
    NotificationStack.cs
  Functions/
    NotificationFunction/
      Function.cs
      NotificationFunction.csproj
```
