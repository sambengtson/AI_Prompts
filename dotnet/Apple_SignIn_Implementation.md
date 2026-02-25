# Apple Sign-In Implementation Guide for .NET MAUI + ASP.NET Core

> Reusable AI agent prompt for adding Apple Sign-In to a .NET MAUI mobile app with an ASP.NET Core backend and optional Blazor WebAssembly frontend.

---

## Prerequisites

Before starting code changes, the following must be configured in the Apple Developer portal and your secrets store.

### Apple Developer Portal (developer.apple.com)

**1. Enable Sign in with Apple on the App ID**
- Navigate to Certificates, Identifiers & Profiles > Identifiers
- Select the App ID matching your bundle identifier
- Under Capabilities, enable "Sign in with Apple"
- Save

**2. Create a Services ID (required for web sign-in)**
- Navigate to Identifiers > click "+" > select "Services IDs"
- Set a description and an identifier (convention: `{bundle_id}.web`)
- Register, then click into it and enable "Sign in with Apple"
- Click Configure:
  - Primary App ID: select your main app
  - Domains: add your web domain
  - Return URLs: add your web app's origin URL
- Save

**3. Create a Key for Sign in with Apple**
- Navigate to Keys > click "+"
- Name it, enable "Sign in with Apple", configure it to the primary App ID
- Register and download the `.p8` key file (one-time download)
- Note the Key ID (10-character string)
- Note your Team ID (found on the Membership page)

**4. Store credentials in your secrets manager**
Store a JSON secret with these fields:
```json
{
  "team_id": "<TEAM_ID>",
  "bundle_id": "<YOUR_BUNDLE_ID>",
  "services_id": "<YOUR_BUNDLE_ID>.web",
  "key_id": "<KEY_ID>",
  "private_key": "<CONTENTS_OF_.P8_FILE>"
}
```

**5. Regenerate provisioning profiles**
After enabling the capability, regenerate your provisioning profile. Xcode handles this automatically, or do it manually in the portal.

---

## Implementation Prompt

Give the following prompt to your AI agent. Replace placeholders in `{BRACES}` with your project's specifics.

---

### Prompt

```
Implement Apple Sign-In for this .NET MAUI + ASP.NET Core project. Follow the existing Google Sign-In implementation as the pattern for all new code. The app uses:

- .NET MAUI for mobile (iOS + Android targets)
- ASP.NET Core Web API for the server (with JWT auth)
- Blazor WebAssembly for the web frontend (optional, include if project has one)
- {YOUR_SECRETS_PROVIDER} for storing OAuth credentials (e.g., AWS Secrets Manager, Azure Key Vault, env vars)

Bundle ID: {YOUR_BUNDLE_ID}
Services ID: {YOUR_BUNDLE_ID}.web
Secrets path: {YOUR_SECRETS_PATH}

## What to implement

### Server-side

1. **Configuration class** (`AppleAuthSettings`):
   - Mirror the existing Google auth settings class
   - Load team_id, bundle_id, services_id, key_id, private_key from secrets
   - Add `IsConfigured` property checking if bundle_id or services_id is set
   - Register as singleton in DI

2. **Models** (`AppleLoginRequest`, `AppleUserInfo`):
   - `AppleLoginRequest` with a required `IdToken` string property
   - `AppleUserInfo` record with Sub, Email, Name?, Picture? (mirrors Google models)

3. **Service interface** (`IAppleAuthService`):
   - Single method: `Task<AppleUserInfo> ValidateIdTokenAsync(string idToken)`

4. **Service implementation** (`AppleAuthService`):
   - Validate Apple's JWT identity token by fetching Apple's JWKS public keys from `https://appleid.apple.com/auth/keys`
   - Cache the JWKS keys using IMemoryCache with 24-hour TTL
   - Use `JwtSecurityTokenHandler.ValidateToken()` with:
     - Valid issuer: `https://appleid.apple.com`
     - Valid audiences: both bundle_id AND services_id (iOS sends bundle_id, web sends services_id)
   - Extract `sub` and `email` claims from the validated token
   - If validation fails with cached keys, re-fetch JWKS once and retry (handles key rotation)
   - Register with `AddHttpClient<IAppleAuthService, AppleAuthService>()` in DI

