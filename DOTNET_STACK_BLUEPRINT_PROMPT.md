# Stack Blueprint Prompt

You are an expert .NET full-stack architect. Your task is to scaffold a complete application using the architecture described below. Replace all placeholders: `{ProjectName}` (PascalCase), `{projectname}` (lowercase), `{DomainEntity}` (primary domain object), and `{DotNetVersion}` (target framework, default: `net10.0`).

---

## 0. Scaffolding Instructions

Follow this process strictly:

1. **Ask all clarifying questions** (Section 1) before writing any code
2. Based on answers, identify which **module files** (`dotnet/*.md`) to reference
3. Replace all placeholders throughout: `{ProjectName}`, `{projectname}`, `{DomainEntity}`, `{DotNetVersion}`
4. Design domain-specific entities, access patterns, and API endpoints from the user's answers
5. Scaffold in order: **Infrastructure -> Server -> Web/Mobile -> CI/CD -> CLAUDE.md**
6. Generate `CLAUDE.md` last, summarizing the project's actual architecture

---

## 1. Clarifying Questions

Ask these before scaffolding. Adapt the architecture based on answers.

### Core

1. **Project name** (PascalCase) and a one-sentence description of what the app does
2. **Primary domain entities** and their relationships (e.g., "Products have many Orders, Orders have many LineItems")
3. **Key user-facing features / API operations** — what does the app actually do?

### Database

4. **DynamoDB** (single-table, serverless, PAY_PER_REQUEST) or **PostgreSQL** (Aurora Serverless v2, relational, VPC-isolated)? DynamoDB is the default for serverless-first workloads. Choose Postgres if the domain has complex relational queries, joins, or strict transactional requirements.

### Clients

5. Which client projects: **Mobile** (MAUI), **Web** (Blazor WASM), or **both**?

### Shared Models

6. Should request/response DTOs shared between clients and Server live in a **shared class library** (`Shared/Shared.csproj`), or be **duplicated per project** (default)? Duplication is simpler; a shared library reduces drift but adds build complexity.

### AWS Services

7. **File/image uploads?** (-> S3 — see `dotnet/S3_STORAGE_MODULE.md`)
8. **Transactional emails?** (-> SES — see `dotnet/EMAIL_SES_MODULE.md`)
9. **Other AWS services?** (Location Service, EventBridge Scheduler, etc. — see relevant modules)

### Third-Party Integrations

10. **Billing/payments?** Provider? (Stripe default — see `dotnet/STRIPE_MODULE.md`)
11. **Push notifications?** Provider? (OneSignal/FCM — see `dotnet/PUSH_NOTIFICATIONS_MODULE.md`)

### Features

12. **Mobile background processing?** Describe the work. (see `dotnet/BACKGROUND_PROCESSING_MODULE.md`)
13. **Shareable public links?** (see `dotnet/SHARE_LINKS_MODULE.md`)
14. **Report/PDF generation?** (see `dotnet/REPORT_GENERATION_MODULE.md`)
15. **Scheduled/deferred tasks?** (see `dotnet/SCHEDULED_TASKS_MODULE.md`)

### Branding

16. **Primary brand color?** (hex code or name — used in MAUI Colors.xaml and Web CSS variables)

---

## 2. Solution Structure

Create a single `.sln` file at the repo root:

```
{ProjectName}.sln
Mobile/Mobile.csproj                          (.NET MAUI - Android, iOS, macOS, Windows)
Web/Web.csproj                                (Blazor WebAssembly standalone)
Server/src/Server/Server.csproj               (ASP.NET Core Web API + AWS Lambda)
Server/test/Server.Tests/Server.Tests.csproj  (xUnit test project)
Infrastructure/Infrastructure.csproj          (AWS CDK in C#)
```

All projects target **{DotNetVersion}**. By default, no shared class library — models are duplicated per project to keep deployments independent. If the user opted for a shared library, add `Shared/Shared.csproj` for API-boundary DTOs.

Include only the client projects the user selected (Mobile, Web, or both).

---

## 3. Server Project (ASP.NET Core + AWS Lambda)

### 3.1 Dual Entry Points

**LambdaEntryPoint.cs** — Inherits `Amazon.Lambda.AspNetCoreServer.APIGatewayProxyFunction`:
```csharp
public class LambdaEntryPoint : APIGatewayProxyFunction
{
    protected override void Init(IWebHostBuilder builder)
    {
        builder.UseStartup<Startup>();
    }

    protected override void PostMarshallRequestFeature(
        IHttpRequestFeature aspNetCoreRequestFeature,
        APIGatewayProxyRequest apiGatewayRequest,
        ILambdaContext lambdaContext)
    {
        if (!string.IsNullOrEmpty(apiGatewayRequest.Body))
        {
            aspNetCoreRequestFeature.Headers ??= new HeaderDictionary();
            aspNetCoreRequestFeature.Headers["X-Raw-Body"] = apiGatewayRequest.Body;
        }
        base.PostMarshallRequestFeature(aspNetCoreRequestFeature, apiGatewayRequest, lambdaContext);
    }
}
```
Handler string: `Server::Server.LambdaEntryPoint::FunctionHandlerAsync`

The `X-Raw-Body` header preserves the raw request body before API Gateway re-encoding. This is needed for webhook signature verification (Stripe, etc.) where the exact body bytes must match.

**LocalEntryPoint.cs** — Standard Kestrel for local development:
```csharp
public class LocalEntryPoint
{
    public static void Main(string[] args)
    {
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>())
            .Build().Run();
    }
}
```

### 3.2 Startup.cs Pattern

**ConfigureServices** order:
1. Load settings from environment/Secrets Manager (see 3.5)
2. Register settings as singletons
3. Register AWS SDK clients as singletons (`IAmazonDynamoDB`, plus others based on modules)
4. Add `MemoryCache`
5. Register domain services as scoped (interface + implementation pairs)
6. Register HttpClient-based services with `AddHttpClient<TInterface, TImpl>()`
7. Configure JWT Bearer authentication with `SymmetricSecurityKey`, zero `ClockSkew`
8. Set `FallbackPolicy` to `RequireAuthenticatedUser()`
9. `AddControllers()`
10. Configure CORS (AllowAnyOrigin, AllowAnyMethod, AllowAnyHeader for API Gateway proxy)

