# Auth & JWT — Full Architecture Reference

> Read this when touching auth, JWT, or ICurrentUserService.

## Why ICurrentUserService Exists

Every handler that does something protected needs to know: who is making this request? Their user ID, their tenant, their role. Without this service, every handler would reach directly into `HttpContext` — that is Infrastructure code leaking into Application, breaking Clean Architecture.

The **interface** lives in Application (the contract). The **implementation** lives in Infrastructure (reads from HttpContext). Handlers inject the interface and never know about HTTP.

## The Full JWT Request Cycle

```
LOGIN:
  POST /api/auth/login (email + password)
  → LoginCommandHandler reads User from DB
  → BCrypt.Verify(password, user.PasswordHash) → true
  → Build JWT claims: uid, tid, ClaimTypes.Role, ctype
  → Sign with our secret key
  → Return JWT string to client

EVERY SUBSEQUENT REQUEST:
  Client sends: Authorization: Bearer <jwt>
  → JWT middleware validates signature + expiry
  → Decodes payload → builds ClaimsPrincipal
  → Assigns to HttpContext.User automatically
  → CurrentUserService reads: HttpContext.User.FindFirstValue("uid")
  → Handler uses: _currentUserService.UserId
```

## Our JWT Claim Names

| Claim key | Contains | Example |
|---|---|---|
| `"uid"` | User.Id (our DB primary key) | `Guid` |
| `"tid"` | User.TenantId (Organisation they belong to) | `Guid` |
| `ClaimTypes.Role` | User.Role enum value | `"SafetyOfficer"` |
| `"ctype"` | User.CompanyType enum value | `"Client"` |

## BCrypt — Why There Is No PasswordSalt Column

BCrypt embeds the salt inside the hash string itself. The output of `BCrypt.HashPassword("password")` is a single string like `"$2a$11$<salt+hash combined>"`. Verification is `BCrypt.Verify("password", storedHash)` — BCrypt extracts the salt internally. A separate `PasswordSalt` column is an artifact of older SHA-256 hashing.

## JWT Middleware Setup

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Secret"]!)),
            ValidateIssuer = false,
            ValidateAudience = false
        };
    });

app.UseAuthentication(); // decodes JWT → HttpContext.User
app.UseAuthorization();  // enforces [Authorize] attributes
```

## Soft-Deleted User with Active JWT — Known Gap

If a user is soft-deleted, they can still use their existing JWT for up to 12 hours (current expiry). `HasQueryFilter(x => !x.IsDeleted)` blocks new logins immediately but does not invalidate existing tokens.

**Fix (Phase 8):** When a user is soft-deleted, write their JWT `jti` claim to a Redis blocklist with TTL = remaining token expiry. `TenantResolutionMiddleware` checks the blocklist on every authenticated request.

## What Changes When Azure AD / Okta Arrives (Phase 14)

Their token is used **once at login** as proof of identity. After that, we issue our own JWT with our own claim names. `CurrentUserService` never changes.

## Refresh Tokens — Phase 14 Decision

Phase 4 uses simple re-login when the JWT expires (12 hours). Phase 14 adds short-lived access tokens (15–60 min) + long-lived refresh tokens (7–30 days). Client sends refresh token to `POST /api/auth/refresh` — we issue a new access token. User never sees a logout.
