# MAISAAS – Deep Technical Analysis



**Project path:** `c:\Users\parth.mashroo\Downloads\MAISAAS_new\MAISAAS`

**Target framework:** ASP.NET Core 3.1 across all projects

**Analysis date:** 2026-03-13



---



## 1. Solution Structure



**File:** `ErpContractorPortal\ErpContractorPortal.sln`



The solution contains 25+ projects organized into logical groups:



### Core Projects

| Project | Type | Purpose |

|---|---|---|

| `ErpContractorPortal.Models` | Class Library | Domain entities (EF Code-First) |

| `ErpContractorPortal.Repository` | Class Library | EF Core DbContext + Dapper + repositories |

| `ErpContractorPortal.Service` | Class Library | Business logic layer |

| `ErpContractorPortal.Infrastructure` | Class Library | Cross-cutting: identity helper, caching, filters |

| `ErpContractorPortal.ViewModels` | Class Library | DTOs, validators, constants |

| `ErpContractorPortal.Encryption` | Class Library | AES-256 utility |

| `ErpContractorPortal.Mail` | Class Library | Email sending + template compiler |

| `ErpContractorPortal.MessageQueue` | Class Library (netstandard2.1) | RabbitMQ abstraction |

| `ErpContractorPortal.ServerCache` | Class Library | Redis IDistributedCache wrapper |



### Runnable Services

| Project | Type | Purpose |

|---|---|---|

| `ErpContractorPortal.Security` | ASP.NET Core API | IdentityServer4 auth server |

| `ErpContractorPortal.WebAPI` | ASP.NET Core API | Main REST API |

| `ErpContractorPortal.Scheduler` | ASP.NET Core API | Quartz.NET job host |

| `ErpContractorPortal.MessageProcessor` | Worker Service | RabbitMQ background consumer |

| `ErpContractorPortal` (Contractor MVC) | ASP.NET Core MVC | Contractor-facing web portal |

| `ErpClientPortal` | ASP.NET Core MVC | Client-facing web portal |

| `ErpCheckinPortal` | ASP.NET Core MVC | Site check-in/out portal |

| `ErpIntegrationAPI` | ASP.NET Core API | External integration API |



### InternalTools Group

- `ErpContractorPortal.CustomReportViewModels` – Report DTO models

- `ErpContractorPortal.PowerBiIntegration` – PowerBI embedded

- `ErpContractorPortal.SCIM` – SCIM 2.0 protocol support (Microsoft.SystemForCrossDomainIdentityManagement)



### Dependency Hierarchy (bottom to top)

```

Encryption

└── Models

├── Repository (+ Dapper, EF Core)

│ └── Service (+ AutoMapper, EPPlus)

│ └── WebAPI / Scheduler

├── ViewModels

│ └── Infrastructure (+ Redis)

│ └── WebAPI / MVC Portals

└── Mail (+ Handlebars)

MessageQueue (netstandard2.1)

└── MessageProcessor (BackgroundService)

```



---



## 2. Authentication & Authorization



### Auth Server: IdentityServer4



**File:** `ErpContractorPortal.Security\Startup.cs`



```csharp

services.AddIdentityServer()

.AddDeveloperSigningCredential()

.AddCustomResourceStore() // CustomResourceStore

.AddCustomClientStore() // CustomClientStore

.AddProfileService<ProfileService>()

.AddResourceOwnerValidator<ResourceOwnerPasswordValidator>();

```



Azure AD external login is also configured:

```csharp

services.AddAuthentication()

.AddOpenIdConnect("aad", "Azure AD", options => {

options.Authority = "https://login.microsoftonline.com/{tenantId}";

// PKCE flow to AAD

});

```



Cookie lifetime: 3 days, sliding expiration.



### OAuth Clients (Hardcoded)



**File:** `ErpContractorPortal.Security\Identity\CustomClientStore.cs`



All clients are hardcoded (no DB-driven client store):



| ClientId | Flow | Purpose |

|---|---|---|

| `"client"` | ResourceOwnerPassword | API-to-API calls |

| `"cf16de7eceff4486b0583f1ac19b8ce8"` | AuthorizationCode | Contractor MVC Portal |

| *(client portal id)* | AuthorizationCode | Client MVC Portal |

| *(checkin portal id)* | AuthorizationCode | CheckIn Portal |

| *(integration id)* | ClientCredentials | External Integration API |

| `"CheckinApp"` | ResourceOwnerPassword | Mobile checkin app |

| `"MaiApp"` | ResourceOwnerPassword | Mobile MAI app |



All clients: `AlwaysIncludeUserClaimsInIdToken = true`, access token lifetime = 86400 seconds (24 hours).



### Token Validation (ResourceOwnerPasswordValidator)



**File:** `ErpContractorPortal.Security\Identity\ResourceOwnerPasswordValidator.cs`



1. Calls `IUserService.GetUserAsync(username, password)`

2. Checks `ClientId == "CheckinApp"` to restrict access by VisitorType

3. Issues claims in the JWT:

- `userCode` (UserId as Guid)

- `name`, `RoleType`, `CompanyId`, `EmployeeId`

- `CompanyName`, `CompanyLogoId`, `Email`

- `DefaultRoleId`, `role` (role name string)

