# تقرير طوبولوجيا مشروع CivicSync مع الأدوار والصلاحيات

**التصنيف:** تقرير معماري (Architecture / Topology Report)  
**المشروع:** CivicSync — نظام تنسيق بيانات المواطنين والخدمات الحكومية  
**النطاق:** الخريطة الطوبولوجية الكاملة + نموذج التحكم بالوصول (RBAC)  
**الإصدار:** 1.0  
**التاريخ:** 21 أكتوبر 2026  
**المُعدّ:** مهندس معمارية برمجيات (Software Architect)

---

## 1. ملخص تنفيذي (Executive Summary)

| البند | القيمة |
|-------|--------|
| نمط المعمارية | Clean Architecture / Layered Architecture (6 مشاريع) |
| حزمة تقنية الواجهة الخلفية | ASP.NET Core Web API على .NET 10 (`net10.0`) |
| حزمة تقنية الواجهة الأمامية | Blazor Server (Render Mode: InteractiveServer) على .NET 10 |
| قاعدة البيانات | Microsoft SQL Server — قاعدة `CivicSyncDB` (EF Core 10.0.12) |
| نموذج الهوية | ASP.NET Core Identity + JWT Bearer (HS256) |
| نموذج الصلاحيات | ثلاثي المستويات: مستخدم ← دور ← صلاحية (Role-Based + Permission-Based) |
| عدد الأدوار (Roles) | 3 (Admin, Auditor, FinanceDirectorate) |
| عدد رموز الصلاحيات (Permissions) | 9 رموز |
| نقاط نهاية API (Controllers) | 1 متحكم / 6 مسارات |
| صفحات الواجهة الأمامية | 14 صفحة Razor |
| عدد الجداول في القاعدة | 14 جدولاً |

---

## 2. الطوبولوجيا العامة للنظام (System Topology)

### 2.1 مخطط السياق — حدود النظام (System Context Diagram)

يُظهر هذا المخطط حدود النظام (نظام CivicSync) والجهات الخارجية التي يتفاعل معها.

```mermaid
flowchart TB
    subgraph Outside["جهات خارجية خارج حدود النظام"]
        User["المستخدم النهائي<br/>مدير النظام / المدقق / حساب المديرية"]
        Ministry["وزارة الداخلية<br/>(خدمة KYC للتحقق من الهوية)"]
        MailSrv["خادم البريد SMTP<br/>منفذ 587"]
    end

    subgraph CS["نظام CivicSync — حدود النظام"]
        Web["CivicSync.Web<br/>Blazor Server<br/>الواجهة الأمامية"]
        Api["CivicSync.Api<br/>ASP.NET Core Web API"]
        DB[("SQL Server<br/>CivicSyncDB")]
    end

    User -->|"HTTPS 7284<br/>واجهة مستخدم تفاعلية"| Web
    User -->|"HTTPS 7171<br/>استدعاء REST مباشر"| Api
    Web -->|"HTTPS 7171<br/>REST + JSON + Bearer Token"| Api
    Api -->|"TCP 1433<br/>TDS / Entity Framework Core"| DB
    Api -->|"HTTPS 443<br/>REST + Token + Cache 115 دقيقة"| Ministry
    Api -.->|"SMTP 587<br/>غير مُفعّل حالياً"| MailSrv

    style CS fill:#e8f0fe,stroke:#4285f4,stroke-width:2px
    style Outside fill:#fef7e0,stroke:#f9ab00,stroke-width:1px
```

### 2.2 مخطط الطبقات المعماري واتجاه الاعتمادية (Layered Architecture & Dependency Rule)

يُظهر هذا المخطط قواعد الاعتمادية: **الطبقة الأبعد (Core) لا تعرف شيئاً عن الطبقات الأعلى**.

```mermaid
flowchart TD
    subgraph Presentation["طبقة العرض (Presentation)"]
        Web["CivicSync.Web<br/>Blazor Server · المنفذ 7284/5297"]
        Api["CivicSync.Api<br/>REST API · المنفذ 7171/5234"]
    end

    subgraph Application["طبقة التطبيق (Application) — منطق الأعمال"]
        AppSvc["CivicSync.Application<br/>UserService · CitizenService"]
    end

    subgraph Infrastructure["طبقة البنية التحتية (Infrastructure) — التقنية"]
        Repo["المستودعات<br/>Repository · UnitOfWork · Specification"]
        DbCtx["ApplicationDbContext<br/>EF Core 10.0.12 + SQL Server"]
        Providers["مزودو الأمان<br/>JwtService · KycService · CurrentUserService"]
        Config["الإعدادات<br/>AppSettings · JwtSettings"]
    end

    subgraph Domain["طبقة النطاق (Domain) — قلب النظام"]
        Entities["الكيانات<br/>Citizen · Directorate · AuditLog<br/>ApplicationUser · ApplicationRole"]
        Interfaces["العقود (Interfaces)<br/>IRepository · IUnitOfWork · IUserService<br/>IJwtService · IKycService"]
    end

    subgraph Shared["طبقة العقود المشتركة (Shared)"]
        Contracts["Contracts<br/>Requests · Responses · Constants · Extensions"]
    end

    Api --> AppSvc
    Api --> DbCtx
    Web --> Contracts
    AppSvc --> Entities
    AppSvc --> Interfaces
    AppSvc --> DbCtx
    AppSvc --> Providers
    AppSvc --> Contracts
    Repo --> DbCtx
    Providers --> Config
    DbCtx --> Entities
    Entities --> Contracts

    style Domain fill:#e6f4ea,stroke:#34a853,stroke-width:2px
    style Shared fill:#f3e8fd,stroke:#a142f4,stroke-width:2px
    style Infrastructure fill:#fef7e0,stroke:#f9ab00,stroke-width:1px
    style Application fill:#e8f0fe,stroke:#4285f4,stroke-width:1px
    style Presentation fill:#fce8e6,stroke:#ea4335,stroke-width:1px
```

**قاعدة الاعتمادية (Dependency Rule):**

| المشروع | يستطيع الاعتماد على | لا يعتمد على |
|--------|---------------------|--------------|
| `CivicSync.Shared` | — (لا شيء) | الجميع |
| `CivicSync.Core` | Shared | Infrastructure، Application، Api، Web |
| `CivicSync.Infrastructure` | Core، Shared | Application، Api، Web |
| `CivicSync.Application` | Core، Infrastructure، Shared | Api، Web |
| `CivicSync.Api` | Application، Infrastructure، Core، Shared | — |
| `CivicSync.Web` | Shared | Core، Infrastructure، Application، Api |

> **ملاحظة معمارية:** `CivicSync.Web` لا تقتبس `CivicSync.Application` أو `Infrastructure` — بل تتواصل مع الـ API عبر contracted HTTP فقط، مما يحقق عزلاً كاملاً (Decoupling) للواجهة الأمامية عن منطق الأعمال. هذه نقطة قوة في التصميم وليست ضعفاً.

### 2.3 مخطط تدفق البيانات (Data Flow Diagram — Level 1)