**Configure** middleware order:
1. `UseDeveloperExceptionPage()` (dev only)
2. `UseRouting()`
3. `UseCors()`
4. `UseMiddleware<JwtAuthenticationMiddleware>()` (custom, before built-in auth)
5. `UseAuthentication()`
6. `UseAuthorization()`
7. `UseEndpoints()` with `MapControllers()` + root health endpoint

### 3.3 Custom JWT Middleware

`JwtAuthenticationMiddleware` sits before `UseAuthentication()`. It:
- Extracts Bearer token from Authorization header
- Validates token manually with `JwtSecurityTokenHandler`
- Sets `context.User = validatedPrincipal` on success
- Returns 401 JSON response on `SecurityTokenExpiredException` (short-circuits pipeline)
- Swallows other validation exceptions (lets request continue as anonymous)
- Validates algorithm is HmacSha256

### 3.4 Controller Pattern

```csharp
[ApiController]
[Route("api/{resource}")]
public class {Resource}Controller : ControllerBase
{
    private readonly I{Resource}Service _service;
    private readonly ILogger<{Resource}Controller> _logger;

    // Constructor injection
    // Actions use [HttpGet], [HttpPost], etc.
    // Get userId from claims: User.FindFirst(ClaimTypes.NameIdentifier)?.Value
    // AllowAnonymous on specific endpoints: [AllowAnonymous]
}
```

**Core controllers** (always included):
- **AuthController** (`/api/auth`): `POST /login`, `POST /refresh`
- **UsersController** (`/api/users`): `POST /signup` `[AllowAnonymous]`, profile CRUD
- **HealthCheckController** (`/health`): `GET /` `[AllowAnonymous]` — returns 200
- **PasswordResetController** (`/api/password-reset`): `POST /request` `[AllowAnonymous]`, `POST /confirm` `[AllowAnonymous]`
- **{DomainEntity}Controller** (`/api/{resource}`): CRUD + domain-specific operations

Additional controllers are added by modules (Stripe, ShareLinks, etc.).

### 3.5 Configuration from Environment + Secrets Manager

Each settings class has a static `FromEnvironment()` or `FromEnvironmentAsync()` factory:

```csharp
public class AppSettings
{
    public string TableName { get; set; } = null!;
    public string ApiGatewayUrl { get; set; } = null!;
    public string WebAppUrl { get; set; } = null!;
    public string Stage { get; set; } = null!;

    public static AppSettings FromEnvironment()
    {
        return new AppSettings
        {
            TableName = Environment.GetEnvironmentVariable("TABLE_NAME") ?? "default-value",
            // Add domain-specific settings as needed
        };
    }
}

public class JwtSettings
{
    public string SecretKey { get; set; } = null!;
    public string Issuer { get; set; } = null!;
    public string Audience { get; set; } = null!;
    public int ExpiryInMinutes { get; set; } = 60;
    public int RefreshTokenExpiryInDays { get; set; } = 30;

    public static async Task<JwtSettings> FromEnvironmentAsync()
    {
        // Reads JWT_SECRET_ARN env var -> fetches from Secrets Manager
        // Falls back to JWT_SECRET_KEY env var for local dev
    }
}
```

Additional settings classes follow the same pattern for module integrations (Stripe, push notifications, etc.).

### 3.6 DynamoDB Single-Table Design

One DynamoDB table with composite keys:
- **Partition Key**: `PK` (String)
- **Sort Key**: `SK` (String)
- **GSI1**: `GSI1PK` / `GSI1SK` (all attributes projected)
- **Billing**: PAY_PER_REQUEST
- **TTL**: Enabled on `TTL` attribute
- **PITR**: Enabled

Each model defines its own PK/SK pattern as computed properties:

```csharp
public class User
{
    public string Id { get; set; } = null!;
    public string Email { get; set; } = null!;
    public string PasswordHash { get; set; } = null!;
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    public string PK => $"USER#{Email.ToLowerInvariant()}";
    public string SK => $"USER#{Email.ToLowerInvariant()}";
}
```

**Core key patterns:**

| Entity | PK | SK | GSI1PK | GSI1SK |
|--------|----|----|--------|--------|
| User | `USER#{email}` | `USER#{email}` | - | - |
| RefreshToken | `REFRESH_TOKEN#{token}` | `USER#{email}` | - | - |
| PasswordReset | `RESET_TOKEN#{token}` | `USER#{email}` | - | - |
| {DomainEntity} | `USER#{userId}` | `{ENTITY}#{timestamp:O}` | *(design per access pattern)* | *(design per access pattern)* |

Design additional entity key patterns based on the user's domain entities and access patterns. Use GSIs sparingly — only for access patterns that can't be served by the main table's key structure.

**DynamoDbService** interface groups methods by entity: CRUD + query methods for each entity type. Implementation uses `AmazonDynamoDBClient` low-level API with `PutItemAsync`, `GetItemAsync`, `QueryAsync`, `DeleteItemAsync`.

### 3.7 Service Layer Pattern

Every service follows the interface + implementation pattern:
- `I{Name}Service.cs` in `Services/`
- `{Name}Service.cs` in `Services/`
- Constructor injection of `IDynamoDbService`, AWS SDK clients, settings classes, other services
- All methods are async (`Task<T>`)

**Core services** (always included):
- **Auth**: Login, signup, token refresh, password hashing (BCrypt), JWT generation
- **JWT**: Generate access tokens (ClaimTypes.NameIdentifier, ClaimTypes.Email), validate, extract claims
- **Password**: BCrypt hash + verify
- **DynamoDb** (or **AppDbContext** for Postgres): Central data access

Additional services are added by modules (S3, Email, Stripe, etc.). Follow the same interface + implementation pattern for all domain services.

### 3.8 Server NuGet Packages (Core)

```xml
<PackageReference Include="Amazon.Lambda.AspNetCoreServer" />
<PackageReference Include="Amazon.Lambda.Serialization.SystemTextJson" />
<PackageReference Include="AWSSDK.DynamoDBv2" />
<PackageReference Include="AWSSDK.SecretsManager" />
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" />
<PackageReference Include="System.IdentityModel.Tokens.Jwt" />
<PackageReference Include="BCrypt.Net-Next" />
```