- `ImpersonateUser`

4. Stores permissions in Redis at key `{userId}_Permissions` at login time

5. Also caches: `HasWPRequestPermission`, `IsAllowContractorsForNC`, `ClientOnBoarding`, `CompanyContract`, `IsExpired`



### Profile Service



**File:** `ErpContractorPortal.Security\Identity\ProfileService.cs`



Called on `/connect/userinfo`. Re-issues all claims from DB. Also adds:

- `VisitorType` claim for CheckinApp clients

- `LanguageId` and `HasWPRequestPermission` for MaiApp clients



### JWT Validation on API



**File:** `ErpContractorPortal.WebAPI\Startup.cs`



```csharp

options.TokenValidationParameters = new TokenValidationParameters {

ValidateAudience = false,

ValidateIssuer = false,

ValidateLifetime = false, // SECURITY ISSUE: expired tokens accepted

ValidateIssuerSigningKey = false,

IssuerSigningKey = new SymmetricSecurityKey(

Encoding.ASCII.GetBytes("secret")) // SECURITY ISSUE: weak key in config

};

```



`ValidateLifetime = false` means expired tokens are always accepted. The signing key `"secret"` is stored in `appsettings.json` as `AppSettings:SecretKey`.



### Permission System



**File:** `ErpContractorPortal.Infrastructure\Implementation\IdentityHelper.cs`



Permissions are NOT stored in JWT claims. They are loaded from Redis cache:



```csharp

// Key pattern: "{userId}_Permissions"

// Populated at login in ResourceOwnerPasswordValidator

public bool HasPermission(string code) {

var permissions = _cache.Get<UserPermissionViewModel>("{userId}_Permissions");

return permissions.Any(p => p.Code == code);

}

```



Helper methods:

- `HasPermission(code)` – exact permission check

- `HasPermissionOrAdmin(code)` – allows SuperAdmin through

- `HasPermissionOrCreator(userId, code)` – allows the record creator through



RoleType enum: `SuperAdmin`, `Client`, `Contractor`



### Permission Filter



**File:** `ErpContractorPortal.Infrastructure\Filters\PermissionAccessFilter.cs`



```csharp

[PermissionAccessAttribute("WORK_PERMIT_CREATE", canAdminAccess: true)]

```



Action filter that checks IIdentityHelper at runtime. Redirects to `PageError/UnAuthorized` on failure.



### Integration API Guard



**File:** `ErpContractorPortal.Infrastructure\Filters\IntegrationServiceOnlyAttribute.cs`



Checks for HTTP header `ValidatedClient: 4c703c0e95ce4a87be3d2e558447da72` — a hardcoded shared secret.



---



## 3. Database & Data Access Patterns



### EF Core DbContext



**File:** `ErpContractorPortal.Repository\Data\EfDataContext.cs`



- Extends `DbContext`, implements `IDataContext`

- Lazy loading proxies enabled

- `OnModelCreating`: applies all configurations from the assembly where the class name starts with `"Ef"` and ends with `"Map"` (convention-based EF configuration discovery)

- No global query filters — tenant filtering is applied manually in every service/repository call



### Connection String



**File:** `ErpContractorPortal.Repository\RepositoryExtensions.cs`



```csharp

IEncryptionManager manager = new EncryptionManager();

var connectionString = manager.Decrypt("Mnandoe#orp",

Configuration.GetConnectionString("Portal"));

```



The connection string in `appsettings.json` is AES-encrypted. Key `"Mnandoe#orp"` is hardcoded.



### Entity Configurations



Stored in `ErpContractorPortal.Repository\Data\Configrations\` (note: typo in folder name).



Pattern: each file is named `Ef{Entity}Map.cs` and extends `BaseEntityTypeConfiguration<T, TKey>`.



Example — `EfWorkPermitMap.cs`:

```csharp

builder.ToTable("WorkPermit");

builder.HasOne(x => x.ClientCompany).WithMany().HasForeignKey(x => x.ClientCompanyId);

builder.HasOne(x => x.TypeOfWork).WithMany().HasForeignKey(x => x.TypeOfWorkId);

```



Example — `EfEmployeeMap.cs`:

```csharp

builder.ToTable("Employees");

// Self-referential supervisor FK:

builder.HasOne(x => x.Supervisor).WithMany().HasForeignKey(x => x.SupervisorId);

```



### Repository Pattern



**File:** `ErpContractorPortal.Repository\Core\IBaseRepository.cs`



```csharp

public interface IBaseRepository<T, TKey> {

IQueryable<T> GetAll();

IQueryable<T> GetAll(Expression<Func<T, bool>> predicate);

IQueryable<T> GetAllNoTracking();

T GetById(TKey id);

Task<T> GetByIdAsync(TKey id);

void Insert(T entity);

void BulkInsert(IEnumerable<T> entities);

void Update(T entity);

void Delete(T entity);

void BulkDelete(IEnumerable<T> entities);

void BulkUpdate(IEnumerable<T> entities);

}

```



Repositories are registered by reflection in `RepositoryExtensions.cs`. All concrete repositories named `XxxRepository` implementing `IXxxRepository` are auto-discovered and registered as Transient.



### Unit of Work



**File:** `ErpContractorPortal.Repository\Core\TransactionRepository.cs`



```csharp