```mermaid
flowchart LR
    subgraph Tier1["1 — الاستعلام"]
        A["المواطن / المدقق<br/>يطلب بيانات مواطن"]
    end
    subgraph Tier2["2 — الواجهة الأمامية"]
        B["CivicSync.Web<br/>Blazor Server<br/>Render + State"]
        B2["CivicSyncMockStore<br/>Singleton (بيانات تجريبية)"]
    end
    subgraph Tier3["3 — منطق الأعمال"]
        C["CivicSync.Api<br/>Controllers + Filters"]
        D["CivicSync.Application<br/>UserService / CitizenService"]
    end
    subgraph Tier4["4 — البيانات والتكامل"]
        E["EF Core + UnitOfWork<br/>SQL Server"]
        F["KycService<br/>وزارة الداخلية"]
        G["AuditLog / LookupRequestLog<br/>سجلات التدقيق"]
    end

    A -->|"1. طلب"| B
    B --> B2
    B -->|"2. HTTPS REST<br/>api/citizens/by-national-no/{no}"| C
    C -->|"3. تفويض (JWT Role)"| D
    D -->|"4. بحث محلي"| E
    D -->|"5. عند عدم وجود السجل"| F
    F -->|"6. رد JSON"| D
    D -->|"7. تخزين + سجل"| E
    D -->|"8. تدقيق"| G
    E -->|"9. بيانات"| D
    D -->|"10. JSON"| C
    C -->|"11. استجابة"| B
    B -->|"12. عرض"| A

    style Tier1 fill:#fce8e6,stroke:#ea4335
    style Tier2 fill:#e8f0fe,stroke:#4285f4
    style Tier3 fill:#e6f4ea,stroke:#34a853
    style Tier4 fill:#fef7e0,stroke:#f9ab00
```

### 2.4 مخطط تسلسل عملية البحث عن مواطن (Sequence Diagram)

يوضح تسلسل الرسائل بين المتصفح والواجهة الأمامية والـ API وقاعدة البيانات وخدمة الداخلية.

```mermaid
sequenceDiagram
    autonumber
    participant BR as المتصفح
    participant W as CivicSync.Web
    participant API as CivicSync.Api
    participant JWT as JwtService
    participant APP as CitizenService
    participant DB as SQL Server
    participant KYC as خدمة KYC

    BR->>W: 1. GET /citizens (Blazor Server SignalR/WebSocket)
    W->>W: 2. عرض الواجهة (Razor + ApexCharts)
    BR->>W: 3. إدخال رقم الوطني + طلب استعلام
    W->>API: 4. GET api/citizens/by-national-no/{no}
    API->>API: 5. التحقق من JWT (Issuer/Audience/Signature/Lifetime)
    alt 5a. الرمز صالح
        API->>APP: 6. استدعاء الخدمة
    else 5b. الرمز غائب أو منتهي
        API-->>W: 6. 401 Unauthorized
    end
    APP->>DB: 7. SELECT من جدول Citizens
    alt 7a. السجل موجود (Cache)
        APP-->>API: 8. البيانات + Source=Cache
    else 7b. السجل غير موجود
        APP->>KYC: 9. GET /kyc/citizen/national-no/{no}
        KYC-->>APP: 10. JSON (بيانات المواطن)
        APP->>APP: 11. تطبيع الأسماء (NormalizeText)
        APP->>DB: 12. INSERT Citizens + INSERT LookupRequestLogs
        APP-->>API: 13. البيانات + Source=Interior
    end
    API-->>W: 14. JSON (GeneralResponse)
    W-->>BR: 15. العرض النهائي (Blazor Diff)

    Note over KYC: في البيئة الحالية تقرأ الخدمة من ملف db.json المحلي (وضع تجريبي)
```

### 2.5 مخطط خط أنابيب الطلب في الـ API (Middleware Pipeline)

```mermaid
flowchart TD
    Req["HTTP Request"] --> M1{"UseDefaultFiles"}
    M1 --> M2["UseStaticFiles<br/>wwwroot + index.html"]
    M2 --> M3["UseCors - AllowAll<br/>AllowAnyHeader/Method + Credentials"]
    M3 --> M4["UseAuthentication<br/>JWT Bearer"]
    M4 --> M5["UseAuthorization<br/>Role-based checks"]
    M5 --> M6{"MapControllers"}
    M6 --> M7["UserController<br/>/api/user/*"]
    M6 --> M8["Static wwwroot<br/>index.html + logo.svg"]
    M7 --> M9["UseSwagger / UseSwaggerUI<br/>/swagger"]
    M8 --> M9
    M9 --> Res["HTTP Response"]

    style M3 fill:#fce8e6,stroke:#ea4335
    style M4 fill:#e6f4ea,stroke:#34a853
    style M5 fill:#e6f4ea,stroke:#34a853
    style M9 fill:#e8f0fe,stroke:#4285f4
```

**ترتيب التهيئة عند الإقلاع (Startup Sequence) في `CivicSync.Api\Program.cs`:**

| الخطوة | الاستدعاء | الأثر |
|--------|-----------|-------|
| 1 | `AddApplicationServices()` | تسجيل `IUserService` و `ICitizenService` (Scoped) |
| 2 | `AddInfrastructureServices(Configuration)` | تسجيل DbContext + Identity + JWT + Cache + SignalR + KycService |
| 3 | `AddApiServices(Configuration)` | تسجيل Controllers + Swagger + CORS + ProblemDetails |
| 4 | `AddControllers().AddJsonOptions(...)` | ضبط `ReferenceHandler.IgnoreCycles` لتفادي الأخطاء المرجعية في JSON |
| 5 | `db.Database.Migrate()` | تطبيق الهجرات تلقائياً على قاعدة البيانات عند الإقلاع |
| 6 | `ApplicationDbSeeder.SeedUsersAsync(userManager)` | بذر حساب المدير الافتراضي |

### 2.6 شجرة مكوّنات الواجهة الأمامية (Blazor Component Tree)