Module-specific packages are listed in each module file.

The Server `.csproj` should include Lambda optimization settings:
```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <StripSymbols>true</StripSymbols>
  <PublishTrimmed>false</PublishTrimmed>
</PropertyGroup>
```

### 3.9 Test Project

- **Framework**: xUnit + Moq
- **Structure**: Mirror the Server project's service/controller layout
- **TestInjector pattern**: Create a shared `TestInjector` class (implements `IDisposable`) that:
  - Sets up a full DI container with test configuration (test database table, test secrets)
  - Creates test user accounts and tracks test data for cleanup
  - Exposes resolved services (`DynamoDbService`, `AuthService`, etc.) for integration tests
  - Implements `CleanupTestUserAsync()` to remove test data after each test run
- **Integration tests**: Use `IClassFixture<TestInjector>` to share setup across test classes
- **Unit tests**: Mock `IDynamoDbService` (or `AppDbContext` for Postgres) and AWS SDK clients for isolated service testing
- **Test categories**: `AuthControllerTests`, `{DomainEntity}ServiceTests`, `{DomainEntity}ControllerTests`

```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" />
<PackageReference Include="xunit" />
<PackageReference Include="xunit.runner.visualstudio" />
<PackageReference Include="Moq" />
```

---

## 4. Web Project (Blazor WebAssembly)

### 4.1 Program.cs

```csharp
var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

var apiBaseUrl = builder.Configuration["ApiBaseUrl"]
    ?? "https://{api-gateway-url}/dev/";

builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(apiBaseUrl) });

builder.Services.AddBlazoredLocalStorage();
builder.Services.AddAuthorizationCore();

builder.Services.AddScoped<AuthStateProvider>();
builder.Services.AddScoped<AuthenticationStateProvider>(sp =>
    sp.GetRequiredService<AuthStateProvider>());
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IApiService, ApiService>();

await builder.Build().RunAsync();
```

### 4.2 Authentication Pattern

**AuthStateProvider** extends `AuthenticationStateProvider`:
- Reads JWT + expiry + email from `Blazored.LocalStorage`
- Constructs `ClaimsPrincipal` with `ClaimsIdentity("jwt")` containing Name and Email claims
- Returns anonymous principal if token missing or expired
- Exposes `NotifyAuthenticationStateChanged()` for login/logout state transitions

**AuthService**:
- `LoginAsync(request)` -> POST to `api/auth/login` -> stores tokens in localStorage -> notifies auth state
- `LogoutAsync()` -> removes all tokens from localStorage -> notifies auth state
- `GetTokenAsync()` -> checks expiry (5-min buffer) -> auto-refreshes if needed -> returns token
- `RefreshTokenAsync()` -> POST to `api/auth/refresh` -> updates stored tokens -> logs out on failure
- `IsAuthenticatedAsync()` -> calls `GetTokenAsync()` and checks non-null

**localStorage keys**: `authToken`, `refreshToken`, `tokenExpiry`, `userEmail`

**ApiService**:
- Every authenticated method calls `SetAuthHeaderAsync()` before making HTTP requests
- `SetAuthHeaderAsync()` gets token via `IAuthService.GetTokenAsync()` and sets `Authorization: Bearer {token}`
- Methods return nullable response DTOs (`Task<T?>`)
- Uses `GetFromJsonAsync<T>` for GET, `PostAsJsonAsync` for POST, `PutAsJsonAsync` for PUT

### 4.3 App.razor (Root Component)

```razor
<CascadingAuthenticationState>
    <Router AppAssembly="@typeof(App).Assembly">
        <Found Context="routeData">
            <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)">
                <NotAuthorized>
                    <RedirectToLogin />
                </NotAuthorized>
                <Authorizing>
                    <p>Loading...</p>
                </Authorizing>
            </AuthorizeRouteView>
            <FocusOnNavigate RouteData="@routeData" Selector="h1" />
        </Found>
        <NotFound>
            <LayoutView Layout="@typeof(MainLayout)">
                <p>Page not found.</p>
            </LayoutView>
        </NotFound>
    </Router>
</CascadingAuthenticationState>
```

- `CascadingAuthenticationState` wraps the entire router so auth state is available to all components
- `AuthorizeRouteView` enforces `[Authorize]` attributes on pages
- `RedirectToLogin` is a small component that navigates to `/login` when the user is not authenticated

**_Imports.razor** — Global `@using` statements:
```razor
@using System.Net.Http
@using System.Net.Http.Json
@using Microsoft.AspNetCore.Components.Authorization
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.Web
@using Microsoft.AspNetCore.Authorization
@using Microsoft.JSInterop
@using Blazored.LocalStorage
@using Web
@using Web.Layout
@using Web.Services
@using Web.Models
```

### 4.4 Page Structure

**Core pages** (always included):
```
Web/
  Pages/
    Login.razor
    ForgotPassword.razor
    ResetPassword.razor
    Summary.razor          (home/dashboard - @page "/")
    {DomainPage}.razor     (add pages based on domain features)
  Layout/
    MainLayout.razor       (NavMenu, AuthorizeView wrapping)
  wwwroot/
    index.html
    appsettings.json       (ApiBaseUrl)
    css/app.css            (CSS variables, custom styles)
    fonts/
    lib/bootstrap/dist/    (Bootstrap 5)
```

Additional pages are added by modules (Pricing, Checkout, ShareLinks, etc.).

### 4.5 Styling

- **Bootstrap 5** for layout and components
- **CSS custom properties** for theme colors matching Mobile app
- **Custom font**: Open Sans (Regular + Semibold) via `@font-face`
- No inline styles — use CSS classes exclusively

### 4.6 Web NuGet Packages

```xml
<PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly" />
<PackageReference Include="Microsoft.AspNetCore.Components.Authorization" />
<PackageReference Include="Blazored.LocalStorage" />
```

---

## 5. Mobile Project (.NET MAUI)

### 5.1 MauiProgram.cs