public interface ITransactionRepository {

void Commit();

Task CommitAsync();

}

// Delegates to IDataContext.SaveChanges()

```



Pattern in services: insert/update entities via repositories, then call `_transactionRepository.CommitAsync()`.



### Dapper for Stored Procedures



**File:** `ErpContractorPortal.Repository\Data\DapperProcedureContext.cs`



```csharp

public interface IProcedureContext {

IEnumerable<T> GetCollection<T>(string procName, DynamicParameters parameters, ...);

bool InsertOrUpdate(string procName, DynamicParameters parameters);

DataSet GetDataSet(string procName, DynamicParameters parameters);

}

```



Each method opens a new `SqlConnection` per call. Exception in `InsertOrUpdate` is swallowed and returns `false`.



Use cases: complex filtered list queries, questionnaire answer batch updates (`UpdateQuestionAnswered` SP), custom report generation.



### TVP Helper



```csharp

// BaseRepository<T, TKey>

public static DataTable ToFilterDataTable<TM>(IEnumerable<TM> items)

```



Converts a list to a `DataTable` for use as a SQL Server Table-Valued Parameter.



### Mixed ORM Strategy Summary

- **EF Core**: CRUD operations, navigation properties, change tracking

- **Dapper**: Complex queries, stored procedures, batch operations, report data retrieval



---



## 4. Service Layer Patterns



**File:** `ErpContractorPortal.Service\ServiceLayerExtension.cs`



All services registered by convention (reflection): `IXxxService → XxxService` as Transient.



AutoMapper profiles registered here: `AccountMappingProfile`, `AdminMappingProfile`. 
entDetailViewModel` with file path and metadata. Files are served by returning `File(relativePath, mimeType, originalName)`.



**Issue:** Files stored on local disk under wwwroot — not cloud storage. No CDN, no blob storage.



---



## 6. Email System



### Email Service



**File:** `ErpContractorPortal.Mail\Implementation\CustomEmailService.cs`



1. Decrypts SMTP credentials from config at construction (AES via EncryptionManager)

2. `SendMailAsync(model)`: looks up `EmailTemplate` by `model.TemplateId` from DB

3. Uses Handlebars.Net to substitute tokens: `{{TokenName}}`

4. Checks `ConfigSetting` table for:

- `EnableSMTPEmail` — master on/off switch

- `IsSMTPDevelopmentMode` — redirects all emails to `DeveloperModeEmail`

- `EnableEmailForCompany` — per-company override of dev mode

5. Sends via `System.Net.Mail.SmtpClient` (not SendGrid despite SendGrid NuGet package being present)

6. Logs all emails to `EmailData` table regardless of success/failure



### Template Compiler



**File:** `ErpContractorPortal.Mail\Implementation\HandleBarTemplateComplier.cs`



```csharp

public string Compile(string templateHtml, object tokens) {

var template = Handlebars.Compile(templateHtml);

return template(tokens);

}

```



Templates are stored in the database in the `EmailTemplate` table (not on disk).



### SendGrid



`SendGrid 9.22.0` is listed in `ErpContractorPortal.Mail.csproj` but actual sending uses `SmtpClient`. SendGrid appears unused / commented out.



### Developer Mode



When `IsSMTPDevelopmentMode = true`, all outgoing emails redirect to `DeveloperModeEmail` UNLESS the company has `EnableEmailForCompany = true`. This prevents accidental emails to real users in dev/staging.



---



## 7. Background Jobs & Scheduling



### Quartz.NET



**File:** `ErpContractorPortal.Scheduler\Startup.cs`



```csharp

services.AddQuartz(q => {

q.SchedulerId = Constants.Common.SCHEDULERNAME;

q.UseMicrosoftDependencyInjectionJobFactory();

q.UsePersistentStore(s => {

s.UseSqlServer(sqlServer => {

sqlServer.TablePrefix = "QRTZ_"; // Quartz tables in same DB

});

s.UseJsonSerializer();

s.UseClustering(c => {

c.CheckinInterval = TimeSpan.FromSeconds(10);

c.CheckinMisfireThreshold = TimeSpan.FromSeconds(20);

});

});

q.UseDefaultThreadPool(tp => { tp.MaxConcurrency = 10; });

});

```



Clustering enabled — multiple Scheduler instances can run safely against the same SQL Server store.



### ReportJob



**File:** `ErpContractorPortal.Scheduler\Core\ReportJob.cs`



```csharp

public async Task Execute(IJobExecutionContext context) {

var reportGroupId = context.JobDetail.JobDataMap.GetLong("ReportGroupId");

// Check if report group is active

// Add trigger record to DB

// Send message to RabbitMQ — MessageProcessor handles actual generation

}

```



### CorrectiveActionJob



**File:** `ErpContractorPortal.Scheduler\Core\CorrectiveActionJob.cs`



```csharp

public async Task Execute(IJobExecutionContext context) {

var schedulerKeyId = context.JobDetail.JobDataMap.GetLong("JobId");

// Get CA schedule details

// Send CANotificationJob messages to RabbitMQ

}

```



Jobs delegate to RabbitMQ messages rather than performing work directly.



---



## 8. Message Queue / Event System