```mermaid
flowchart TD
    App["App.razor<br/>HTML Root · dir=rtl · lang=ar"] --> Routes["Routes.razor<br/>Router + FocusOnNavigate"]

    Routes --> L1["MainLayout.razor<br/>التخطيط الرئيسي"]
    Routes --> L2["EmptyLayout.razor<br/>تخطيط الصفحات العامة"]
    Routes --> L3["NotFound.razor"]

    L1 --> Nav["AppNavbar.razor<br/>شريط التنقل + المستخدم + حالة الداخلية"]
    L1 --> RM["ReconnectModal.razor<br/>نافذة إعادة الاتصال"]

    Nav --> IC["AppIcon.razor<br/>مكتبة الأيقونات"]
    Nav --> SP["StatusPill.razor"]
    Nav --> SC["StatusCard.razor"]
    Nav --> PM["PageHeader.razor"]
    Nav --> ST["StatTile.razor"]
    Nav --> MC["MetricCard.razor"]
    Nav --> GB["GlassCard.razor"]
    Nav --> IB["InfoBlock.razor"]
    Nav --> ES["EmptyState.razor"]
    Nav --> AD["AccessDenied.razor<br/>صفحة منع الوصول"]
    Nav --> RC["RankedBars.razor"]
    Nav --> BC["BarChart.razor"]
    Nav --> DC["DonutChart.razor"]
    Nav --> AC["AreaChart.razor"]
    Nav --> SG["SegmentedControl.razor"]

    Pages["صفحات التطبيق (14)"] --> P1["Home.razor — /"]
    Pages --> P2["Citizens.razor — /citizens"]
    Pages --> P3["CitizenDetails.razor — /citizens/{Id:int}"]
    Pages --> P4["Users.razor — /users"]
    Pages --> P5["Directorates.razor — /directorates"]
    Pages --> P6["Interior.razor — /interior"]
    Pages --> P7["AuditLogs.razor — /logs/audit"]
    Pages --> P8["RequestLogs.razor — /logs/requests"]
    Pages --> P9["Lookup.razor — /lookup"]
    Pages --> P10["Account.razor — /account"]
    Pages --> P11["Login.razor — /login"]
    Pages --> P12["Error.razor — /Error"]
    Pages --> P13["NotFound.razor — /not-found"]
    Pages --> P14["BorderDemo.razor — /dev/borders"]

    style AD fill:#fce8e6,stroke:#ea4335,stroke-width:2px
    style P4 fill:#fce8e6,stroke:#ea4335
    style P5 fill:#fce8e6,stroke:#ea4335
    style P6 fill:#fce8e6,stroke:#ea4335
    style P9 fill:#fce8e6,stroke:#ea4335
```

### 2.7 مخطط قاعدة البيانات — العلاقات (ER Diagram)

```mermaid
erDiagram
    ApplicationUsers ||--o{ AspNetUserRoles : "يملك"
    ApplicationRoles ||--o{ AspNetUserRoles : "يُسند"
    ApplicationRoles ||--o{ ApplicationRolePermissions : "يملك"
    ApplicationPermissions ||--o{ ApplicationRolePermissions : "يُمنح"
    ApplicationUsers ||--o{ AspNetUserClaims : ""
    ApplicationUsers ||--o{ AspNetUserLogins : ""
    ApplicationUsers ||--o{ AspNetUserTokens : ""
    ApplicationRoles ||--o{ AspNetRoleClaims : ""
    Directorates ||--o{ ApplicationUsers : "يتبع"
    Directorates ||--o{ LookupRequestLogs : "ينتج"
    ApplicationUsers ||--o{ LookupRequestLogs : "ينفّذ"
    ApplicationUsers ||--o{ AuditLogs : "ينتج"
    Citizens ||--o{ CitizenHistory : "تاريخ"
    Citizens ||--o{ LookupRequestLogs : "يرتبط"

    ApplicationUsers {
        Guid Id PK
        string UserName
        string Email
        string FullName
        bool IsActive
        bool MustChangePassword
        Guid DirectorateId FK "nullable"
        int TokenLifetimeMinutes "nullable"
        datetime LastLoginAt
        datetime CreatedAt
    }
    ApplicationRoles {
        Guid Id PK
        string Name "NormalizedName فريد"
        string Description
    }
    ApplicationPermissions {
        int Id PK
        string Code "فريد — IX_ApplicationPermissions_Code"
        string Description
    }
    ApplicationRolePermissions {
        Guid RoleId PK_FK
        int PermissionId PK_FK
    }
    Directorates {
        Guid Id PK
        string Code "فريد"
        string Name
        int DailyRequestLimit "nullable"
        bool IsActive
    }
    Citizens {
        Guid Id PK
        string NationalNo "فهرس IX_Citizens_NationalNo"
        string FirstName
        string LastName
        string FatherName
        string MotherName
        date BirthDate
        string NormFirstName
        string NormLastName
        string NormFatherName
        string NormMotherName
        string RawPayload
        datetime FetchedAt
        datetime LastVerifiedAt
        byte RecordStatus
        byteArray RowVersion "concurrency token"
    }
    CitizenHistory {
        Guid Id PK
        Guid CitizenId FK
        string Snapshot "JSON"
        datetime ChangedAt
    }
    LookupRequestLogs {
        Guid Id PK
        Guid UserId FK
        Guid DirectorateId FK
        Guid CitizenId FK "nullable"
        int SearchType
        string NationalNo
        int Source "RequestSource"
        string ErrorMessage
        int DurationMs
        string IpAddress
        datetime CreatedAt
    }
    AuditLogs {
        Guid Id PK
        Guid UserId FK
        string Action
        string EntityType
        string EntityId
        string Details "JSON"
        string IpAddress
        datetime CreatedAt
    }
```

**جداول قاعدة البيانات الـ 14:**

| # | اسم الجدول | النوع | الوصف |
|---|-----------|-------|-------|
| 1 | `AspNetUsers` | هوية | حسابات المستخدمين (يرث `ApplicationUser`) |
| 2 | `AspNetRoles` | هوية | الأدوار (يرث `ApplicationRole`) |
| 3 | `AspNetUserRoles` | هوية | ربط المستخدمين بالأدوار (علاقة متعدد-متعدد) |
| 4 | `AspNetUserClaims` | هوية | مطالبات المستخدم |
| 5 | `AspNetUserLogins` | هوية | بيانات الدخول الخارجية |
| 6 | `AspNetUserTokens` | هوية | رموز إعادة التعيين وتأكيد البريد |
| 7 | `AspNetRoleClaims` | هوية | مطالبات الأدوار |
| 8 | `ApplicationPermissions` | صلاحيات | رموز الصلاحيات (9 رموز) |
| 9 | `ApplicationRolePermissions` | صلاحيات | ربط الأدوار بالصلاحيات (مفتاح مركّب) |
| 10 | `Directorates` | أعمال | المديريات المالية والفروع |
| 11 | `Citizens` | أعمال | سجلات المواطنين (مع التطبيع والفهرسة) |
| 12 | `CitizenHistory` | أعمال | لقطات تاريخية JSON لكل تعديل |
| 13 | `LookupRequestLogs` | تدقيق | سجل كل طلبات الاستعلام (مصدر، مدة، خطأ) |
| 14 | `AuditLogs` | تدقيق | سجل إجراءات المستخدمين |

---

## 3. جرد المكونات التفصيلي (Component Inventory)

### 3.1 الواجهة الأمامية — `CivicSync.Web`