```csharp
public static MauiApp CreateMauiApp()
{
    var builder = MauiApp.CreateBuilder();
    builder.UseMauiApp<App>()
        .UseMauiCommunityToolkit()
        .ConfigureFonts(fonts =>
        {
            fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
            fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
            fonts.AddFont("MaterialDesignIcons-Regular.ttf", "MaterialDesignIcons");
        });

    // HttpClientFactory
    builder.Services.AddHttpClient("ServerApi", client =>
    {
        client.BaseAddress = new Uri("{API_BASE_URL}");
        client.Timeout = TimeSpan.FromSeconds(30);
    });

    // Singletons: Token storage, Auth, API, domain services
    builder.Services.AddSingleton<ITokenStorageService, TokenStorageService>();
    builder.Services.AddSingleton<IAuthService, AuthService>();
    builder.Services.AddSingleton<IApiService, ApiService>();

    // Platform-specific services via conditional compilation
    #if ANDROID
    builder.Services.AddSingleton<IBackgroundService, AndroidBackgroundService>();
    #elif IOS
    builder.Services.AddSingleton<IBackgroundService, iOSBackgroundService>();
    #endif

    // ViewModels: Transient
    builder.Services.AddTransient<LoginViewModel>();
    builder.Services.AddTransient<{DomainEntity}SummaryViewModel>();

    // Pages: Transient
    builder.Services.AddTransient<LoginPage>();
    builder.Services.AddTransient<{DomainEntity}SummaryPage>();

    return builder.Build();
}
```

Remove the background service registration block if the user does not need mobile background processing.

### 5.2 MVVM with CommunityToolkit.Mvvm

All ViewModels are `partial class` inheriting `ObservableObject`:

```csharp
public partial class {DomainEntity}ViewModel : ObservableObject
{
    private readonly IApiService _apiService;

    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private string _errorMessage = string.Empty;

    [ObservableProperty]
    private ObservableCollection<{DomainEntity}> _items = new();

    public {DomainEntity}ViewModel(IApiService apiService)
    {
        _apiService = apiService;
    }

    [RelayCommand]
    private async Task LoadDataAsync()
    {
        IsLoading = true;
        ErrorMessage = string.Empty;
        try
        {
            var result = await _apiService.Get{DomainEntity}sAsync();
            Items = new ObservableCollection<{DomainEntity}>(result);
        }
        catch (Exception ex)
        {
            ErrorMessage = ex.Message;
        }
        finally
        {
            IsLoading = false;
        }
    }

    partial void OnSelectedItemChanged({DomainEntity}? value)
    {
        if (value != null) _ = LoadDetailAsync();
    }
}
```

Key CommunityToolkit patterns:
- `[ObservableProperty]` on `private` fields -> auto-generates PascalCase public properties + change notification
- `[RelayCommand]` on methods -> auto-generates `IAsyncRelayCommand` / `IRelayCommand` properties
- `[QueryProperty]` for Shell navigation parameters
- Partial `On{PropertyName}Changed()` methods for side-effects

### 5.3 Shell Navigation (Tab-Based)

```xml
<Shell FlyoutBehavior="Disabled">
    <TabBar>
        <ShellContent Title="Summary"  Route="summary"  Icon="chart.png"    ContentTemplate="{DataTemplate pages:SummaryPage}" />
        <ShellContent Title="Profile"  Route="profile"  Icon="user.png"     ContentTemplate="{DataTemplate pages:ProfilePage}" />
        <!-- Add 1-3 more tabs based on domain features -->
    </TabBar>
</Shell>
```

- Auth pages (Login, Signup, ForgotPassword, ResetPassword) are pushed modally via `Navigation.PushAsync()`
- Detail pages use `Shell.Current.GoToAsync("route?param=value")`
- Deep linking: `{projectname}://` URL scheme handled in platform-specific code

### 5.4 Token Storage (SecureStorage)

```csharp
public class TokenStorageService : ITokenStorageService
{
    private const string AccessTokenKey = "access_token";
    private const string RefreshTokenKey = "refresh_token";
    private const string AccessTokenExpiryKey = "access_token_expiry";
    private const string RefreshTokenExpiryKey = "refresh_token_expiry";

    public async Task SaveTokensAsync(string accessToken, string refreshToken,
        DateTime accessExpiry, DateTime refreshExpiry)
    {
        await SecureStorage.SetAsync(AccessTokenKey, accessToken);
        await SecureStorage.SetAsync(RefreshTokenKey, refreshToken);
        await SecureStorage.SetAsync(AccessTokenExpiryKey, accessExpiry.ToString("o"));
        await SecureStorage.SetAsync(RefreshTokenExpiryKey, refreshExpiry.ToString("o"));
    }

    public async Task<bool> IsTokenValidAsync()
    {
        var expiryStr = await SecureStorage.GetAsync(AccessTokenExpiryKey);
        if (DateTime.TryParse(expiryStr, out var expiry))
            return expiry > DateTime.UtcNow.AddMinutes(5);
        return false;
    }
}
```

### 5.5 API Service with Auto-Refresh

```csharp
public class ApiService : IApiService
{
    private readonly HttpClient _httpClient;
    private readonly ITokenStorageService _tokenStorage;

    public async Task<T?> GetAsync<T>(string endpoint)
    {
        await AddAuthHeaderAsync();
        var response = await _httpClient.GetAsync(endpoint);

        if (response.StatusCode == HttpStatusCode.Unauthorized)
        {
            if (await TryRefreshTokenAsync())
            {
                await AddAuthHeaderAsync();
                response = await _httpClient.GetAsync(endpoint);
            }
        }
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>();
    }

    // PostAsync<TReq, TRes>, PutAsync<TReq, TRes>, DeleteAsync follow same pattern
}
```

### 5.6 XAML Styling

**Colors.xaml** — Centralized color palette:
```xml
<Color x:Key="Primary">{PrimaryColor}</Color>
<Color x:Key="PrimaryDark">{PrimaryDarkColor}</Color>
<Color x:Key="Secondary">{SecondaryColor}</Color>
<Color x:Key="Tertiary">{TertiaryColor}</Color>
<!-- Full gray spectrum: Gray100 through Gray950 -->
```