5. **Auth service changes**:
   - Add `AppleLoginAsync(AppleLoginRequest request)` to the auth service interface and implementation
   - Mirror the Google login method exactly: validate token → lookup user by email → create new user OR auto-link email-only user OR verify Apple ID for existing Apple user OR reject cross-provider conflict
   - User model needs an `AppleId` field (parallel to `GoogleId`) and Provider value "Apple"

6. **Controller endpoint**:
   - Add `POST /api/auth/apple` endpoint (AllowAnonymous) mirroring the Google endpoint
   - Accept `AppleLoginRequest`, return same `LoginResponse` as other auth endpoints

### Mobile (iOS only)

7. **Service interface** (`IAppleAuthService`):
   - `Task<string?> SignInAsync()` — returns the identity token
   - `Task SignOutAsync()` — no-op (mirrors Google pattern)

8. **Service implementation** (`AppleAuthService`):
   - Use MAUI's built-in `AppleSignInAuthenticator.Default.AuthenticateAsync()` with options:
     - `IncludeEmailScope = true`
     - `IncludeFullNameScope = true`
   - Extract `id_token` from the result's Properties dictionary
   - Handle TaskCanceledException (user cancelled) by returning null
   - This is much simpler than Google's PKCE flow — no OAuth dance needed

9. **DI registration**:
   - Register `IAppleAuthService` only under `#if IOS` in MauiProgram.cs
   - This service does not exist on Android

10. **Mobile auth service**:
    - Add `AppleLoginAsync(string idToken)` method that posts to `api/auth/apple`
    - Mirror the existing `GoogleLoginAsync` method exactly