| المكوّن | النوع | الموقع | الدور |
|---------|-------|--------|-------|
| `Program.cs` | نقطة الدخول | `Web\Program.cs` | تسجيل Razor Components + Interactive Server + Typed HttpClient + HSTS + Anti-forgery |
| `App.razor` | جذر HTML | `Web\Components\App.razor` | تحميل `app.css`، `blazor.web.js`، `civicsync.js`، ApexCharts، وخط IBM Plex Sans Arabic |
| `Routes.razor` | الموجّه | `Web\Components\Routes.razor` | توجيه الصفحات + `FocusOnNavigate` + صفحة 404 |
| `MainLayout.razor` | تخطيط | `Web\Components\Layout\` | الهيكل العام + شريط التنقل + نافذة إعادة الاتصال |
| `EmptyLayout.razor` | تخطيط | `Web\Components\Layout\` | تخطيط صفحات الدخول (بدون تنقل) |
| `ReconnectModal.razor` | مكوّن تفاعلي | `Web\Components\Layout\` | التقاط انقطاع SignalR وإعادة الاتصال |
| `AppNavbar.razor` | تنقل | `Web\Components\Shared\` | روابط مشروطة بالدور + قائمة المستخدم + حالة الاتصال + تبديل المظهر |
| `AccessDenied.razor` | حراسة | `Web\Components\Shared\` | صفحة منع الوصول (403) للمستخدم غير المخوّل |
| `CivicSyncMockStore` | خدمة Singleton | `Web\Services\CivicSyncMockStore.cs` | مخزن بيانات تجريبي في الذاكرة + سجل تدقيق + إدارة المستخدمين والمديريات |
| `CitizenApiClient` | Typed HttpClient | `Web\Services\Citizens\` | استدعاء `api/citizens/*` عبر `ApiBaseUrl` + رمي `CitizenApiException` |
| `ICitizenApiClient` | عقد | `Web\Services\Citizens\` | واجهة: `GetByNationalNoAsync`, `SearchByPentadAsync`, `SearchAsync` |
| `CivicModels.cs` | نماذج العرض | `Web\Models\CivicModels.cs` | `UserRole`, `InteriorState`, `RequestSource`, `BoardUser`, `Directorate`, `Citizen`, `RequestLog`, `AuditLog` |
| `app.css` | تنسيق | `Web\wwwroot\` | ناتج بناء Tailwind + daisyUI |
| `civicsync.js` | JS | `Web\wwwroot\` | تبديل المظهر (Light/Dark) + الحفظ في localStorage |
| `charts.js` / `chartFx.js` | JS | `Web\wwwroot\js\` | تهيئة الرسوم البيانية ApexCharts + تأثيرات |
| `borderFx.js` | JS | `Web\wwwroot\js\` | تأثيرات حدود التنقل + تركيز لوحة المفاتيح |
| `apexcharts.min.js` | مكتبة | `Web\wwwroot\lib\apexcharts\` | مكتبة الرسوم البيانية (تُنسخ من `node_modules` عند البناء) |

### 3.2 واجهة برمجة التطبيقات — `CivicSync.Api`

| المكوّن | النوع | الدور |
|---------|-------|-------|
| `Program.cs` | نقطة الدخول | تسجيل الخدمات، تطبيق الهجرات، البذر، بناء خط الأنابيب |
| `ServiceCollection.cs` | امتداد DI | CORS + Swagger + ProblemDetails + LowercaseUrls + تعريف Bearer في Swagger UI |
| `UserController.cs` | متحكم | 6 مسارات: `login`, `ResetPassword`, `GetUserById`, `GetAllUsers`, `Register`, `DeleteUser` |
| `EnumSchemaFilter.cs` | مرشّح Swagger | عرض قيم الـ Enums في وثائق OpenAPI |
| `db.json` | بيانات تجريبية | مصدر بديل لبيانات KYC في وضع التطوير |
| `wwwroot/index.html` | صفحة ثابتة | صفحة ترحيب بديلة عند تشغيل الـ API مباشرة |

### 3.3 طبقة التطبيق — `CivicSync.Application`

| الخدمة | الواجهة | الدوال |
|--------|---------|--------|
| `UserService` | `IUserService` | `LoginAsync`, `RegisterAsync`, `GetAllUsersAsync`, `GetUserByIdAsync`, `ResetPasswordAsync`, `DeleteUserAsync` |
| `CitizenService` | `ICitizenService` | `GetOrFetchByNationalNoAsync`, `GetOrFetchByPentadAsync`, `SearchCitizensAsync` + دالة `NormalizeText` |

### 3.4 طبقة البنية التحتية — `CivicSync.Infrastructure`

| المكوّن | النمط | الدور |
|---------|-------|-------|
| `ApplicationDbContext` | DbContext | `IdentityDbContext<ApplicationUser, ApplicationRole, Guid>` + 9 `DbSet` + `ApplyConfigurationsFromAssembly` |
| `Repository<T>` | Repository | `GetByIdAsync`, `GetAllAsync(spec)`, `AddAsync`, `UpdateAsync`, `DeleteAsync`, `SaveAsync` |
| `UnitOfWork` | Unit of Work | `Repository<T>()` + `CompleteAsync()` (معاملة ذرية) |
| `Specification<T>` / `Spec` | Specification | معايير استعلام قابلة لإعادة الاستخدام: Criteria, Includes, OrderBy, Paging |
| `JwtService` | Provider | توليد رمز JWT بتوقيع HMAC-SHA256 مع تضمين الأدوار كـ Claims |
| `KycService` | Provider | جلب بيانات المواطن من وزارة الداخلية + Cache للتوكن (115 دقيقة) + `SemaphoreSlim` لمنع التزاحم |
| `CurrentUserService` | Provider | استخراج `UserIdentifier` و `IsActive` من `HttpContext.User` |
| `AppSettings` / `JwtSettings` | Configuration | تحميل `appsettings.json` حسب `ASPNETCORE_ENVIRONMENT` (Singleton كسول) |
| `ApplicationDbSeeder` | Seeder | بذر حساب المدير الافتراضي عند أول تشغيل |
| `BackgroundJobService` | Background Job | **غير مُنفَّذ** — `RunUserCleanupAsync()` يرمي `NotImplementedException` |

### 3.5 طبقة النطاق — `CivicSync.Core`

| الفئة / العقد | النوع | ملاحظات |
|---------------|-------|---------|
| `ApplicationUser : IdentityUser<Guid>` | كيان | `FullName` (≤100)، `IsActive`، `MustChangePassword`، `DirectorateId`، `TokenLifetimeMinutes`، `LastLoginAt` |
| `ApplicationRole : IdentityRole<Guid>` | كيان | `Description` + `RolePermissions` |
| `ApplicationPermission : IBaseEntity<int>` | كيان | `Code` (فهرس فريد)، `Description` |
| `ApplicationRolePermission` | كيان ربط | مفتاح مركّب `(RoleId, PermissionId)` |
| `Citizen : IBaseEntity<Guid>, IAuditableEntity` | كيان | حقول مُطبَّعة (Norm*) + `RawPayload` + `RowVersion` (توافق متزامن) |
| `CitizenHistory` | كيان | `Snapshot` بصيغة JSON + `ChangedAt` |
| `Directorate : IBaseEntity<Guid>, IAuditableEntity` | كيان | `Code` فريد، `DailyRequestLimit`، `IsActive` |
| `LookupRequestLog` | كيان | `SearchType`, `Source`, `ErrorMessage`, `DurationMs`, `IpAddress` |
| `AuditLog` | كيان | `Action`, `EntityType`, `EntityId`, `Details` (JSON), `IpAddress` |
| `IBaseEntity<TId>` / `IAuditableEntity` | عقود | واجهات marcador (Marker Interfaces) |
| `IRepository<T>` / `IUnitOfWork` / `ISpecification<T>` | عقود مستودع | أنماط المستودع والـ UoW |
| `IUserService` / `ICitizenService` | عقود تطبيق | عقود طبقة الأعمال |
| `IJwtService` / `ICurrentUserService` / `IKycService` | عقود مزودي | عقود الأمان والتكامل الخارجي |
| `IBackgroundJobService` | عقد | عقد المهام الخلفية (غير مُفعّل) |

---

## 4. جدول المنافذ والبروتوكولات (Ports & Protocols Matrix)

| # | المصدر | الوجهة | البروتوكول | المنفذ | الاتجاه | الاتجاه/الغرض | التكوين |
|---|--------|--------|------------|--------|----------|--------------|---------|
| 1 | المتصفح | CivicSync.Web | HTTPS / TLS | **7284** | ثنائي الاتجاه | عرض الواجهة التفاعلية (Blazor Server) | `launchSettings.json` (Web/https) |
| 2 | المتصفح | CivicSync.Web | HTTP | 5297 | ثنائي الاتجاه | عرض الواجهة (تطوير فقط) | `launchSettings.json` (Web/http) |
| 3 | CivicSync.Web | CivicSync.Api | HTTPS / REST+JSON | **7171** | ثنائي الاتجاه | استدعاء `api/citizens/*` و `api/user/*` | `appsettings.json` → `ApiBaseUrl` |
| 4 | CivicSync.Web | CivicSync.Api | HTTP / REST+JSON | 5234 | ثنائي الاتجاه | بديل HTTP (تطوير) | `launchSettings.json` (Api/http) |
| 5 | CivicSync.Api | SQL Server | TCP / TDS | **1433** | ثنائي الاتجاه | EF Core 10.0.12 (ADO.NET) | `ConnectionStrings:DefaultConnection` |
| 6 | CivicSync.Api | خدمة KYC الخارجية | HTTPS / REST+JSON | 443 | صادر | `GET /kyc/citizen/national-no/{no}` | `KycSettings:BaseUrl` |
| 7 | CivicSync.Api | خدمة KYC الخارجية | HTTPS / REST+JSON | 443 | صادر | `POST /kyc/login` (استحواذ على التوكن) | `KycSettings:Username/Password` |
| 8 | CivicSync.Api | خادم SMTP | SMTP / STARTTLS | 587 | صادر | إرسال البريد (غير مُفعّل — القيم فارغة) | `SmtpSettings:*` |
| 9 | المتصفح | CivicSync.Api | HTTPS | 7171 | صادر | واجهة Swagger UI (`/swagger`) | Swashbuckle 6.4.0 |
| 10 | المتصفح | CivicSync.Api | HTTP | 31736 | صادر | IIS Express (تطوير) | `launchSettings.json` (IIS Express) |
| 11 | المتصفح | CivicSync.Api | HTTPS | 44317 | صادر | IIS Express SSL (تطوير) | `launchSettings.json` (IIS Express) |
| 12 | CivicSync.Web ↔ المتصفح | Blazor Server | WebSocket / SignalR | 7284 | ثنائي الاتجاه | نقل دلتا الـ UI و events التفاعلية | `AddInteractiveServerComponents()` |

> **ملاحظة:** المنفذان 7284 و 7171 هما المنفذان المُشار إليهما في ملفات التكوين الفعلية للمشروع، ويجب تعديلهما عند النشر على خادم الإنتاج.

---

## 5. نموذج الأدوار والصلاحيات (Roles & Permissions Model)

### 5.1 الهيكل العام لنموذج الصلاحيات (RBAC Architecture)

```mermaid
flowchart TD
    U["ApplicationUser<br/>المستخدم<br/>Guid Id · IsActive · DirectorateId"]
    R["ApplicationRole<br/>الدور<br/>Name + Description"]
    P["ApplicationPermission<br/>الصلاحية<br/>Code + Description"]
    J["ApplicationRolePermission<br/>جدول الربط<br/>(RoleId, PermissionId)"]

    U -->|"AspNetUserRoles<br/>متعدد-متعدد"| R
    R -->|"ApplicationRolePermissions"| J
    P -->|"ApplicationRolePermissions"| J

    U -.->|"Claim: UserIdentifier · FullName<br/>Email · IsActive · Phone"| TOK["JWT Token<br/>HS256 · 15 دقيقة"]
    R -.->|"ClaimTypes.Role<br/>(one per role)"| TOK
    TOK -->|"Authorization: Bearer"| API["CivicSync.Api<br/>[Authorize(Roles = ...)]"]

    style P fill:#e6f4ea,stroke:#34a853,stroke-width:2px
    style J fill:#f3e8fd,stroke:#a142f4,stroke-width:2px
    style TOK fill:#e8f0fe,stroke:#4285f4,stroke-width:2px
```

### 5.2 تعريف الأدوار الثلاثة (Roles)

مصدر التعريف: `CivicSync.Shared\Constants\Roles.cs`

| # | رمز الدور (Code) | الاسم بالعربية | الوصف الوظيفي | الوصول للوحة التحكم |طاقم API | توجيه |
|---|------------------|----------------|----------------|-------------------|-----------|--------|
| 1 | `Admin` | مدير النظام | مسؤول كامل عن النظام: إدارة المستخدمين، المديريات، الربط مع الداخلية، الإعدادات | ✅ نعم | ✅ نعم | `Roles.DashboardRoles` |
| 2 | `Auditor` | مدقق | صلاحية قراءة فقط: الاطلاع على لوحة المعلومات، بيانات المواطنين، المديريات، وسجلات التدقيق | ✅ نعم | ✅ نعم | `Roles.DashboardRoles` |
| 3 | `FinanceDirectorate` | حساب المديرية | حسابHighered للمديريات المالية — يستهلك **واجهة API فقط** ولا يصل إلى لوحة التحكم | ❌ لا | ✅ نعم | خارج `DashboardRoles` |

**الكود المصدري:**

```csharp
public static class Roles
{
    public const string Admin = "Admin";
    public const string Auditor = "Auditor";
    public const string FinanceDirectorate = "FinanceDirectorate";

    public static readonly string[] All = { Admin, Auditor, FinanceDirectorate };

    /// <summary>الأدوار التي تصل إلى لوحة التحكم (بخلاف FinanceDirectorate الذي يستعمل API فقط).</summary>
    public static readonly string[] DashboardRoles = { Admin, Auditor };
}
```

### 5.3 جدول المستخدمين والأدوار التجريبية (من `CivicSyncMockStore`)

| المعرّف | اسم المستخدم | الاسم الكامل | الدور | المديرية | فعّال | آخر دخول | تغيير كلمة المرور إجباري | عمر التوكن (د) |
|---------|-------------|--------------|-------|----------|-------|----------|---------------------------|----------------|
| 1 | `admin1` | مدير النظام | Admin | — | ✅ | منذ 18 دقيقة | ❌ | — |
| 2 | `auditor1` | أحمد المدقق | Auditor | — | ✅ | منذ 3 ساعات | ✅ | — |
| 3 | `tax_damascus` | حساب مديرية دمشق | FinanceDirectorate | DM | ✅ | منذ 5 دقائق | ❌ | 60 دقيقة |
| 4 | `tax_aleppo` | حساب مديرية حلب | FinanceDirectorate | ALP | ❌ موقوف | — | ❌ | 120 دقيقة |
| 5 | `sarah.user` | سارة خليل | Auditor | — | ✅ | منذ 40 دقيقة | ❌ | — |

### 5.4 تعريف رموز الصلاحيات (Permissions)

مصدر التعريف: `CivicSync.Shared\Constants\Permissions.cs` — إجمالي **9 رموز**.

| # | رمز الصلاحية (Code) | الوصف | وحدة النظام (Module) | طبيعة العملية |
|---|---------------------|-------|----------------------|--------------|
| 1 | `Dashboard.View` | عرض لوحة المعلومات والإحصاءات | لوحة التحكم | قراءة (Read) |
| 2 | `Interior.View` | الاطلاع على حالة الربط مع وزارة الداخلية | الربط الداخلي | قراءة (Read) |
| 3 | `Interior.Manage` | إدارة الربط: تسجيل الدخول، التجديد، فصل الحساب | الربط الداخلي | كتابة (Write) |
| 4 | `Users.Manage` | إنشاء وتعديل وإيقاف/تفعيل وحذف المستخدمين | إدارة المستخدمين | كتابة (Write) |
| 5 | `Directorates.View` | عرض المديريات والفروع | المديريات | قراءة (Read) |
| 6 | `Directorates.Manage` | تعديل المديريات (الحد اليومي، الحالة، الفروع) | المديريات | كتابة (Write) |
| 7 | `Citizens.View` | عرض قوائم citizens وتفاصيلهم | بيانات المواطنين | قراءة (Read) |
| 8 | `Citizens.Lookup` | تنفيذ عمليات الاستعلام (بالرقم الوطني / بالقيد الخماسي) | بيانات المواطنين | كتابة (Write) |
| 9 | `Logs.View` | عرض سجلات الطلبات وسجل التدقيق الإداري | السجلات | قراءة (Read) |

**الكود المصدري:**

```csharp
public static class Permissions
{
    public const string DashboardView = "Dashboard.View";
    public const string InteriorView = "Interior.View";
    public const string InteriorManage = "Interior.Manage";
    public const string UsersManage = "Users.Manage";
    public const string DirectoratesView = "Directorates.View";
    public const string DirectoratesManage = "Directorates.Manage";
    public const string CitizensView = "Citizens.View";
    public const string CitizensLookup = "Citizens.Lookup";
    public const string LogsView = "Logs.View";

    public static readonly string[] All = { ... };  // 9 رموز
}
```

### 5.5 مصفوفة الأدوار × الصلاحيات (RBAC Matrix)

**التوزيع المستهدف (المُصمَّم):**

| الصلاحية | Admin | Auditor | FinanceDirectorate | ملاحظات |
|----------|:-----:|:-------:|:------------------:|---------|
| `Dashboard.View` | ✅ | ✅ | ❌ | المديرية لا تصل للوحة التحكم |
| `Interior.View` | ✅ | ✅ | ❌ | قراءة حالة الربط |
| `Interior.Manage` | ✅ | ❌ | ❌ | عملية حساسة — مدير النظام فقط |
| `Users.Manage` | ✅ | ❌ | ❌ | لا يجوز أن يدير المدقق المستخدمين |
| `Directorates.View` | ✅ | ✅ | ❌ | المدقق يقرأ فقط |
| `Directorates.Manage` | ✅ | ❌ | ❌ | مدير النظام فقط |
| `Citizens.View` | ✅ | ✅ | ✅ | المديرية تستهلك عبر API |
| `Citizens.Lookup` | ✅ | ✅ | ✅ | الاستعلام متاح للجميع عبر API |
| `Logs.View` | ✅ | ✅ | ❌ | سجلات التدقيق للجهة الرقابية |

**الحصيلة:**
- `Admin`: **9 من 9** صلاحيات (100%) — وصول كامل.
- `Auditor`: **6 من 9** صلاحيات (67%) — وصول قراءة فقط.
- `FinanceDirectorate`: **2 من 9** صلاحيات (22%) — صلاحية citizens فقط، وبدون لوحة تحكم.

### 5.6 تطبيق الصلاحيات على نقاط نهاية الـ API (مُطبَّق فعلياً)

| المسار | الطريقة | الحماية الفعلية | الدور المسموح | الصلاحية المكافئة |
|--------|---------|------------------|---------------|--------------------|
| `/api/user/login` | POST | `[AllowAnonymous]` | الجميع | — |
| `/api/user/ResetPassword` | POST | `[AllowAnonymous]` | الجميع | — |
| `/api/user/GetUserById` | GET | `[Authorize(Roles = "Admin")]` | Admin | `Users.Manage` |
| `/api/user/GetAllUsers` | GET | `[Authorize(Roles = "Admin")]` | Admin | `Users.Manage` |
| `/api/user/Register` | POST | `[Authorize(Roles = "Admin")]` | Admin | `Users.Manage` |
| `/api/user/DeleteUser` | DELETE | `[Authorize(Roles = "Admin")]` | Admin | `Users.Manage` |

> **ملاحظة:** المسارات تُحذَّر(conventions) بأحرف صغيرة عبر `options.LowercaseUrls = true`، لذا المسار الفعلي هو `/api/user/getallusers` وليس `/api/user/GetAllUsers`.

### 5.7 تطبيق الصلاحيات على صفحات الواجهة الأمامية (مُطبَّق فعلياً)

| الصفحة | المسار | البوابة (Gate) | Accessibility |
|--------|--------|----------------|---------------|
| لوحة المعلومات | `/` | لا توجد بوابة | Admin + Auditor |
| تسجيل الدخول | `/login` | عامة (EmptyLayout) | الجميع |
| بيانات المواطنين | `/citizens` | لا توجد بوابة | Admin + Auditor |
| تفاصيل المواطن | `/citizens/{Id:int}` | لا توجد بوابة | Admin + Auditor |
| المستخدمون والصلاحيات | `/users` | `Role != Admin` → `AccessDenied` | **Admin فقط** |
| المديريات والفروع | `/directorates` | `Role != Admin` → `AccessDenied` | **Admin فقط** |
| ربط الداخلية | `/interior` | `Role != Admin` → شرط داخلي | **Admin فقط** |
| أداة الاستعلام | `/lookup` | `Role != Admin` → `AccessDenied` | **Admin فقط** |
| سجل الإدارة | `/logs/audit` | لا توجد بوابة | Admin + Auditor |
| سجل الطلبات | `/logs/requests` | لا توجد بوابة | Admin + Auditor |
| إعدادات الحساب | `/account` | لا توجد بوابة | Admin + Auditor |
| صفحة الخطأ | `/Error` | عامة | الجميع |
| صفحة غير موجود | `/not-found` | عامة | الجميع |
| تجربة الحدود | `/dev/borders` | بلا بوابة | الجميع (تطوير) |

**إخفاء عناصر القائمة حسب الدور (في `AppNavbar.razor`):**

| عنصر القائمة | يظهر لـ Admin | يظهر لـ Auditor | يظهر لـ FinanceDirectorate |
|--------------|:------------:|:--------------:|:------------------------:|
| لوحة المعلومات | ✅ | ✅ | ❌ (لا يصل للوحة) |
| بيانات المواطنين | ✅ | ✅ | ❌ |
| سجل الطلبات | ✅ | ✅ | ❌ |
| سجل الإدارة | ✅ | ✅ | ❌ |
| ربط الداخلية | ✅ | ✅ | ❌ |
| المستخدمون والصلاحيات | ✅ | ❌ | ❌ |
| المديريات والفروع | ✅ | ❌ | ❌ |
| أداة الاستعلام | ✅ | ❌ | ❌ |
| إعدادات الحساب | ✅ | ✅ | ❌ |

### 5.8 محتوى رمز JWT (Claims Structure)

يُولَّد الرمز في `JwtService.GenerateJwtToken` بتوقيع **HMAC-SHA256**، ويُحمّل الأدوار كـ `ClaimTypes.Role` (ادعاء واحد لكل دور):

| # | Claim | النوع | المصدر | الاستخدام |
|---|-------|-------|--------|----------|
| 1 | `UserIdentifier` | Guid | `user.Id` | ربط العمليات بالمستخدم في سجلات التدقيق |
| 2 | `FullName` | string | `user.FullName` | عرض الاسم دون استدعاء إضافي |
| 3 | `Email` | string | `user.Email` | تعريف الحساب |
| 4 | `EmailConfirmed` | bool | `user.EmailConfirmed` | حالة تأكيد البريد |
| 5 | `IsActive` | bool | `user.IsActive` | التحقق من حالة الحساب في `CurrentUserService` |
| 6 | `Phone` | string | `user.PhoneNumber` | بيانات الاتصال |
| 7 | `ClaimTypes.Role` | string (متكرر) | `userManager.GetRolesAsync(user)` | التحقق من الدور في `[Authorize(Roles=...)]` |

**معاملات التحقق (`TokenValidationParameters`):**

| المعامل | القيمة | الأثر |
|---------|--------|-------|
| `ValidateIssuer` | `true` | التحقق من مطابقة `Issuer` |
| `ValidateAudience` | `true` | التحقق من مطابقة `Audience` |
| `ValidateLifetime` | `true` | رفض التوكنات المنتهية |
| `ValidateIssuerSigningKey` | `true` | التحقق من صحة التوقيع |
| `ValidIssuer` | `MyApi` | من `JwtSettings:Issuer` |
| `ValidAudience` | `MyUsers` | من `JwtSettings:Audience` |
| `IssuerSigningKey` | `SymmetricSecurityKey` | من `JwtSettings:SecretKey` |
| مدة الصلاحية | 15 دقيقة | قابلة للتخصيص لكل حساب عبر `TokenLifetimeMinutes` (5–1440 دقيقة) |

### 5.9 دورة حياة الحساب (Account Lifecycle State Diagram)

```mermaid
stateDiagram-v2
    [*] --> Active: إنشاء عبر Seeder<br/>(admin@gmail.com)
    [*] --> PendingApproval: RegisterAsync<br/>IsActive=false · EmailConfirmed=false<br/>Role=FinanceDirectorate

    PendingApproval --> Active: تفعيل من لوحة التحكم<br/>(Users → تفعيل)
    PendingApproval --> Suspended: رفض / إيقاف

    Active --> Suspended: إيقاف يدوي (IsActive=false)<br/>أو إيقاف المديرية (عبر SecurityStamp)
    Suspended --> Active: إعادة التفعيل

    Active --> PasswordChangeRequired: MustChangePassword=true
    PasswordChangeRequired --> Active: تغيير كلمة المرور

    Active --> TokenIssued: تسجيل دخول ناجح
    TokenIssued --> TokenIssued: تجديد التوكن
    TokenIssued --> Expired: انتهاء 15 دقيقة
    Expired --> TokenIssued: تسجيل دخول جديد

    Active --> Deleted: DeleteUserAsync
    Deleted --> [*]

    note right of Suspended
        يُرفض الدخول فوراً
        حتى لو كانت كلمة المرور صحيحة
        (تحقق صريح في UserService)
    end note
```

**قواعد الحارس المطبَّقة في `UserService`:**

| القاعدة | التفصيل |
|---------|---------|
| منع تكرار البريد | `FindByEmailAsync` قبل الإنشاء |
| منع تكرار الهاتف | `AnyAsync(s => s.PhoneNumber == phone)` |
| البريد فريد | `options.User.RequireUniqueEmail = true` (Identity) |
| حد دقل كلمة المرور | 8 خانات + حرف كبير + حرف صغير + رقم |
| المحاولات الفاشلة | 5 محاولات → قفل 15 دقيقة (`LockoutEnd`) |
| إيقاف المديرية | يوقف حساباتها تلقائياً عبر `SecurityStamp` |
| مدة التوكن | من 5 إلى 1440 دقيقة (تحقّق في `Users.razor`) |

### 5.10 الثغرات المعمارية في نموذج الصلاحيات (Gaps & Findings)

| # | الملاحظة | الخطورة | الأثر | التوصية |
|---|----------|--------|-------|---------|
| 1 | **رموز `Permissions` غير مفعّلة في الكود** | 🔴 عالية | الرموز التسعة معرَّفة في `Permissions.cs` لكن لا يوجد أي `AuthorizationPolicy` أو `IAuthorizationHandler` يربطها | إنشاء سياسات 基于 على `RequireClaim` / `RoleClaims` وربطها بـ `[Authorize(Policy = "...")]` |
| 2 | **جداول الصلاحيات غير مبذورة** | 🔴 عالية | `ApplicationPermissions` و `ApplicationRolePermissions` لا تُملأ ببيانات أولية عند التشغيل | توسيع `ApplicationDbSeeder` لبذر الـ 9 رموز وربطها بالأدوار الثلاثة |
| 3 | **لا يوجد متحكم Citizens** | 🟠 متوسطة | `CitizenApiClient` يستدعي `api/citizens/*` لكن لا يوجد `CitizensController` في `CivicSync.Api` | إنشاء `CitizensController` مع `[Authorize(Policy = "Citizens.Lookup")]` |
| 4 | **الواجهة الأمامية تستخدم Mock Store لا JWT** | 🟠 متوسطة | `CivicSyncMockStore` يملك `CurrentUser` خاصاً به — التحقق من الدور في الواجهة لا يستند إلى الرمز المُصدَر من الـ API | إضافة `DelegatingHandler` لحقن `Authorization: Bearer` وربط الدور بمصدر واحد موثوق |
| 5 | **سياسة CORS مفتوحة بالكامل** | 🟠 متوسطة | `SetIsOriginAllowed(_ => true)` مع `AllowCredentials` | تقييد `AllowedOrigins` في الإنتاج |
| 6 | **الصلاحيات لا تُطبَّق على الصفحات** | 🟡 منخفضة | الفحص يتم بالدور (`UserRole.Admin`) وليس برمز الصلاحية (`Permissions.*`) | توحيد نموذج الفحص على مستوى الصلاحيات |
| 7 | **الأسرار في ملفات JSON** | 🔴 عالية | `SecretKey` وكلمة مرور قاعدة البيانات داخل المستودع | نقلها إلى متغيرات البيئة / Secret Manager |
| 8 | **`BackgroundJobService` غير مُنفَّذ** | 🟡 منخفضة | `RunUserCleanupAsync()` يرمي `NotImplementedException` | تنفيذ الخدمة أو حذف العقد حتى لا تُربك المتصلين |
| 9 | **لا يوجد Refresh Token** | 🟡 منخفضة | التوكن صالح 15 دقيقة فقط، والم amaz في `JwtService` معطّل | تفعيل تدفق Refresh Token للتجربة السلسة |

---

## 6. ملخص الطوبولوجيا في صور مختصرة

### 6.1 خريطة المسارات (Route Map)

```mermaid
flowchart TD
    subgraph WebApp["CivicSync.Web — 7284/5297"]
        R1["/"] --> R2["/citizens"] --> R3["/citizens/{Id:int}"]
        R1 --> R4["/logs/requests"]
        R1 --> R5["/logs/audit"]
        R1 --> R6["/interior"]
        R1 --> R7["/users"]
        R1 --> R8["/directorates"]
        R1 --> R9["/lookup"]
        R1 --> R10["/account"]
        R11["/login"]
        R12["/Error"] 
        R13["/not-found"]
        R14["/dev/borders"]
    end

    subgraph ApiApp["CivicSync.Api — 7171/5234"]
        A1["/api/user/login<br/>AllowAnonymous"]
        A2["/api/user/ResetPassword<br/>AllowAnonymous"]
        A3["/api/user/GetUserById<br/>Admin"]
        A4["/api/user/GetAllUsers<br/>Admin"]
        A5["/api/user/Register<br/>Admin"]
        A6["/api/user/DeleteUser<br/>Admin"]
        A7["/api/citizens/*<br/>مطلوب — غير مُنفَّذ"]
        A8["/swagger"]
        A9["/index.html"]
    end

    R2 -.->|"HttpClient<br/>ApiBaseUrl"| A7
    R3 -.->|"HttpClient"| A7
    R11 -.->|"REST (مخطط له)"| A1
    A3 --> DB2[("SQL Server")]
    A4 --> DB2
    A6 --> DB2
    A7 --> DB2

    style A7 fill:#fce8e6,stroke:#ea4335,stroke-width:2px,stroke-dasharray: 5 5
    style R14 fill:#f3e8fd,stroke:#a142f4
```

### 6.2 بطاقة تعريف المكوّنات (Component Cards)

| المكوّن | التقنية | البروتوكول الداخلي | البروتوكول الخارجي | المنافذ | قاعدة البيانات | المصادقة |
|---------|---------|--------------------|--------------------|---------|---------------|-----------|
| **CivicSync.Web** | Blazor Server · .NET 10 · Tailwind 4.3.3 · ApexCharts 7.6.1 | Blazor Server ↔ Browser (WebSocket/SignalR) | HTTPS REST → CivicSync.Api | 7284 (HTTPS) / 5297 (HTTP) | ❌ لا يتصل مباشرة | Mock Store (وحدة) / JWT (مخطط له) |
| **CivicSync.Api** | ASP.NET Core Web API · .NET 10 · Swashbuckle 6.4.0 | ASP.NET Core Pipeline + DI | HTTPS REST ↔ Web · HTTPS → KYC · TCP → SQL Server | 7171 (HTTPS) / 5234 (HTTP) | ✅ SQL Server عبر EF Core 10.0.12 | JWT Bearer (HS256) + Identity |
| **CivicSync.Application** | مكتبة .NET 10 | استدعاءات مباشرة (Interfaces) | — | — | عبر `IUnitOfWork` | — |
| **CivicSync.Infrastructure** | مكتبة .NET 10 | EF Core + HttpClient | HTTPS → KYC | 443 (خارجي) | ✅ `ApplicationDbContext` | JwtService / CurrentUserService |
| **CivicSync.Core** | مكتبة .NET 10 | عقود (Interfaces) | — | — | يعرّف الكيانات فقط | — |
| **CivicSync.Shared** | مكتبة .NET 10 | عقود DTO | — | — | — | — |
| **SQL Server** | Microsoft SQL Server | TDS / TCP | — | 1433 | `CivicSyncDB` (14 جدولاً) | مصادقة SQL أو Windows |

---

## 7. الخلاصة والتوصيات المعمارية

### 7.1 نقاط القوة

1. **فصل Concerns محكم** عبر Clean Architecture مع قاعدة اعتمادية ملتزم بها (Core لا يعتمد على أحد).
2. **عزل الواجهة الأمامية** عن منطق الأعمال (Web ↔ API عبر HTTP فقط) يسمح بتبديل الـ SPA مستقبلاً دون تعديل الـ Backend.
3. **تغطية شاملة للتدقيق** عبر جدولَي `AuditLogs` و `LookupRequestLogs` مع `DurationMs` و `IpAddress`.
4. **نموذج صلاحيات مُعدّ مسبقاً** بكيانات Relation كاملة (ApplicationPermission + ApplicationRolePermission) و 9 رموز منطقية منظّمة.
5. **تطبيع النصوص العربية** (توحيد الهمزات والتاء المربوطة والتشكيل) مع فهرس مركّب `IX_Citizens_NormalizedFive` — قرار معماري صائب للبحث بالقيد الخماسي.
6. **سجل Token متزامن** (`RowVersion` على Citizen) لحماية من تعارضات الكتابة المتزامنة.

### 7.2 الأولويات المقترحة (Roadmap)

| الأولوية | البند | الجهد | الأثر |
|:--------:|-------|:-----:|-------|
| 🔴 P0 | تفعيل رموز `Permissions` عبر سياسات ASP.NET Core Authorization | متوسط | إغلاق فجوة التصميم والتطبيق |
| 🔴 P0 | بذر جداول الصلاحيات والأدوار في `ApplicationDbSeeder` | صغير | تشغيل النموذج فعلياً |
| 🔴 P0 | نقل الأسرار إلى متغيرات البيئة / Secret Manager | صغير | أمان إنتاجي |
| 🟠 P1 | إنشاء `CitizensController` يغطي مسارات `CitizenApiClient` | متوسط | اكتمال الـ API |
| 🟠 P1 | ربط الواجهة الأمامية برمز JWT حقيقي عبر `DelegatingHandler` | متوسط | مصدر واحد موثوق للصلاحيات |
| 🟠 P1 | تقييد CORS بنطاقات الإنتاج | صغير | أمان |
| 🟡 P2 | تفعيل Refresh Token | متوسط | تجربة مستخدم |
| 🟡 P2 | تنفيذ `BackgroundJobService` (تنظيف المستخدمين) | صغير | اكتمال الوظائف |
| 🟡 P2 | إضافة اختبارات آلية (xUnit) لطبقة الخدمات | متوسط | ضمان الجودة |

---

**نهاية التقرير**  
تم إعداد هذا التقرير استناداً إلى فحص مباشر لكود مصدر المشروع `CivicSync` (6 مشاريع · 64 ملف C# · 14 صفحة Razor · 14 جدولاً في قاعدة البيانات) وإعداده بنمط جاهز للنسخ إلى Microsoft Word.