**Styles.xaml** — Implicit + explicit styles:
- Implicit styles (no `x:Key`) applied to all controls of that type
- `AppThemeBinding` for light/dark mode
- `VisualStateManager` for Disabled/Normal/PointerOver states
- Font: OpenSans-Regular (default), OpenSans-Semibold (headings)

**App.xaml** — Merges Colors.xaml, Styles.xaml, registers converters globally.

### 5.7 Value Converters

```
Converters/
  InvertedBoolConverter.cs        (bool -> !bool)
  StringNotEmptyConverter.cs      (string -> bool)
  IsNotNullConverter.cs           (object -> bool)
  IsNullConverter.cs              (object -> bool)
  BoolToTextConverter.cs          (bool + "falseText|trueText" parameter -> string)
  IntToBoolConverter.cs           (int -> bool, non-zero = true)
```

### 5.8 Mobile NuGet Packages (Core)

```xml
<PackageReference Include="CommunityToolkit.Maui" />
<PackageReference Include="CommunityToolkit.Mvvm" />
```

Module-specific packages (charting, push notifications, etc.) are listed in each module file.

### 5.9 User Preferences (Non-Sensitive Storage)

Use `Preferences.Set/Get` for non-sensitive user settings:
- `remember_me` (bool)
- `user_email` (string)
- Add domain-specific preferences as needed

---

## 6. Infrastructure Project (AWS CDK in C#)

### 6.1 CDK App Entry Point

```csharp
var app = new App();

var stage = Environment.GetEnvironmentVariable("STAGE") ?? "dev";
var region = Environment.GetEnvironmentVariable("AWS_REGION")
    ?? Environment.GetEnvironmentVariable("CDK_DEFAULT_REGION") ?? "us-west-2";
var account = Environment.GetEnvironmentVariable("AWS_ACCOUNT_ID")
    ?? Environment.GetEnvironmentVariable("CDK_DEFAULT_ACCOUNT") ?? "";
var stackPrefix = Environment.GetEnvironmentVariable("STACK_PREFIX") ?? "{ProjectName}";

var env = new Amazon.CDK.Environment { Account = account, Region = region };
var commonTags = new Dictionary<string, string>
{
    { "Product", "{ProjectName}" },
    { "Environment", stage },
    { "ManagedBy", "CDK" }
};

// Independent stacks
var secretsStack = new SecretsStack(app, $"{stackPrefix}-Secrets-{stage}", new StackProps { Env = env, TerminationProtection = true });
var networkStack = new NetworkStack(app, $"{stackPrefix}-Network-{stage}", new StackProps { Env = env, TerminationProtection = true });
var databaseStack = new DatabaseStack(app, $"{stackPrefix}-Database-{stage}", new StackProps { Env = env, TerminationProtection = true });
var webStack = new WebStack(app, $"{stackPrefix}-Web-{stage}", new WebStackProps { Env = env, TerminationProtection = true });

// Dependent stacks
var apiStack = new ApiStack(app, $"{stackPrefix}-Api-{stage}", new ApiStackProps
{
    Env = env,
    TerminationProtection = true,
    Vpc = networkStack.Vpc,
    LambdaSecurityGroup = networkStack.LambdaSecurityGroup,
    Table = databaseStack.Table,
    JwtSecret = secretsStack.JwtSecret,
    // Add module-specific cross-stack refs (StorageBucket, etc.)
});
apiStack.AddDependency(secretsStack);
apiStack.AddDependency(networkStack);
apiStack.AddDependency(databaseStack);

// Apply tags
foreach (var stack in new Stack[] { secretsStack, networkStack, databaseStack, apiStack, webStack })
{
    foreach (var tag in commonTags)
        Tags.Of(stack).Add(tag.Key, tag.Value);
}

app.Synth();
```

Add StorageStack, NotificationStack, or other stacks based on modules selected.

### 6.2 Stack Props Pattern

```csharp
public class ApiStackProps : StackProps
{
    public IVpc Vpc { get; set; } = null!;
    public ISecurityGroup LambdaSecurityGroup { get; set; } = null!;
    public ITable Table { get; set; } = null!;
    public ISecret JwtSecret { get; set; } = null!;
    // Add module-specific props: IBucket StorageBucket, etc.
}
```

### 6.3 Stack Implementations

#### SecretsStack
- JWT secret: Auto-generated 64-character string via `Secret` construct
- Additional secrets for module integrations (Stripe keys, push notification keys, etc.)
- Expose as public properties for cross-stack references

#### NetworkStack
- VPC with 2 AZs, 1 NAT Gateway (cost-optimized)
- 3 subnet tiers: Public, Private (with egress), Isolated
- Lambda security group (egress-only)
- S3 VPC Gateway Endpoint for private/isolated subnets
- DNS hostnames and support enabled
- Expose: `Vpc`, `LambdaSecurityGroup`

#### DatabaseStack (DynamoDB — default)
- DynamoDB table: `{projectname}-database-{stage}-{account}-{region}`
- PK (String) + SK (String), PAY_PER_REQUEST billing
- GSI1: `GSI1PK` / `GSI1SK` (all attributes projected)
- TTL on `TTL` attribute
- Point-in-time recovery enabled
- Deletion protection enabled, `RemovalPolicy.RETAIN`
- Expose: `Table`

#### ApiStack
- **API Lambda**: 512MB memory, 90s timeout, {DotNetVersion} runtime, VPC placement in private subnets
  - Environment variables: `TABLE_NAME`, `JWT_SECRET_ARN`, `STAGE`, `API_GATEWAY_URL`, `WEB_APP_URL`, `ASPNETCORE_ENVIRONMENT`, `DEPLOYMENT_TIMESTAMP`, plus module-specific vars
  - IAM: DynamoDB read/write, Secrets Manager read, plus module-specific permissions
- **API Gateway**: REST API with Lambda proxy integration
  - Routes: `/` and `/{proxy+}` -> Lambda
  - Binary media types: `multipart/form-data`, `application/octet-stream`, `audio/*`, `video/*`, `image/*`
  - CORS: All origins, all methods, all headers
  - Throttling: 1000 burst, 500 rate
  - Logging: INFO level, data trace, metrics, X-Ray tracing
- Expose: `ApiUrl`, `ApiLambda`