### RabbitMQ Manager



**File:** `ErpContractorPortal.MessageQueue\RabbitMessageQueueManager.cs`



```csharp

public void SendMessage(string queueName, object data) {

using var connection = _factory.CreateConnection();

using var channel = connection.CreateModel();

channel.QueueDeclare(queueName, durable: false, exclusive: false, autoDelete: false);

var body = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(data));

channel.BasicPublish(exchange: "", routingKey: queueName, body: body);

}



public void ListionQueue(string queueName, Action<string> callback) {

var channel = connection.CreateModel();

channel.QueueDeclare(queueName, durable: false, ...);

var consumer = new EventingBasicConsumer(channel);

consumer.Received += (model, ea) => {

callback(Encoding.UTF8.GetString(ea.Body.ToArray()));

};

channel.BasicConsume(queue: queueName, autoAck: true, consumer: consumer);

}

```



**Issues:**

- New connection created per `SendMessage` call (no connection pooling)

- `durable: false` — queues and messages lost on RabbitMQ restart

- `autoAck: true` — messages acknowledged before processing; no retry on failure

- No dead-letter exchanges or error queues



### Queue Names



**File:** `ErpContractorPortal.ViewModels\Core\Constants.cs`



Queue names use environment suffix: `"{QueueName}_{environment}"` e.g., `"RequestWorkPermitApproval_{0}"` where `{0}` is replaced with `Constants.Common.ENVIRONMENT`.



### Message Processor (BackgroundService)



**File:** `ErpContractorPortal.MessageProcessor\BackgroudTaskProcessor.cs`



```csharp

private Dictionary<string, Action<string>> _messageStrategies = new() {

["WorkPermitApproval_{env}"] = HandleWorkPermitApproval,

["WorkPermitRejection_{env}"] = HandleWorkPermitRejection,

// ... 25+ handlers

};

```



Handles 25+ queue types including:

- WorkPermit approval/rejection/completion/closure

- Email notifications (password reset, 2FA codes)

- Document events

- Vendor rating

- Induction notifications

- Check-in/check-out events

- CA (Corrective Action) notifications

- Questionnaire completion



Each handler creates a DI scope:

```csharp

private void HandleWorkPermitApproval(string message) {

using var scope = _serviceScopeFactory.CreateScope();

var service = scope.ServiceProvider.GetService<IWorkPermitService>();

// process message

}

```



`IServiceScopeFactory` is used correctly to resolve scoped services within the singleton BackgroundService.



---



## 9. Caching



### Redis Cache Wrapper



**File:** `ErpContractorPortal.ServerCache\Implementation\CommonDataCache.cs`



```csharp

public interface IDataCache {

List<T> Get<T>(string key);

T GetSingle<T>(string key);

void Set<T>(string key, List<T> data);

void SetSingle<T>(string key, T data);

void Set(string key, string value);

string Get(string key);

}

```



Uses `IDistributedCache` (Redis) with **BinaryFormatter** serialization:



```csharp

using var ms = new MemoryStream();

new BinaryFormatter().Serialize(ms, data);

_cache.Set(key, ms.ToArray());

```



**Issue:** `BinaryFormatter` is obsolete and removed in .NET 5+. This blocks migration to modern .NET. Objects must be `[Serializable]`.



### What is Cached



1. **User permissions** — key `{userId}_Permissions` (set at login in ResourceOwnerPasswordValidator)

2. **Localization strings** — via `RedisLocalizeManager` (singleton)

3. **Company data, contract info** — `CompanyContract`, `IsExpired` flags per user

4. **Common reference data** — via `IDataCache` (group codes, site areas etc.)



### Redis Configuration



**File:** `ErpContractorPortal.WebAPI\appsettings.json`



```json

"ConnectionStrings": {

"RedisCache": "152.67.4.152:6379"

}

```



Registered in `InfrastructureExtensions.cs` with optional auth (password supplied only if present in config).



---
## 10. File Storage



### Strategy: Local Disk under wwwroot



**File:** `ErpContractorPortal.WebAPI\Controllers\AttachmentController.cs`



Files saved to: `{ContentRootPath}/wwwroot/{ModuleFolder}/File_{Guid}{extension}`



Module folders:

```

Documents, WorkPermit, Company, Employee, Skills, NonConformance,

CorrectiveAction, MOM, MicrosoftLoginCertificate, Audit

```



Files served via the `Download` action which reads the path and returns `File(path, mimeType, fileName)`.



### Attachment Model



Fields (inferred from usage): file path, original name, MIME type, module type, FK to parent record.



**Issues:**

- No cloud storage (no Azure Blob, no S3)

- Files lost on server redeployment/migration

- No file size limits enforced at upload

- No virus scanning

- No CDN for delivery



---



## 11. Multi-Tenancy



### Primary Mechanism: CompanyId Claim



Every JWT token contains a `CompanyId` claim (Guid). `IIdentityHelper.CompanyId` reads this claim and is injected into all services.



```csharp

// In every service GetAll() method:

var companyId = _identityHelper.CompanyId;

var results = _repository.GetAll(x => x.CompanyId == companyId);

```



There is **no global query filter** at the DbContext level — tenant filtering is applied manually in every query.



### Client-Contractor Relationship



