# Module: Mobile Background Processing

Adds platform-specific background task execution for the MAUI mobile app.

---

## Pattern Overview

Mobile background processing lets the app perform work when it is not in the foreground. The actual work performed depends entirely on the user's domain — this module describes the platform patterns and key decisions.

---

## Android — WorkManager

### Architecture

- **`AndroidBackgroundService.cs`** — Implements `IBackgroundService`, manages WorkManager registration
- **`{Domain}Worker.cs`** — Extends `Worker`, performs the actual background task
- **`WatchdogWorker.cs`** (optional) — Periodic worker that ensures the main worker stays scheduled
- **`ForegroundService.cs`** (optional) — Persistent notification + foreground service for long-running work

### Key Decisions

- **Periodic vs one-time work**: Use `PeriodicWorkRequestBuilder` for recurring tasks (minimum 15-minute interval), `OneTimeWorkRequestBuilder` for on-demand tasks
- **Constraints**: Set network, battery, charging, or storage constraints on work requests
- **Foreground service**: Required if work exceeds ~10 minutes or needs real-time execution. Displays a persistent notification
- **Battery optimization**: Prompt the user to disable battery optimization (`ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`) for reliable background execution
- **ExistingPeriodicWorkPolicy**: Use `KEEP` to avoid re-registering if already scheduled, or `REPLACE` to update parameters

### Registration in MauiProgram.cs

```csharp
#if ANDROID
builder.Services.AddSingleton<IBackgroundService, AndroidBackgroundService>();
#endif
```

---

## iOS — BGTaskScheduler

### Architecture

- **`iOSBackgroundService.cs`** — Implements `IBackgroundService`, registers and schedules `BGTask` types
- **Custom delegate/handler** — Performs the actual work when the system grants execution time

### Key Decisions

- **BGAppRefreshTask** — Short tasks (~30s), system determines frequency based on app usage patterns
- **BGProcessingTask** — Longer tasks (minutes), can request network and external power requirements
- **CoreLocation significant change monitoring** — Battery-efficient alternative for location-based apps; triggers on cell tower changes rather than on a timer
- **Info.plist registration**: All background task identifiers must be registered in `BGTaskSchedulerPermittedIdentifiers`

### Registration in MauiProgram.cs

```csharp
#if IOS
builder.Services.AddSingleton<IBackgroundService, iOSBackgroundService>();
#endif
```

---

## Shared Interface

```csharp
public interface IBackgroundService
{
    Task StartAsync();
    Task StopAsync();
    bool IsRunning { get; }
}
```

Both platform implementations conform to this interface. The ViewModel or App lifecycle code calls `StartAsync()` after login and `StopAsync()` on logout.

---

## File Additions

```
Mobile/
  Platforms/
    Android/
      Services/
        AndroidBackgroundService.cs
        {Domain}Worker.cs
        ForegroundService.cs        (if long-running work)
        WatchdogWorker.cs           (if reliability is critical)
    iOS/
      Services/
        iOSBackgroundService.cs
```