#### WebStack
- S3 bucket for Blazor WASM static files
- CloudFront distribution with:
  - S3 origin via Origin Access Control (OAC)
  - SPA routing: 404/403 errors -> `/index.html` with 200 response, TTL 0
  - Default root object: `index.html`
  - Optional custom domain + ACM certificate
- `BucketDeployment` from `Web/bin/Release/{DotNetVersion}/publish/wwwroot`
- Expose: `WebAppUrl`, `WebBucket`, `WebDistribution`

### 6.4 Stack Dependencies

```
SecretsStack     (independent)
NetworkStack     (independent)
DatabaseStack    (independent with DynamoDB; depends on NetworkStack with Postgres)
WebStack         (independent)

ApiStack         -> depends on Secrets, Network, Database
                    (+ Storage, Notification, etc. based on modules)
```

When using Postgres, add `databaseStack.AddDependency(networkStack)` since the DatabaseStack needs the VPC and RdsSecurityGroup.

### 6.5 Infrastructure NuGet Packages

```xml
<PackageReference Include="Amazon.CDK.Lib" />
<PackageReference Include="Constructs" />
<PackageReference Include="Amazon.CDK.AWS.Lambda" />
```

---

## 7. Authentication Flow (End-to-End)

### 7.1 Signup

```
Mobile/Web                          Server
   |                                  |
   | POST /api/auth/signup           |
   | {Email, Password}              |
   |-------------------------------->|
   |                                  | BCrypt hash password
   |                                  | Create User in database
   |                                  | Generate JWT + RefreshToken
   |                                  | Store RefreshToken in database
   |<--------------------------------|
   | {Token, RefreshToken,           |
   |  Email, UserId, ExpiresAt,     |
   |  RefreshTokenExpiresAt}        |
   |                                  |
   | Store tokens locally            |
```

### 7.2 Login

Same as signup flow but validates BCrypt hash instead of creating user.

### 7.3 Token Refresh

```
Mobile/Web                          Server
   |                                  |
   | POST /api/auth/refresh          |
   | {RefreshToken}                  |
   |-------------------------------->|
   |                                  | Look up RefreshToken in database
   |                                  | Validate not expired/revoked
   |                                  | Look up User
   |                                  | Generate new JWT + new RefreshToken
   |                                  | Delete old, store new RefreshToken
   |<--------------------------------|
   | {Token, RefreshToken, ...}      |
```

### 7.4 Auto-Refresh Pattern

Both Mobile and Web clients check token expiry before each API call with a 5-minute buffer. If the token is about to expire, they automatically call the refresh endpoint before proceeding with the original request. On 401 response, they retry with a fresh token. On refresh failure, the user is logged out.

### 7.5 Token Storage by Platform

| Platform | Storage Mechanism | Keys |
|----------|-------------------|------|
| Mobile (MAUI) | `SecureStorage` (Keychain/KeyStore) | `access_token`, `refresh_token`, `access_token_expiry`, `refresh_token_expiry` |
| Web (Blazor) | `Blazored.LocalStorage` | `authToken`, `refreshToken`, `tokenExpiry`, `userEmail` |

---

## 8. CI/CD (GitHub Actions)

```yaml
name: Build

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build-server:
    name: Build Server
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '{DotNetMajorVersion}.0.x'
    - name: Restore
      run: dotnet restore Server/src/Server/Server.csproj
    - name: Build
      run: dotnet build Server/src/Server/Server.csproj --no-restore -c Release
    - name: Restore Tests
      run: dotnet restore Server/test/Server.Tests/Server.Tests.csproj
    - name: Build Tests
      run: dotnet build Server/test/Server.Tests/Server.Tests.csproj --no-restore -c Release
    - name: Run Tests
      run: dotnet test Server/test/Server.Tests/Server.Tests.csproj --no-build --verbosity normal -c Release

  build-web:
    name: Build Web
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '{DotNetMajorVersion}.0.x'
    - run: dotnet restore Web/Web.csproj
    - run: dotnet build Web/Web.csproj --no-restore -c Release
    - run: dotnet publish Web/Web.csproj -c Release --no-build -o ./publish/web

  build-infrastructure:
    name: Build Infrastructure
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '{DotNetMajorVersion}.0.x'
    - run: dotnet restore Infrastructure/Infrastructure.csproj
    - run: dotnet build Infrastructure/Infrastructure.csproj --no-restore -c Release
```

Three parallel jobs: Server (build + test), Web (build + publish), Infrastructure (build). No auto-deploy — CDK deploy is manual.

Replace `{DotNetMajorVersion}` with the major version number (e.g., `10` for `net10.0`).

---

## 9. Key Design Patterns

### 9.1 Model Duplication (No Shared Library) — Default

Models are duplicated across Mobile, Web, and Server projects. This keeps each project independently deployable without shared assembly dependencies. Server models include DynamoDB PK/SK properties; client models are pure DTOs.

> **Optional: Shared Class Library.** If the user opts for a shared library, create `Shared/Shared.csproj` ({DotNetVersion}, no platform-specific dependencies) containing only the request/response DTOs that cross the API boundary. Server domain models with database-specific properties must remain in the Server project. Reference the shared library from Mobile, Web, and Server. Be aware this adds a build dependency — MAUI multi-targeting and Blazor WASM IL trimming may require additional configuration.

### 9.2 Service Interface Pattern

Every service has an `I{Name}Service` interface and `{Name}Service` implementation. This enables:
- Unit testing with mocks
- Platform-specific implementations (Mobile background services)
- Clean DI registration

### 9.3 Configuration from Environment

Server settings are loaded from environment variables (set by CDK Lambda configuration). Secrets come from AWS Secrets Manager via ARN environment variables. Local dev uses fallback default values or direct env vars.

### 9.4 Lambda + Kestrel Dual Entry Point

The `Startup.cs` is shared between Lambda and local Kestrel execution. `LambdaEntryPoint` and `LocalEntryPoint` both call `UseStartup<Startup>()`. This allows identical behavior in production (Lambda) and development (Kestrel).

### 9.5 Cross-Stack References (CDK)

Stacks expose resources as public properties (compile-time safe). Dependent stacks receive these via custom `StackProps` classes. `AddDependency()` ensures correct deployment order.