**File:** `ErpContractorPortal.Models\Admin\ClientContractor.cs`



```csharp

public class ClientContractor {

public Guid ContractorCompanyId { get; set; }

public Guid ClientCompanyId { get; set; }

public Guid? SiteID { get; set; }

public bool IsSubContractor { get; set; }

public int Level { get; set; }

}

```



Core multi-tenancy junction table. A Contractor company can work for multiple Client companies. Work Permits belong to `ClientCompanyId`.



### RoleType and Tenant Scoping



```csharp

public enum RoleType {

SuperAdmin = 1, // sees all companies

Client = 2, // sees their own company + their contractors

Contractor = 3 // sees their own company's data within client context

}

```



SuperAdmin bypasses company filtering via `HasPermissionOrAdmin()`.



### Contract Expiry Guard



**File:** `ErpContractorPortalWeb\Filters\CompanyContractFilter.cs`



```csharp

public void OnActionExecuting(ActionExecutingContext context) {

if (_identityHelper.IsContractExpired) {

context.Result = new RedirectToActionResult("Index", "ContractorPlan", null);

}

}

```



Applied globally via `BaseController`. `IsContractExpired` read from Redis cache (cached at login).



---



## 12. Questionnaire / Assessment Module



### Core Entities



**File:** `ErpContractorPortal.Models\Questionaire\`



```

Questionaire — definition (Title, IsScorable, TotalScore, QuestionaireModuleId, PkId)

QuestionaireSection — groups questions within a questionnaire

Questions — individual question definitions

QuestionOptions — multiple choice options for a question

QuestionaireAttend — a specific user's attempt at a questionnaire

QuestionaireAttendedQuestion — a specific answer to one question in one attempt

```



### Questionnaire Fields



`Questionaire`:

- `PkId` — string FK to the owning record (WorkPermit ID, Induction ID etc.)

- `QuestionaireModuleId` — which module owns this questionnaire

- `IsScorable`, `TotalScore`

- `IsSubsectionAvailable`



`Questions`:

- `AnswerTypeId` — text, multiple choice, table, file upload, etc.

- `Score`, `IsRequired`, `IsScorable`

- `NumberOfRows` — for table-type answers

- `IsRemarkVisible`, `IsUploadVisible`

- `RelatedQuestionId` — conditional branching (skip logic)

- `DocumentTypeId` — for document upload questions

- `DisplayOrder`



`QuestionaireAttendedQuestion`:

- `AnswerContent` — simple text/choice answer

- `AnswerJSON` — complex answers (tables, multi-select stored as JSON string)

- `ScoreEarned` vs `Score` — actual vs possible

- `IsSkipped` — for branching/skip logic



### Integration with WorkPermit



WorkPermit questionnaire answers are batch-saved via stored procedure:

```csharp

_procedureContext.InsertOrUpdate("UpdateQuestionAnswered", parameters);

```



The SP takes a TVP of question/answer pairs. When WorkPermit status is reverted, questionnaire answers are cleared based on which "CheckType" they belong to.



### Scoring



`IQuestionaireScoreManager` is registered separately in `ServiceLayerExtension.cs`:

```csharp

services.AddTransient<IQuestionaireScoreManager, QuestionaireScoreManager>();

```



---



## 13. Work Permit Module



The most complex module in the system.



### Entity



**File:** `ErpContractorPortal.Models\WorkPermit\WorkPermit.cs`



Key fields:

- `ClientCompanyId` (Guid) — which client this permit belongs to

- `Number` — auto-generated permit number

- `StartDate`, `EndDate`

- `ZoneId`, `ExactLocation`

- `TypeOfWorkId` → GroupCode lookup

- `StatusId` — current status

- `PermitType` — enum (standard, hot work, confined space, etc.)

- `HasEquipment`, `HasIsolation`, `HasGasTest` — flags driving sub-permit inclusion

- `IsRams`, `SafePlanOfAction`, `HasPPERequired`



Navigation collections:

```

SubPermits, Assignees, Steps, Equipments, Isolations, GasTests,

WorkPermitAttachments, WorkPermitExtensions, WorkPermitSignOffSteps,

WorkPermitSelectedApprovers, WorkPermitStatusTracks,

WorkPermitSiteAreas, WorkPermitPPERequirements, CheckInOutWorkPermits

```



### Status Lifecycle



```

Draft → PendingApproval → Approved → Completed → Closed

↘ Rejected ↗

```



Status transitions managed in `WorkPermitService.UpdateStatus()`. Each transition may:

- Clear/populate approver records

- Clear/populate sign-off step records

- Delete questionnaire answers for certain check types

- Send RabbitMQ messages triggering email notifications



### Sign-Off Steps



`WorkPermitSignOffStep` records are created when moving to certain statuses and must be completed before advancing. They represent multi-level approval or sign-off workflows.



### Selected Approvers



`WorkPermitSelectedApprover` records define who must approve at each stage. Cleared when reverting to Draft.



---



## 14. Key Entities / Database Tables



### Hub Entity: Company



**File:** `ErpContractorPortal.Models\Admin\Company.cs`



```csharp