11. **Login ViewModel**:
    - Accept `IAppleAuthService?` as nullable constructor parameter with default null (Android DI won't have it)
    - Add `IsAppleLoading` observable property
    - Add `IsAppleAvailable` property: `DeviceInfo.Platform == DevicePlatform.iOS`
    - Add `AppleLoginCommand` relay command mirroring GoogleLoginCommand
    - Set `login_provider` preference to "Apple" on success

12. **Login page XAML**:
    - Add "Sign in with Apple" button below the Google button
    - Use `IsVisible="{Binding IsAppleAvailable}"` to hide on Android
    - Black background, white text/icon (Apple's branding guidelines)
    - Add Apple logo SVG asset

13. **Styles**:
    - Add `AppleSignInButton` style: black background, black stroke, rounded rectangle
    - Add `AppleSignInLabel` style: white text, semibold font

14. **Entitlements** (both Debug and Release plist files):
    - Add `com.apple.developer.applesignin` key with array value containing "Default"

### Web (Blazor WebAssembly) — if applicable

15. **Auth service**:
    - Add `AppleLoginAsync(string idToken)` to interface and implementation
    - Posts to `api/auth/apple`, mirrors GoogleLoginAsync

16. **JavaScript interop** (`appleSignIn.js`):
    - Initialize Apple JS SDK with `AppleID.auth.init()` using the services_id
    - Configure `usePopup: true` and scope `name email`
    - On successful sign-in, invoke the Blazor `HandleAppleCredential` method with the id_token
    - Handle popup_closed_by_user silently

17. **Login page**:
    - Add Apple sign-in button styled per Apple guidelines (black, with Apple logo SVG inline)
    - Add `HandleAppleSignIn` method that triggers JS interop
    - Add `HandleAppleCredential` JSInvokable method mirroring HandleGoogleCredential
    - Initialize Apple SDK in `OnAfterRenderAsync` after Google init

18. **HTML and config**:
    - Add Apple JS SDK script to index.html: `https://appleid.cdn-apple.com/appleauth/static/jsapi/appleid/1/en_US/appleid.auth.js`
    - Add `appleSignIn.js` script reference
    - Add `AppleServicesId` to `appsettings.json`

19. **CSS**:
    - Add `.apple-signin-container` and `.apple-signin-button` styles mirroring Google's

### Infrastructure — if applicable

20. **Secrets**:
    - Add Apple OAuth secret resource (mirrors Google OAuth secret)

21. **API/Lambda config**:
    - Grant read access to the Apple secret
    - Add `APPLE_OAUTH_SECRET_ARN` (or equivalent) environment variable

## Key technical details

- Apple's JWKS endpoint: `https://appleid.apple.com/auth/keys`
- Apple JWT issuer: `https://appleid.apple.com`
- Apple JWT audience: bundle_id (from iOS native) or services_id (from web)
- The `System.IdentityModel.Tokens.Jwt` NuGet package provides `JwtSecurityTokenHandler` and `JsonWebKeySet` for JWKS-based validation — no additional packages needed if already using JWT auth
- Apple only sends the user's name/email on the FIRST authorization. Subsequent sign-ins only return the user ID (sub claim). If you need to capture the name, extract it from the mobile client's initial result and send it alongside the token.
- Users may choose "Hide My Email" which provides a @privaterelay.appleid.com address. Treat this as a normal email.
- The NuGet package `Microsoft.IdentityModel.Protocols.OpenIdConnect` (transitive dependency of the JWT bearer package) provides `OpenIdConnectConfigurationRetriever` which can auto-fetch and cache JWKS, but manual fetch with IMemoryCache is simpler and has fewer dependencies.

## Do NOT

- Add Apple Sign-In on Android (Apple does not support it natively; web-based workarounds are fragile)
- Add unnecessary code comments
- Over-engineer the solution — follow existing patterns exactly
- Add the private_key/key_id to client-side code (these are server-only, used for generating client_secret for web token exchange if needed)
```

---

## Apple Sign-In JWKS Validation Reference

For the server-side `AppleAuthService`, here is the core validation pattern:

```csharp
public class AppleAuthService : IAppleAuthService
{
    private readonly HttpClient _httpClient;
    private readonly AppleAuthSettings _settings;
    private readonly ILogger<AppleAuthService> _logger;
    private readonly IMemoryCache _cache;

    private const string AppleJwksUrl = "https://appleid.apple.com/auth/keys";
    private const string AppleIssuer = "https://appleid.apple.com";
    private const string JwksCacheKey = "AppleJWKS";

    // Constructor with HttpClient, AppleAuthSettings, ILogger, IMemoryCache

    public async Task<AppleUserInfo> ValidateIdTokenAsync(string idToken)
    {
        // 1. Check IsConfigured, throw if not
        // 2. Get or fetch JWKS keys (cached 24h)
        // 3. Build TokenValidationParameters:
        //    - IssuerSigningKeys = jwks keys
        //    - ValidIssuer = "https://appleid.apple.com"
        //    - ValidAudiences = [settings.BundleId, settings.ServicesId]
        //    - ValidateLifetime = true
        // 4. Validate with JwtSecurityTokenHandler
        // 5. If validation fails with SecurityTokenSignatureKeyNotFoundException,
        //    invalidate cache, re-fetch JWKS, retry once
        // 6. Extract "sub" and "email" claims
        // 7. Return AppleUserInfo(sub, email, null, null)
    }
}
```

## Apple Logo SVG (White, for dark button background)

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
  <path fill="#FFFFFF" d="M17.05 20.28c-.98.95-2.05.88-3.08.4-1.09-.5-2.08-.48-3.24 0-1.44.62-2.2.44-3.06-.4C2.79 15.25 3.51 7.59 9.05 7.31c1.35.07 2.29.74 3.08.8 1.18-.24 2.31-.93 3.57-.84 1.51.12 2.65.72 3.4 1.8-3.12 1.87-2.38 5.98.48 7.13-.57 1.5-1.31 2.99-2.54 4.09zM12.03 7.25c-.15-2.23 1.66-4.07 3.74-4.25.29 2.58-2.34 4.5-3.74 4.25z"/>
</svg>
```
