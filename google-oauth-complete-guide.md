# Complete End-to-End Google OAuth 2.0 Implementation Guide
## .NET 8 Web API with FastEndpoints

---

## 📋 Table of Contents
1. [Overview & Visual Flow](#overview--visual-flow)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Complete Implementation](#complete-implementation)
4. [Detailed Explanations](#detailed-explanations)
5. [Security Considerations](#security-considerations)
6. [Troubleshooting](#troubleshooting)

---

## 🎯 Overview & Visual Flow

### The Complete OAuth 2.0 Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GOOGLE OAUTH 2.0 FLOW                              │
└─────────────────────────────────────────────────────────────────────────────┘

STEP 1: INITIATE LOGIN
┌──────────┐                                                    ┌──────────┐
│          │  1. User clicks "Sign in with Google"             │          │
│  User's  │  ──────────────────────────────────────────────▶  │  Your    │
│  Browser │                                                    │  Web API │
│          │  2. Redirect to /api/auth/google/signin           │          │
└──────────┘  ◀──────────────────────────────────────────────  └──────────┘
     │                                                                │
     │        3. API responds with HTTP 302 Redirect                 │
     │           Location: https://accounts.google.com/o/oauth2...   │
     │        ◀──────────────────────────────────────────────────────┘
     │
     ▼
┌──────────┐
│  Google  │  4. User sees Google login page
│  Login   │     - Enters email/password
│  Page    │     - Consents to permissions (email, profile)
└────┬─────┘
     │
     │        5. Google validates credentials
     │           Generates authorization code
     │
     ▼
┌──────────┐
│  Google  │  6. Redirects back to your callback URL
│  OAuth   │     https://yourapi.com/api/auth/google/callback
│  Server  │       ?code=4/0AbCdEf123...xyz
└────┬─────┘       &state=security_token_xyz
     │             &scope=openid%20email%20profile
     │
     │
     ▼
┌──────────┐                                                    ┌──────────┐
│  User's  │  7. Browser automatically redirects to callback   │          │
│  Browser │  ──────────────────────────────────────────────▶  │  Your    │
│          │     with authorization code in URL                │  Web API │
└──────────┘                                                    └────┬─────┘
                                                                     │
                  STEP 2: TOKEN EXCHANGE (Backend)                  │
                  ────────────────────────────────────              │
                                                                     │
     ┌───────────────────────────────────────────────────────────┐ │
     │ 8. AuthenticateAsync() is called                          │ │
     │    - Extracts 'code' from URL query string                │ │
     │    - Validates 'state' parameter (CSRF protection)        │ │
     │    - Prepares token exchange request                      │ │
     └───────────────────────────────────────────────────────────┘ │
                                                                     │
┌──────────┐                                                    ┌────▼─────┐
│          │  9. Backend POST to Google Token Endpoint          │          │
│  Google  │  ◀──────────────────────────────────────────────── │  Your    │
│  OAuth   │     POST https://oauth2.googleapis.com/token       │  Web API │
│  Server  │     Body: {                                         │          │
│          │       code: "4/0AbCdEf123...xyz",                   │          │
│          │       client_id: "your-app-id",                     │          │
│          │       client_secret: "your-secret",    ◀────────────┤ (SECRET) │
│          │       redirect_uri: "your-callback-url",            │          │
│          │       grant_type: "authorization_code"              │          │
│          │     }                                               │          │
└────┬─────┘                                                    └──────────┘
     │
     │        10. Google validates request and responds
     │            with tokens and user information
     │
     ▼
┌──────────┐                                                    ┌──────────┐
│  Google  │  11. Response with tokens                          │          │
│  OAuth   │  ──────────────────────────────────────────────▶   │  Your    │
│  Server  │      {                                             │  Web API │
│          │        "access_token": "ya29.a0AfH6...",           │          │
│          │        "expires_in": 3599,                         │          │
│          │        "token_type": "Bearer",                     │          │
│          │        "scope": "openid email profile",            │          │
│          │        "id_token": "eyJhbGciOiJS..."  ◀────────────┤  JWT!    │
│          │      }                                             │          │
└──────────┘                                                    └────┬─────┘
                                                                     │
                  STEP 3: PROCESS USER DATA                         │
                  ────────────────────────────                      │
                                                                     │
     ┌───────────────────────────────────────────────────────────┐ │
     │ 12. Your API processes the response:                      │ │
     │     - Decodes id_token (JWT)                              │ │
     │     - Extracts user claims (email, name, picture, etc.)   │ │
     │     - Validates JWT signature                             │ │
     │     - Creates ClaimsPrincipal object                      │ │
     └───────────────────────────────────────────────────────────┘ │
                                                                     │
     ┌───────────────────────────────────────────────────────────┐ │
     │ 13. Your business logic:                                  │ │
     │     - Check if user exists in your database               │ │
     │     - Create new user OR update existing user             │ │
     │     - Generate YOUR OWN JWT token for API access          │ │
     │     - Set session/authentication state                    │ │
     └───────────────────────────────────────────────────────────┘ │
                                                                     │
┌──────────┐                                                    ┌────▼─────┐
│  User's  │  14. Return YOUR JWT token to frontend             │          │
│  Browser │  ◀──────────────────────────────────────────────── │  Your    │
│          │      {                                             │  Web API │
│          │        "accessToken": "your-jwt-token",            │          │
│          │        "email": "user@gmail.com",                  │          │
│          │        "name": "John Doe",                         │          │
│          │        "picture": "https://..."                    │          │
│          │      }                                             │          │
└────┬─────┘                                                    └──────────┘
     │
     ▼
┌──────────┐
│ Frontend │  15. Store token in localStorage/sessionStorage
│   App    │      Use token for subsequent API calls
│          │      Authorization: Bearer your-jwt-token
└──────────┘

STEP 4: AUTHENTICATED API REQUESTS
┌──────────┐                                                    ┌──────────┐
│ Frontend │  16. Make API calls with JWT token                 │          │
│   App    │  ──────────────────────────────────────────────▶   │  Your    │
│          │      GET /api/user/profile                         │  Web API │
│          │      Authorization: Bearer your-jwt-token          │          │
│          │                                                     │          │
│          │  17. API validates JWT and returns data            │          │
│          │  ◀──────────────────────────────────────────────── │          │
└──────────┘                                                    └──────────┘
```

---

## 🔧 Prerequisites & Setup

### 1. Google Cloud Console Setup

#### Step-by-Step Google OAuth Configuration:

```
1. Go to: https://console.cloud.google.com
2. Create a new project or select existing
3. Navigate to: APIs & Services → Credentials
4. Click: Create Credentials → OAuth 2.0 Client ID
5. Configure OAuth Consent Screen (if first time):
   - User Type: External (for public apps)
   - App Name: Your App Name
   - User Support Email: your-email@domain.com
   - Scopes: Add email, profile, openid
   - Test Users: Add your Gmail for testing
6. Create OAuth Client ID:
   - Application Type: Web Application
   - Name: Your App Name
   - Authorized Redirect URIs:
     * Development: http://localhost:5000/api/auth/google/callback
     * Production: https://yourdomain.com/api/auth/google/callback
7. Copy Client ID and Client Secret
```

### 2. Install NuGet Packages

```bash
# Core FastEndpoints
dotnet add package FastEndpoints
dotnet add package FastEndpoints.Security

# Google Authentication
dotnet add package Microsoft.AspNetCore.Authentication.Google

# JWT Token Generation (for your own tokens)
dotnet add package System.IdentityModel.Tokens.Jwt
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

# Optional: For database operations
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

---

## 💻 Complete Implementation

### 📁 Project Structure

```
YourProject/
├── Program.cs                          # Application entry point
├── appsettings.json                    # Configuration
├── appsettings.Development.json        # Dev configuration
├── Features/
│   ├── Auth/
│   │   ├── GoogleSignInEndpoint.cs     # Initiates Google OAuth
│   │   ├── GoogleCallbackEndpoint.cs   # Handles OAuth callback
│   │   ├── SignOutEndpoint.cs          # Logout endpoint
│   │   └── Models/
│   │       └── AuthModels.cs           # DTOs and models
│   ├── User/
│   │   ├── GetCurrentUserEndpoint.cs   # Protected endpoint example
│   │   └── Models/
│   │       └── UserModels.cs
│   └── Services/
│       ├── ITokenService.cs            # JWT generation interface
│       ├── TokenService.cs             # JWT generation implementation
│       ├── IUserService.cs             # User management interface
│       └── UserService.cs              # User CRUD operations
└── Data/
    ├── ApplicationDbContext.cs         # EF Core context
    └── Entities/
        └── User.cs                     # User entity
```

---

### 📄 1. Program.cs - Application Configuration

```csharp
using FastEndpoints;
using FastEndpoints.Security;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.Google;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using YourApp.Data;
using YourApp.Services;

var builder = WebApplication.CreateBuilder(args);

// ═══════════════════════════════════════════════════════════════════════
// SECTION 1: DATABASE CONFIGURATION
// ═══════════════════════════════════════════════════════════════════════
// Configure Entity Framework Core with SQL Server
// This is where user data will be stored after Google authentication
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        // Add retry logic for transient failures
        sqlServerOptions => sqlServerOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null
        )
    )
);

// ═══════════════════════════════════════════════════════════════════════
// SECTION 2: DEPENDENCY INJECTION - Services
// ═══════════════════════════════════════════════════════════════════════
// Register application services
builder.Services.AddScoped<IUserService, UserService>();      // User CRUD operations
builder.Services.AddScoped<ITokenService, TokenService>();    // JWT generation

// ═══════════════════════════════════════════════════════════════════════
// SECTION 3: AUTHENTICATION CONFIGURATION
// ═══════════════════════════════════════════════════════════════════════

/*
 * AUTHENTICATION SCHEMES EXPLAINED:
 * 
 * In this app, we use TWO authentication schemes:
 * 
 * 1. COOKIE AUTHENTICATION (for OAuth flow)
 *    - Used temporarily during Google OAuth process
 *    - Stores authentication state between redirect and callback
 *    - Not used for API requests
 * 
 * 2. JWT BEARER AUTHENTICATION (for API requests)
 *    - Used for all subsequent API calls after login
 *    - Token sent in Authorization header
 *    - Stateless and scalable
 * 
 * 3. GOOGLE AUTHENTICATION (OAuth provider)
 *    - Handles communication with Google OAuth servers
 *    - Only used during login flow
 */

builder.Services.AddAuthentication(options =>
{
    // Default scheme for reading authentication cookies during OAuth flow
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    
    // When user needs to login, challenge with Google OAuth
    options.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
})
// ─────────────────────────────────────────────────────────────────────
// Cookie Authentication Configuration
// ─────────────────────────────────────────────────────────────────────
.AddCookie(options =>
{
    // Where to redirect if user is not authenticated
    options.LoginPath = "/api/auth/google/signin";
    
    // Where to redirect for logout
    options.LogoutPath = "/api/auth/signout";
    
    // Cookie settings for security
    options.Cookie.Name = "YourApp.Auth";
    options.Cookie.HttpOnly = true;        // Prevent JavaScript access
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always; // HTTPS only
    options.Cookie.SameSite = SameSiteMode.Lax;  // CSRF protection
    
    // How long the authentication cookie lasts
    options.ExpireTimeSpan = TimeSpan.FromMinutes(30);
    options.SlidingExpiration = true;  // Refresh expiration on activity
})
// ─────────────────────────────────────────────────────────────────────
// Google OAuth Configuration
// ─────────────────────────────────────────────────────────────────────
.AddGoogle(options =>
{
    // ┌─────────────────────────────────────────────────────────────┐
    // │ CLIENT ID: Public identifier for your app                   │
    // │ - Visible in browser/network requests                       │
    // │ - Not secret, can be exposed                                │
    // └─────────────────────────────────────────────────────────────┘
    options.ClientId = builder.Configuration["Google:ClientId"]!;
    
    // ┌─────────────────────────────────────────────────────────────┐
    // │ CLIENT SECRET: Private key for your app                     │
    // │ - NEVER expose to frontend                                  │
    // │ - Only used in backend token exchange                       │
    // │ - Store in User Secrets (dev) or Azure Key Vault (prod)    │
    // └─────────────────────────────────────────────────────────────┘
    options.ClientSecret = builder.Configuration["Google:ClientSecret"]!;
    
    // ┌─────────────────────────────────────────────────────────────┐
    // │ CALLBACK PATH: Where Google redirects after user login      │
    // │ - Must match EXACTLY what's configured in Google Console   │
    // │ - Example: https://yourdomain.com/api/auth/google/callback │
    // └─────────────────────────────────────────────────────────────┘
    options.CallbackPath = "/api/auth/google/callback";
    
    // ┌─────────────────────────────────────────────────────────────┐
    // │ SCOPES: What permissions we request from Google             │
    // │ - openid: Basic authentication (added automatically)        │
    // │ - profile: Name, picture, etc.                             │
    // │ - email: Email address                                      │
    // │ More scopes: https://developers.google.com/identity/scopes │
    // └─────────────────────────────────────────────────────────────┘
    options.Scope.Add("profile");
    options.Scope.Add("email");
    // options.Scope.Add("https://www.googleapis.com/auth/calendar.readonly");  // Example: Calendar access
    
    // ┌─────────────────────────────────────────────────────────────┐
    // │ SAVE TOKENS: Store Google's access token for later use      │
    // │ - Enables calling Google APIs on behalf of user            │
    // │ - Access via HttpContext.GetTokenAsync("access_token")     │
    // └─────────────────────────────────────────────────────────────┘
    options.SaveTokens = true;
    
    // ┌─────────────────────────────────────────────────────────────┐
    // │ EVENTS: Hooks into authentication flow                       │
    // │ - Useful for logging, custom validation, etc.               │
    // └─────────────────────────────────────────────────────────────┘
    options.Events.OnCreatingTicket = context =>
    {
        // Custom logic after successful Google authentication
        var email = context.Principal?.FindFirst(c => c.Type == System.Security.Claims.ClaimTypes.Email)?.Value;
        Console.WriteLine($"User authenticated: {email}");
        return Task.CompletedTask;
    };
    
    options.Events.OnRemoteFailure = context =>
    {
        // Handle authentication failures
        context.Response.Redirect("/login?error=google_auth_failed");
        context.HandleResponse();
        return Task.CompletedTask;
    };
})
// ─────────────────────────────────────────────────────────────────────
// JWT Bearer Authentication Configuration
// ─────────────────────────────────────────────────────────────────────
.AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options =>
{
    // ┌─────────────────────────────────────────────────────────────┐
    // │ JWT VALIDATION PARAMETERS                                    │
    // │ These settings determine how incoming JWT tokens are        │
    // │ validated for authenticity and expiration                   │
    // └─────────────────────────────────────────────────────────────┘
    options.TokenValidationParameters = new TokenValidationParameters
    {
        // Validate the token signature to ensure it wasn't tampered with
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!)
        ),
        
        // Validate the 'iss' (issuer) claim - who created the token
        ValidateIssuer = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        
        // Validate the 'aud' (audience) claim - who the token is for
        ValidateAudience = true,
        ValidAudience = builder.Configuration["Jwt:Audience"],
        
        // Validate the token hasn't expired
        ValidateLifetime = true,
        
        // Allow 5 minutes clock skew for expiration time
        // Accounts for time differences between servers
        ClockSkew = TimeSpan.FromMinutes(5)
    };
    
    // ┌─────────────────────────────────────────────────────────────┐
    // │ EVENT HANDLERS for JWT authentication                        │
    // └─────────────────────────────────────────────────────────────┘
    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            // Log authentication failures for debugging
            if (context.Exception.GetType() == typeof(SecurityTokenExpiredException))
            {
                context.Response.Headers.Append("Token-Expired", "true");
            }
            Console.WriteLine($"JWT Authentication failed: {context.Exception.Message}");
            return Task.CompletedTask;
        },
        
        OnTokenValidated = context =>
        {
            // Token is valid - can add custom claims or logging here
            var email = context.Principal?.FindFirst(System.Security.Claims.ClaimTypes.Email)?.Value;
            Console.WriteLine($"JWT validated for user: {email}");
            return Task.CompletedTask;
        }
    };
});

// ═══════════════════════════════════════════════════════════════════════
// SECTION 4: AUTHORIZATION POLICIES (Optional but recommended)
// ═══════════════════════════════════════════════════════════════════════
builder.Services.AddAuthorization(options =>
{
    // Default policy: Require authenticated user
    options.FallbackPolicy = new Microsoft.AspNetCore.Authorization.AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    
    // Custom policy example: Email verified users only
    options.AddPolicy("EmailVerified", policy =>
        policy.RequireClaim("email_verified", "true"));
    
    // Custom policy example: Admin users
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
});

// ═══════════════════════════════════════════════════════════════════════
// SECTION 5: CORS CONFIGURATION
// ═══════════════════════════════════════════════════════════════════════
// Required if your frontend is on a different domain/port
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins(
                "http://localhost:3000",      // React dev server
                "http://localhost:4200",      // Angular dev server
                "https://yourfrontend.com"    // Production frontend
            )
            .AllowAnyMethod()                  // GET, POST, PUT, DELETE, etc.
            .AllowAnyHeader()                  // Authorization, Content-Type, etc.
            .AllowCredentials();               // Allow cookies
    });
});

// ═══════════════════════════════════════════════════════════════════════
// SECTION 6: FASTENDPOINTS CONFIGURATION
// ═══════════════════════════════════════════════════════════════════════
builder.Services.AddFastEndpoints();

// ═══════════════════════════════════════════════════════════════════════
// BUILD THE APPLICATION
// ═══════════════════════════════════════════════════════════════════════
var app = builder.Build();

// ═══════════════════════════════════════════════════════════════════════
// MIDDLEWARE PIPELINE CONFIGURATION
// ═══════════════════════════════════════════════════════════════════════
/*
 * MIDDLEWARE ORDER IS CRITICAL!
 * The order matters because each middleware processes the request in order
 * and the response in reverse order.
 */

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();  // Detailed error pages in dev
}
else
{
    app.UseExceptionHandler("/error");  // Generic error page in production
    app.UseHsts();                      // HTTP Strict Transport Security
}

app.UseHttpsRedirection();  // Redirect HTTP to HTTPS

// CORS must come before Authentication
app.UseCors("AllowFrontend");

// Authentication reads the token/cookie from request
app.UseAuthentication();

// Authorization checks if user has permission for the requested resource
app.UseAuthorization();

// FastEndpoints handles routing and endpoint execution
app.UseFastEndpoints(config =>
{
    config.Endpoints.RoutePrefix = "api";  // All endpoints start with /api
    
    // Configure JSON serialization
    config.Serializer.Options.PropertyNamingPolicy = 
        System.Text.Json.JsonNamingPolicy.CamelCase;
});

// ═══════════════════════════════════════════════════════════════════════
// DATABASE MIGRATION (Development only)
// ═══════════════════════════════════════════════════════════════════════
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await dbContext.Database.MigrateAsync();  // Apply pending migrations
}

// ═══════════════════════════════════════════════════════════════════════
// START THE APPLICATION
// ═══════════════════════════════════════════════════════════════════════
app.Run();
```

---

### 📄 2. appsettings.json - Configuration

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=YourAppDb;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  
  "Google": {
    "ClientId": "YOUR_CLIENT_ID.apps.googleusercontent.com",
    "ClientSecret": "YOUR_CLIENT_SECRET"
  },
  
  "Jwt": {
    "SecretKey": "your-super-secret-key-must-be-at-least-32-characters-long-for-security",
    "Issuer": "YourApp",
    "Audience": "YourAppUsers",
    "ExpirationMinutes": 10080
  },
  
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.Authentication": "Debug"
    }
  },
  
  "AllowedHosts": "*"
}
```

**🔒 SECURITY NOTE**: Never commit secrets to source control!

```bash
# For Development: Use User Secrets
dotnet user-secrets init
dotnet user-secrets set "Google:ClientId" "your-client-id"
dotnet user-secrets set "Google:ClientSecret" "your-secret"
dotnet user-secrets set "Jwt:SecretKey" "your-jwt-secret"

# For Production: Use environment variables or Azure Key Vault
```

---

### 📄 3. Auth Models

```csharp
// Features/Auth/Models/AuthModels.cs

namespace YourApp.Features.Auth.Models;

/// <summary>
/// Response returned after successful Google authentication
/// Contains JWT token and user information
/// </summary>
public class GoogleAuthResponse
{
    /// <summary>
    /// Your application's JWT token (NOT Google's token)
    /// This is what the frontend stores and uses for API calls
    /// </summary>
    public string AccessToken { get; set; } = string.Empty;
    
    /// <summary>
    /// Token type - always "Bearer" for JWT tokens
    /// Used in Authorization header: "Bearer {token}"
    /// </summary>
    public string TokenType { get; set; } = "Bearer";
    
    /// <summary>
    /// When the token expires (Unix timestamp in seconds)
    /// </summary>
    public long ExpiresAt { get; set; }
    
    /// <summary>
    /// User's email address from Google
    /// </summary>
    public string Email { get; set; } = string.Empty;
    
    /// <summary>
    /// User's full name from Google
    /// </summary>
    public string Name { get; set; } = string.Empty;
    
    /// <summary>
    /// URL to user's profile picture from Google
    /// </summary>
    public string Picture { get; set; } = string.Empty;
    
    /// <summary>
    /// Your application's internal user ID
    /// </summary>
    public string UserId { get; set; } = string.Empty;
}

/// <summary>
/// Error response for authentication failures
/// </summary>
public class AuthErrorResponse
{
    public string Error { get; set; } = string.Empty;
    public string ErrorDescription { get; set; } = string.Empty;
    public int StatusCode { get; set; }
}
```

---

### 📄 4. Google Sign-In Endpoint

```csharp
// Features/Auth/GoogleSignInEndpoint.cs

using FastEndpoints;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Google;

namespace YourApp.Features.Auth;

/// <summary>
/// STEP 1 of OAuth Flow: Initiates Google Sign-In
/// 
/// When user hits this endpoint, they are redirected to Google's login page
/// 
/// FLOW:
/// User → This Endpoint → Google Login Page → User Authenticates → Google Callback
/// </summary>
public class GoogleSignInEndpoint : EndpointWithoutRequest
{
    public override void Configure()
    {
        // ┌─────────────────────────────────────────────────────────┐
        // │ ENDPOINT CONFIGURATION                                   │
        // └─────────────────────────────────────────────────────────┘
        Get("/auth/google/signin");
        
        // Allow anonymous access - anyone can initiate login
        AllowAnonymous();
        
        // Optional: Add rate limiting to prevent abuse
        // Throttle(10, 60);  // 10 requests per 60 seconds
        
        // Endpoint metadata for documentation
        Description(d => d
            .WithTags("Authentication")
            .WithSummary("Initiate Google OAuth Sign-In")
            .WithDescription("Redirects user to Google's OAuth login page")
            .Produces(302)  // HTTP 302 Redirect
        );
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        /*
         * ═══════════════════════════════════════════════════════════
         * WHAT HAPPENS HERE:
         * ═══════════════════════════════════════════════════════════
         * 
         * 1. Create AuthenticationProperties with callback URL
         * 2. Call ChallengeAsync with Google scheme
         * 3. ASP.NET Core redirects user to:
         *    
         *    https://accounts.google.com/o/oauth2/v2/auth?
         *      client_id={YOUR_CLIENT_ID}&
         *      redirect_uri=https://yourapi.com/api/auth/google/callback&
         *      response_type=code&
         *      scope=openid%20profile%20email&
         *      state={RANDOM_CSRF_TOKEN}
         * 
         * 4. User sees Google login page
         * 5. After login, Google redirects to callback URL