public class Company : BaseStatusEntity<Guid> {

public string Name { get; set; }

public string Code { get; set; }

public bool HasContractorInfo { get; set; }

public Guid? DefaultRoleId { get; set; }

public Guid? LogoId { get; set; }

public int? DefaultApplicationId { get; set; }

public bool IsTestContractor { get; set; }

// + Address, City, State, CountryId, ZipCode, PhoneNumber

}

```



Nearly every other entity has a `CompanyId` FK to this table.



### User & Employee (Separate Tables)



`User` — authentication/authorization (`ErpContractorPortal.Models\Account\User.cs`):

- `UserCode` (Guid), `UserName`, `Email`, `Password` (hashed), `PasswordSalt`

- `RoleId`, `CompanyId`, `EmployeeId` (FK to Employee)

- `SecurityCode`, `TwoFactorSecutiryCode`, `VerificationCode`

- `IsActive`, `IsVerified`



`Employee` — business entity (`ErpContractorPortal.Models\Admin\Employee.cs`):

- `CompanyId`, `FirstName`, `LastName`, `EmailAddress`, `EmployeeCode`

- `EmployeeTypeId`, `SiteId`, `JobTitleId`, `DepartmentId`

- `SupervisorId` (self-referential)

- `IsSystemUser`, `VisitorTypeId`



### GroupCode (Lookup/Reference Table)



Nearly all reference data (department types, job titles, audit types, NC categories, etc.) stored in a single `GroupCode` table with a `GroupId` discriminator. All FK columns like `DepartmentId`, `AuditTypeId`, `TypeOfWorkId` reference this table.



### Role & Permission



`Role` (`ErpContractorPortal.Models\Account\Role.cs`):

- `CompanyId` — roles are company-scoped

- `Code`, `Name`, `IsAllSiteAccess`

- Navigation: `RolePermissions` (many-to-many with `PermissionModule`), `RoleSites`



### Base Entity Hierarchy



**File:** `ErpContractorPortal.Models\Core\BaseEntity.cs`



```csharp

public class BaseEntity<TKey> { public TKey Id { get; set; } }



public class BaseAuditableEntity<TKey> : BaseEntity<TKey> {

public Guid? CreatedBy { get; set; }

public DateTime? CreatedOn { get; set; }

public Guid? ModifiedBy { get; set; }

public DateTime? ModifiedOn { get; set; }

}



public class BaseStatusEntity<TKey> : BaseAuditableEntity<TKey> {

public bool IsActive { get; set; } = true;

}

```



### Other Key Entities



| Entity | Table | Purpose |

|---|---|---|

| `SiteArea` | SiteAreas | Site/zone management |

| `NonConformance` | NonConformances | NC reporting |

| `CorrectiveAction` | CorrectiveActions | CA tracking |

| `Audits` | Audits | Internal audit management |

| `EmailTemplate` | EmailTemplates | Handlebars HTML templates |

| `EmailData` | EmailData | Email send audit log |

| `ConfigSetting` | ConfigSettings | Key-value app configuration |

| `Attachment` | Attachments | File metadata |

| `ClientContractor` | ClientContractors | Client-Contractor relationship |



---



## 15. Configuration & Environment Management



### appsettings.json Pattern



**File:** `ErpContractorPortal.WebAPI\appsettings.json`



```json

{

"ConnectionStrings": {

"Portal": "<<AES_ENCRYPTED_CONNECTION_STRING>>",

"RedisCache": "152.67.4.152:6379"

},

"AppSettings": {

"SecretKey": "secret",

"Environment": "Development"

},

"Services": {

"Identity": "http://...",

"OpenApi": "http://..."

},

"RabbitMq": {

"MessageQueueUrl": "152.67.4.152",

"MessageQueryPort": 5672,

"MessageQueueUser": "guest",

"MessageQueuePassword": "guest"

},

"SendGrid": {

"ApiKey": "SG.xxx"

}

}

```

**Issues:** `SecretKey = "secret"` and RabbitMQ `guest/guest` credentials suggest dev values used in production.



### Encryption



**File:** `ErpContractorPortal.Encryption\EncryptionManager.cs`



AES-256 with `Rfc2898DeriveBytes`, fixed salt, key `"Mnandoe#orp"`.



Used for:

1. SQL Server connection string in `appsettings.json`

2. SMTP credentials in `appsettings.json`

3. Quartz SQL connection (decrypted inline in `Scheduler\Startup.cs`)



### Runtime Config from Database



`ConfigSetting` table provides runtime-configurable settings:

- `EnableSMTPEmail` — email on/off master switch

- `IsSMTPDevelopmentMode` — redirect all emails to dev address

- `DeveloperModeEmail` — redirect target

- `EnableEmailForCompany` — per-company email override



### Environment Differentiation



Queue names include environment suffix: `Constants.Common.ENVIRONMENT` (e.g., `"Development"`), producing queues like `"RequestWorkPermitApproval_Development"`.



---



## 16. Dependency Injection Setup



### Repository Layer



**File:** `ErpContractorPortal.Repository\RepositoryExtensions.cs`



Convention-based registration: finds all `IXxxRepository → XxxRepository` pairs in the assembly and registers each as Transient.



Explicit: `IProcedureContext → DapperProcedureContext` (Transient)



### Service Layer



**File:** `ErpContractorPortal.Service\ServiceLayerExtension.cs`



Convention-based registration: finds all `IXxxService → XxxService` pairs. Also registers:

```csharp