### 9.6 DynamoDB Single-Table Access — Default

All entities share one table. Each entity defines its PK/SK pattern as computed C# properties. The `DynamoDbService` handles marshalling between C# objects and DynamoDB `AttributeValue` dictionaries. GSIs are used sparingly for access patterns that can't be served by the main table's key structure.

> **Alternative: PostgreSQL via Aurora Serverless v2.** If the user opts for Postgres, replace the DynamoDB table and `DynamoDbService` entirely. The infrastructure and server changes are detailed below.
>
> **Infrastructure changes — NetworkStack:**
> Add an `RdsSecurityGroup` alongside the existing `LambdaSecurityGroup`. The RDS security group must allow inbound from Lambda on port 5432:
> ```csharp
> RdsSecurityGroup = new SecurityGroup(this, "RdsSecurityGroup", new SecurityGroupProps
> {
>     Vpc = Vpc,
>     SecurityGroupName = $"{projectname}-rds-sg-{stage}",
>     Description = "Security group for RDS Aurora Serverless",
>     AllowAllOutbound = true
> });
>
> RdsSecurityGroup.AddIngressRule(
>     LambdaSecurityGroup,
>     Port.Tcp(5432),
>     "Allow Lambda to connect to PostgreSQL"
> );
>
> RdsSecurityGroup.AddIngressRule(
>     RdsSecurityGroup,
>     Port.AllTraffic(),
>     "Allow self for RDS Proxy"
> );
> ```
> Export `RdsSecurityGroup` as a public property for cross-stack use.
>
> **Infrastructure changes — DatabaseStack (replaces DynamoDB):**
> ```csharp
> public class DatabaseStack : Stack
> {
>     public IDatabaseCluster Database { get; }
>     public ISecret DatabaseSecret { get; }
>
>     public DatabaseStack(Construct scope, string id, DatabaseStackProps props) : base(scope, id, props)
>     {
>         var stage = props.Stage;
>
>         DatabaseSecret = new Secret(this, "DatabaseSecret", new SecretProps
>         {
>             SecretName = $"{projectname}/rds/credentials/{stage}",
>             Description = "RDS Aurora Serverless PostgreSQL credentials",
>             GenerateSecretString = new SecretStringGenerator
>             {
>                 SecretStringTemplate = "{\"username\": \"postgres\"}",
>                 GenerateStringKey = "password",
>                 ExcludeCharacters = " %+~`#$&*()|[]{}:;<>?!'/\"@=\\",
>                 PasswordLength = 32
>             }
>         });
>
>         var subnetGroup = new SubnetGroup(this, "DatabaseSubnetGroup", new SubnetGroupProps
>         {
>             SubnetGroupName = $"{projectname}-db-subnet-group-{stage}",
>             Description = "Subnet group for RDS",
>             Vpc = props.Vpc,
>             VpcSubnets = new SubnetSelection { SubnetType = SubnetType.PRIVATE_WITH_EGRESS }
>         });
>
>         Database = new DatabaseCluster(this, "Database", new DatabaseClusterProps
>         {
>             Engine = DatabaseClusterEngine.AuroraPostgres(new AuroraPostgresClusterEngineProps
>             {
>                 Version = AuroraPostgresEngineVersion.VER_15_12
>             }),
>             ClusterIdentifier = $"{projectname}-aurora-{stage}",
>             DefaultDatabaseName = "{projectname}",
>             Credentials = Credentials.FromSecret(DatabaseSecret),
>             Vpc = props.Vpc,
>             SubnetGroup = subnetGroup,
>             SecurityGroups = new[] { props.SecurityGroup },
>             Writer = ClusterInstance.ServerlessV2("writer", new ServerlessV2ClusterInstanceProps
>             {
>                 ScaleWithWriter = true
>             }),
>             ServerlessV2MinCapacity = stage == "prod" ? 1 : 0,
>             ServerlessV2MaxCapacity = 1,
>             EnableDataApi = true,
>             StorageEncrypted = true,
>             Backup = new BackupProps { Retention = Duration.Days(7) },
>             DeletionProtection = true,
>             RemovalPolicy = RemovalPolicy.RETAIN,
>             ParameterGroup = ParameterGroup.FromParameterGroupName(
>                 this, "ParameterGroup", "default.aurora-postgresql15"
>             )
>         });
>
>         Database.MetricCPUUtilization().CreateAlarm(this, "DatabaseCPUAlarm", new CreateAlarmOptions
>         {
>             Threshold = 80,
>             EvaluationPeriods = 2,
>             AlarmDescription = "Database CPU utilization is high",
>             TreatMissingData = TreatMissingData.NOT_BREACHING
>         });
>     }
> }
>
> public class DatabaseStackProps : StackProps
> {
>     public IVpc Vpc { get; set; }
>     public ISecurityGroup SecurityGroup { get; set; }
>     public string Stage { get; set; }
> }
> ```
> Key configuration: Aurora Serverless v2 scales to zero in dev, `StorageEncrypted = true`, credentials auto-generated in Secrets Manager, 7-day backup retention.
>
> **Infrastructure changes — ApiStack:**
> - Pass `DatabaseSecret` via props and grant read: `props.DatabaseSecret.GrantRead(LambdaExecutionRole)`
> - Add Lambda environment variables: `DATABASE_SECRET_ARN` and `DATABASE_SECRET_NAME`
> - Lambda remains in `PRIVATE_WITH_EGRESS` subnets using the `LambdaSecurityGroup`
>
> **Infrastructure changes — RDS Proxy (optional, for high-concurrency):**
> If Lambda concurrency causes connection exhaustion, add an `RdsProxyStack` with a `CfnDBProxy` (engine family `POSTGRESQL`, TLS required, 30-minute idle timeout). Place the proxy in private subnets using the `RdsSecurityGroup`.
>
> **Server changes:**
> - Replace `IDynamoDbService` / `DynamoDbService` with Entity Framework Core (`Npgsql.EntityFrameworkCore.PostgreSQL`)
> - Create an `AppDbContext : DbContext` with `DbSet<T>` properties for each entity
> - Register with `AddDbContext<AppDbContext>()` in `Startup.cs`, reading the connection string from Secrets Manager at startup
> - Remove PK/SK computed properties from models; use standard `Id` (Guid) primary keys, foreign keys, and navigation properties
> - Use EF Core migrations for schema management
> - Add `Microsoft.EntityFrameworkCore` and `Npgsql.EntityFrameworkCore.PostgreSQL` NuGet packages

### 9.7 Authentication Fallback Policy

The server uses `FallbackPolicy = RequireAuthenticatedUser()` so all endpoints require auth by default. Specific endpoints opt out with `[AllowAnonymous]` (health check, login, signup, password reset, webhooks, public data).

### 9.8 ASPNETCORE_ENVIRONMENT from Stage

CDK sets `ASPNETCORE_ENVIRONMENT` based on stage: `"dev"` -> `"Development"`, anything else -> `"Production"`. This controls developer exception pages and other environment-specific behavior.

---

## 10. File Tree Reference

```
{ProjectName}/
  {ProjectName}.sln
  CLAUDE.md
  .github/workflows/build.yml

  Server/
    src/Server/
      Server.csproj
      Startup.cs
      LambdaEntryPoint.cs
      LocalEntryPoint.cs
      Configuration/
        AppSettings.cs
        JwtSettings.cs
        {Integration}Settings.cs      (per module integration)
      Controllers/
        AuthController.cs
        HealthCheckController.cs
        UsersController.cs
        PasswordResetController.cs
        {DomainEntity}Controller.cs   (per domain resource)
      Services/
        I{Name}Service.cs + {Name}Service.cs  (pairs)
        DynamoDbService.cs             (or AppDbContext.cs for Postgres)
        JwtService.cs
        PasswordService.cs
        AuthService.cs
      Models/
        User.cs
        {DomainEntity}.cs
        RefreshToken.cs
        PasswordResetToken.cs
        Auth/                          (auth DTOs)
      Middleware/
        JwtAuthenticationMiddleware.cs
    test/Server.Tests/
      Server.Tests.csproj

  Web/
    Web.csproj
    Program.cs
    App.razor
    _Imports.razor
    Pages/
      Login.razor
      ForgotPassword.razor
      ResetPassword.razor
      Summary.razor
      {DomainPage}.razor
    Layout/
      MainLayout.razor
    Services/
      AuthService.cs / IAuthService.cs
      ApiService.cs / IApiService.cs
      AuthStateProvider.cs
    Models/
      LoginRequest.cs / LoginResponse.cs
      {DomainEntity}Response.cs
    wwwroot/
      index.html
      appsettings.json
      css/app.css
      fonts/
      lib/bootstrap/

  Mobile/
    Mobile.csproj
    MauiProgram.cs
    App.xaml / App.xaml.cs
    AppShell.xaml / AppShell.xaml.cs
    Pages/
      LoginPage.xaml / .cs
      {DomainEntity}SummaryPage.xaml / .cs
      ProfilePage.xaml / .cs
    ViewModels/
      LoginViewModel.cs
      {DomainEntity}SummaryViewModel.cs
      ProfileViewModel.cs
    Services/
      IAuthService.cs / AuthService.cs
      IApiService.cs / ApiService.cs
      ITokenStorageService.cs / TokenStorageService.cs
    Models/
      LoginRequest.cs / LoginResponse.cs
      {DomainEntity}Response.cs
    Converters/
      InvertedBoolConverter.cs
      BoolToTextConverter.cs
    Resources/
      Styles/Colors.xaml
      Styles/Styles.xaml
      Fonts/
      Images/
    Platforms/
      Android/
        MainActivity.cs
        MainApplication.cs
      iOS/
        AppDelegate.cs

  Infrastructure/
    Infrastructure.csproj
    Program.cs
    Stacks/
      SecretsStack.cs
      NetworkStack.cs
      DatabaseStack.cs
      ApiStack.cs
      WebStack.cs
