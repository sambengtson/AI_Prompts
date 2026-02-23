# Module: Push Notifications

Adds push notification support via OneSignal (default) or FCM.

---

## Server Additions

### NotificationService

HttpClient-based service for sending push notifications via the provider's REST API.

Core operations:
- Send notification to a specific user (by external user ID or player ID)
- Send notification to a segment/topic
- Schedule a notification for future delivery

### UsersController Update

Add endpoint for subscription sync: when a user logs in on mobile, the app sends the push notification player/device ID to the server. The server stores this mapping to target notifications to specific users.

### Settings

`PushNotificationSettings` class with `FromEnvironmentAsync()` factory. Reads provider API key from Secrets Manager.

### NuGet Package

If using OneSignal REST API, no additional NuGet package needed (uses `HttpClient`). If using Firebase Admin SDK:
```xml
<PackageReference Include="FirebaseAdmin" />
```

---

## Infrastructure Additions

### SecretsStack

Add push notification provider secret:
```
{projectname}/push-notifications/{stage}
```

### ApiStack

- Pass secret ARN as Lambda environment variable: `PUSH_NOTIFICATION_SECRET_ARN`
- Grant Secrets Manager read permission

---

## Mobile Additions

### Platform-Specific Registration

**Android** (`Platforms/Android/`):
- Initialize push notification SDK in `MainApplication.cs`
- Handle notification permissions (Android 13+)

**iOS** (`Platforms/iOS/`):
- Request notification authorization in `AppDelegate.cs`
- Register for remote notifications
- Handle device token registration

### Subscription Sync

On login, retrieve the push notification player/device ID from the SDK and send it to the server via `POST /api/users/push-subscription` (or similar endpoint).

### NuGet Package

```xml
<!-- OneSignal -->
<PackageReference Include="Com.OneSignal" />

<!-- OR Firebase Cloud Messaging -->
<PackageReference Include="Plugin.Firebase.CloudMessaging" />
```

### Notification Preferences

Store user preferences for notification types in `Preferences`:
- `notifications_enabled` (bool)
- `notification_types` (comma-separated string of enabled types, if applicable)