services.AddTransient<IQuestionaireScoreManager, QuestionaireScoreManager>();

services.AddTransient<UserPreferenceAbstract, UserPreferenceImplementation>();

services.AddSingleton<SendGridDetailOptions>(...);

```



AutoMapper registered as singleton.



### Infrastructure Layer



**File:** `ErpContractorPortal.Infrastructure\InfrastructureExtensions.cs`



```csharp

services.AddStackExchangeRedisCache(options => { options.Configuration = redisConn; });

services.AddSingleton<ILocalizeManager, RedisLocalizeManager>();

services.AddTransient<IDataCache, CommonDataCache>();

services.AddHttpContextAccessor();

services.AddTransient<IIdentityHelper, IdentityHelper>();

services.AddTransient<ILocalizeTextHelper, LocalizeTextHelper>();

services.AddTransient<IEncryptionManager, EncryptionManager>();

services.AddTransient<QueueNameFactory>();

```



### MVC Portal DI



**File:** `ErpContractorPortal\Startup.cs`



```csharp

services.AddHttpClient<IContractorPortalClient, ContractorPortalClient>(client => {

client.BaseAddress = new Uri(Configuration["Services:OpenApi"]);

})

.AddUserAccessTokenHandler() // IdentityModel.AspNetCore automatic token attachment

.AddPolicyHandler(GetRetryPolicy()); // Polly retry



services.AddAccessTokenManagement();

```



### DI Lifetime Summary

- **Transient**: all repositories, services, helpers, IDataCache

- **Singleton**: AutoMapper, ILocalizeManager, SendGridDetailOptions

- **Scoped**: DbContext (EF Core standard via `AddDbContext`)

- **Named/Typed HTTP clients**: IContractorPortalClient, ISchedulerClient



---



## 17. Error Handling



### Global Exception Handler



Pattern used in all API projects (`WebAPI\Startup.cs`, `Scheduler\Startup.cs`, `Security\Startup.cs`):



```csharp

app.UseExceptionHandler(errorApp => {

errorApp.Run(async context => {

context.Response.StatusCode = 500;

var feature = context.Features.Get<IExceptionHandlerFeature>();

if (feature != null) {

var logPath = Path.Combine(env.ContentRootPath, "Logs", "Error.txt");

Directory.CreateDirectory(Path.GetDirectoryName(logPath));

using var writer = new StreamWriter(logPath, append: true);

writer.WriteLine(feature.Error.Message);

writer.WriteLine(feature.Error.StackTrace);

}

});

});

```



**Issues:**

- Logs to flat text file (`Logs/Error.txt`) — no structured logging

- No Serilog, NLog, or Application Insights integration

- No `ILogger<T>` usage found in reviewed files

- Error response body is empty (500 returned with no JSON body)

- File append is not thread-safe under concurrent exceptions



### Model Validation (FluentValidation)



**File:** `ErpContractorPortal.WebAPI\Startup.cs`



```csharp

services.AddControllers().ConfigureApiBehaviorOptions(options => {

options.InvalidModelStateResponseFactory = context => {

var errors = context.ModelState

.Select(e => new ModelErrorViewModel {

Name = e.Key,

Errors = e.Value.Errors.Select(s => s.ErrorMessage).ToList()

});

return new BadRequestObjectResult(errors);

};

});

```



Returns 400 with `List<ModelErrorViewModel>` on validation failure.



### Dapper Exception Swallowing



```csharp

// DapperProcedureContext.InsertOrUpdate:

try { /* execute SP */ return true; }

catch { return false; } // All SP errors silently swallowed

```



---



## 18. Frontend Architecture



### MVC Portals (Razor Views)



All three portals (Contractor, Client, CheckIn) use:

- ASP.NET Core MVC with Razor views

- Razor Runtime Compilation enabled (changes without recompile in dev)

- Session timeout: 30 minutes

- Cookie expiry: 3 hours with sliding expiration



### Portal → API Communication



**File:** `ErpContractorPortal\Core\ErpContractorPortalRestClient.cs`



```csharp

public class ContractorPortalClient {

public async Task<T> GetAsync<T>(string endpoint);

public async Task<T> PostAsync<T>(string endpoint, object body);

// PostAsync parses 400 body into ModelStateDictionary

public async Task<T> PostFileAsync<T>(string endpoint, MultipartFormDataContent content);

public async Task<T> DownloadFileAsync<T>(string endpoint);

}