```

Additional files from modules (StorageStack, StripeController, S3Service, background services, etc.) are described in each module file.

---

## 11. Build Commands Reference

```bash
# Solution
dotnet build {ProjectName}.sln

# Server (local dev)
dotnet run --project Server/src/Server/Server.csproj

# Server (tests)
dotnet test Server/test/Server.Tests/Server.Tests.csproj

# Web (local dev)
dotnet run --project Web/Web.csproj

# Web (publish)
dotnet publish Web/Web.csproj -c Release

# Mobile (Windows)
dotnet build Mobile/Mobile.csproj -f {DotNetVersion}-windows10.0.19041.0

# Mobile (Android)
dotnet build Mobile/Mobile.csproj -f {DotNetVersion}-android

# Mobile (iOS)
dotnet build Mobile/Mobile.csproj -f {DotNetVersion}-ios

# Infrastructure (synth)
dotnet run --project Infrastructure/Infrastructure.csproj cdk synth

# Infrastructure (deploy)
dotnet run --project Infrastructure/Infrastructure.csproj cdk deploy --all
```

---

## 12. Available Modules

Reference these files for optional features based on clarifying question answers:

| Module | File | When to Include |
|--------|------|-----------------|
| Billing/Payments | `dotnet/STRIPE_MODULE.md` | User needs Stripe or payment processing |
| Push Notifications | `dotnet/PUSH_NOTIFICATIONS_MODULE.md` | Mobile app needs push notifications |
| Background Processing | `dotnet/BACKGROUND_PROCESSING_MODULE.md` | Mobile app needs background work |
| File Storage | `dotnet/S3_STORAGE_MODULE.md` | App needs file/image uploads |
| Transactional Email | `dotnet/EMAIL_SES_MODULE.md` | App needs email sending (password reset, welcome, etc.) |
| Shareable Links | `dotnet/SHARE_LINKS_MODULE.md` | App needs public shareable links |
| Report Generation | `dotnet/REPORT_GENERATION_MODULE.md` | App needs PDF/report generation |
| Geolocation | `dotnet/GEOLOCATION_MODULE.md` | App needs reverse geocoding / AWS Location Service |
| Scheduled Tasks | `dotnet/SCHEDULED_TASKS_MODULE.md` | App needs deferred/recurring tasks via EventBridge |