```



Tokens automatically forwarded via `AddUserAccessTokenHandler()`. Polly retry handles transient failures.



### PowerBI Integration



`ErpClientPortal` references `ErpContractorPortal.PowerBiIntegration` for embedded analytics dashboards in the client-facing portal.



### SCIM Support



`ErpContractorPortal.SCIM` implements SCIM 2.0 (System for Cross-domain Identity Management) via `Microsoft.SystemForCrossDomainIdentityManagement` — supports user provisioning from Azure AD / identity providers.



### Localization



`ILocalizeManager` (RedisLocalizeManager, singleton) caches UI text in Redis, loaded from DB. `ILocalizeTextHelper` is request-scoped and reads user's `LanguageId` from JWT claims.



---



## 19. API Communication Patterns



### MVC → WebAPI



All MVC portals talk to `ErpContractorPortal.WebAPI` via typed HttpClient wrappers. Token management via IdentityModel.AspNetCore `AddUserAccessTokenHandler()` — automatic token refresh and Bearer attachment.



### JWT Validation Strategy



Tokens are validated locally via symmetric key — no token introspection or userinfo endpoint calls at runtime. Because `ValidateLifetime = false`, no token refresh is strictly required by the API.



### Scheduler → WebAPI



`SchedulerClient` (typed HttpClient) is used by the Scheduler project to call WebAPI when jobs trigger actions requiring API logic.



### Integration API



Separate `ErpIntegrationAPI` project for external system integrations. Uses `ClientCredentials` OAuth flow plus `ValidatedClient` header guard (hardcoded value `4c703c0e95ce4a87be3d2e558447da72`).



### Request/Response Models (Conventions)



- Requests: `*AddEditViewModel` for create/update, `*FilterViewModel` for query parameters

- Responses: `ResultViewModel<T>` for single results, `List<T>` for collections, paginated wrappers for lists

- Errors: `List<ModelErrorViewModel>` on 400 responses



### Swagger



Every API project exposes Swagger UI via Swashbuckle 5.5.1, with Bearer token authentication definition configured.



### Token Refresh (MVC Portals)



IdentityModel.AspNetCore manages token refresh automatically via `AddAccessTokenManagement()`. Polly retry is configured on all outbound HTTP clients.



---



## 20. Code Generation / Scaffolding Patterns



### Convention-Based DI Registration



The system avoids manual DI registration using reflection conventions:



```csharp

// Repository registration (RepositoryExtensions.cs):

var assembly = Assembly.GetExecutingAssembly();

foreach (var serviceType in assembly.GetTypes().Where(IsRepositoryInterface)) {

var implType = assembly.GetTypes()

.First(t => t.Name == serviceType.Name.TrimStart('I'));

services.AddTransient(serviceType, implType);

}

// Same pattern in ServiceLayerExtension.cs for services

```



Enforced naming convention: `IWorkPermitService → WorkPermitService`.



### EF Configuration Convention



DbContext discovers all entity maps automatically:

```csharp

// EfDataContext.OnModelCreating:

var configs = Assembly.GetExecutingAssembly().GetTypes()

.Where(t => t.Name.StartsWith("Ef") && t.Name.EndsWith("Map"));

foreach (var config in configs)

modelBuilder.ApplyConfiguration((dynamic)Activator.CreateInstance(config));

```



Enforced naming: `EfWorkPermitMap`, `EfEmployeeMap`, etc.



### Message Handler Convention



`BackgroudTaskProcessor.cs` maps queue names to handler methods in a `Dictionary<string, Action<string>>`. Adding a new message type requires:

1. Add queue name constant to `Constants.cs`

2. Add handler method to `BackgroudTaskProcessor`

3. Register in `_messageStrategies` dictionary



No code generation tooling (no T4, no Roslyn source generators, no EF scaffolding CLI, no Swagger codegen) was found. All code is handwritten following the naming conventions above.



---



## Architecture Issues & Improvement Notes



### Security Issues

1. **JWT validation disabled**: `ValidateLifetime = false`, `ValidateIssuer = false` — expired tokens accepted indefinitely

2. **Weak signing key**: `AppSettings:SecretKey = "secret"` stored in appsettings.json

3. **Integration guard**: `ValidatedClient` header uses hardcoded magic string — not a proper auth mechanism

4. **Hardcoded encryption key**: `"Mnandoe#orp"` in source code — should be in Azure Key Vault or environment variables

5. **`AddDeveloperSigningCredential()`** used in Security project — not suitable for production



### Reliability Issues

6. **RabbitMQ**: `durable: false` (queues and messages lost on restart), `autoAck: true` (messages lost on handler failure), new connection created per message send

7. **BinaryFormatter**: Obsolete in .NET 5, removed in .NET 6 — blocks upgrade path

8. **Files on disk**: Attachments in wwwroot lost on redeployment or server migration

9. **Exception swallowing**: Dapper `InsertOrUpdate` silently swallows all stored procedure errors



### Observability Issues

10. **No structured logging**: Errors written to `Logs/Error.txt` flat file

11. **No distributed tracing**: No correlation IDs, no Application Insights, no OpenTelemetry

12. **No health checks**: No `/health` or `/ready` endpoints



### Architecture Issues

13. **Manual tenant filtering**: No global EF query filter — one missed `WHERE CompanyId = ?` clause leaks data across tenants

14. **Fat service constructors**: `WorkPermitService` injects 14 dependencies — violates Single Responsibility Principle

15. **No CQRS**: Read and write models mixed in same service; complex queries should use dedicated read models

16. **Hardcoded client store**: `CustomClientStore.cs` hardcodes all OAuth clients — adding a client requires code change and redeployment



### Upgrade Blockers

17. **Target .NET Core 3.1** (EOL November 2022): All projects must be upgraded

18. **BinaryFormatter usage** blocks migration to .NET 5+

19. **IdentityServer4 EOL** (November 2022): Should migrate to Duende IdentityServer or OpenIddict
