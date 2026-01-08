# 11-background-tasks.md

# المهام الخلفية (Background Tasks) و Hangfire

## المقدمة

في تطبيقات الويب، غالباً ما تكون هناك مهام لا تحتاج إلى معالجة فورية أو قد تستغرق وقتاً طويلاً لإكمالها. تنفيذ هذه المهام بشكل متزامن (Synchronously) ضمن دورة طلب HTTP يمكن أن يؤدي إلى تجربة مستخدم سيئة (استجابات بطيئة) واستهلاك غير فعال لموارد الخادم. هنا يأتي دور **المهام الخلفية (Background Tasks)**، حيث يتم تنفيذ هذه العمليات خارج سياق طلب HTTP، مما يحرر الخادم للاستجابة لطلبات أخرى بسرعة. ASP.NET Core يوفر آليات لتشغيل المهام الخلفية، وتعتبر مكتبة **Hangfire** حلاً قوياً لإدارة هذه المهام.

## 1. المهام الخلفية في ASP.NET Core

ASP.NET Core يوفر واجهة `IHostedService` لإنشاء خدمات تعمل في الخلفية طوال عمر التطبيق. هذه الخدمات تبدأ عند بدء تشغيل التطبيق وتتوقف عند إيقافه.

### أ. إنشاء `IHostedService` بسيط

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public class MyBackgroundService : BackgroundService
{
    private readonly ILogger<MyBackgroundService> _logger;

    public MyBackgroundService(ILogger<MyBackgroundService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("MyBackgroundService is starting.");

        stoppingToken.Register(() =>
            _logger.LogInformation("MyBackgroundService is stopping."));

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("MyBackgroundService performing background work at: {time}", DateTimeOffset.Now);
            
            // قم بتنفيذ مهمتك الخلفية هنا
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }

        _logger.LogInformation("MyBackgroundService has stopped.");
    }
}
```

### ب. تسجيل `IHostedService`

يتم تسجيل الخدمات المستضافة في `Program.cs`:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// ... إضافة خدمات أخرى ...

builder.Services.AddHostedService<MyBackgroundService>();

var app = builder.Build();
// ...
```

**متى تستخدم `IHostedService`؟**

*   للمهام التي تحتاج إلى التشغيل المستمر طوال عمر التطبيق (مثل مراقبة قائمة انتظار، معالجة رسائل).
*   للمهام التي لا تتطلب جدولة معقدة أو متانة (Persistence).

## 2. Hangfire: إدارة المهام الخلفية المتقدمة

Hangfire هي مكتبة قوية وسهلة الاستخدام لإدارة المهام الخلفية في ASP.NET Core. توفر جدولة المهام (Scheduling)، معالجة المهام المتكررة (Recurring Tasks)، ومعالجة المهام التي تفشل (Retries)، كل ذلك مع واجهة مستخدم (Dashboard) لمراقبة المهام.

### أ. ميزات Hangfire

*   **Persistence:** تخزين المهام في قاعدة بيانات (SQL Server, PostgreSQL, MongoDB, Redis) لضمان عدم فقدانها حتى لو تعطل التطبيق.
*   **Different Job Types:** دعم لمهام لمرة واحدة (Fire-and-forget)، مهام مؤجلة (Delayed)، مهام متكررة (Recurring)، ومهام متابعة (Continuations).
*   **Dashboard:** واجهة ويب لمراقبة حالة المهام، إعادة تشغيلها، أو حذفها.
*   **Automatic Retries:** إعادة محاولة تنفيذ المهام التي تفشل تلقائياً.

### ب. إعداد Hangfire

1.  **تثبيت حزم NuGet:**

    ```bash
    dotnet add package Hangfire
dotnet add package Hangfire.AspNetCore
dotnet add package Hangfire.SqlServer # أو Hangfire.PostgreSql, Hangfire.Mongo, إلخ.
    ```

2.  **تكوين Hangfire في `Program.cs`:**

    ```csharp
    // Program.cs
    using Hangfire;
    using Hangfire.SqlServer;

    var builder = WebApplication.CreateBuilder(args);

    // ... إضافة خدمات أخرى ...

    // إضافة Hangfire Services
    builder.Services.AddHangfire(configuration => configuration
        .SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
        .UseSimpleAssemblyNameTypeSerializer()
        .UseRecommendedSerializerSettings()
        .UseSqlServerStorage(builder.Configuration.GetConnectionString("HangfireConnection")));

    // إضافة Hangfire Server (لتشغيل المهام)
    builder.Services.AddHangfireServer();

    var app = builder.Build();

    // ... تكوين الـ Middleware ...

    // استخدام Hangfire Dashboard (يجب أن يكون في بيئة آمنة في الإنتاج)
    app.UseHangfireDashboard();

    // جدولة المهام هنا أو في خدمة أخرى
    // BackgroundJob.Enqueue(() => Console.WriteLine("Fire-and-forget job!"));
    // RecurringJob.AddOrUpdate("daily-report", () => Console.WriteLine("Daily report!"), Cron.Daily);

    app.Run();
    ```

3.  **إضافة سلسلة اتصال Hangfire في `appsettings.json`:**

    ```json
    {
      "ConnectionStrings": {
        "DefaultConnection": "...",
        "HangfireConnection": "Server=(localdb)\\mssqllocaldb;Database=HangfireDb;Trusted_Connection=True;MultipleActiveResultSets=true"
      },
      // ...
    }
    ```

### ج. أنواع المهام في Hangfire

*   **Fire-and-Forget Jobs:** يتم تنفيذها مرة واحدة في الخلفية بعد فترة وجيزة.

    ```csharp
    BackgroundJob.Enqueue(() => Console.WriteLine("Hello from a fire-and-forget job!"));
    ```

*   **Delayed Jobs:** يتم تنفيذها مرة واحدة في الخلفية بعد فترة زمنية محددة.

    ```csharp
    BackgroundJob.Schedule(() => Console.WriteLine("Hello from a delayed job!"), TimeSpan.FromDays(7));
    ```

*   **Recurring Jobs:** يتم تنفيذها بشكل متكرر وفقاً لجدول زمني (Cron expression).

    ```csharp
    RecurringJob.AddOrUpdate(
        "easy-job",
        () => Console.WriteLine("Easy! "),
        Cron.Minutely); // كل دقيقة

    RecurringJob.AddOrUpdate(
        "hard-job",
        () => Console.WriteLine("Hard! "),
        "0 12 * * *", // كل يوم الساعة 12 ظهراً
        TimeZoneInfo.Local);
    ```

*   **Continuations:** يتم تنفيذها بعد اكتمال مهمة أخرى.

    ```csharp
    var jobId = BackgroundJob.Enqueue(() => Console.WriteLine("Parent job"));
    BackgroundJob.ContinueWith(jobId, () => Console.WriteLine("Continuation job"));
    ```

### د. حماية Hangfire Dashboard

واجهة Hangfire Dashboard تعرض معلومات حساسة عن المهام. في بيئة الإنتاج، يجب حمايتها. يمكنك استخدام `IDashboardAuthorizationFilter` لتقييد الوصول.

```csharp
// Filters/HangfireAuthorizationFilter.cs
using Hangfire.Dashboard;

public class HangfireAuthorizationFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        // هنا يمكنك إضافة منطق التحقق من الصلاحيات
        // مثلاً، التحقق من أن المستخدم مصادق عليه ولديه دور معين
        // var httpContext = context.Get//HttpContext();
        // return httpContext.User.Identity.IsAuthenticated && httpContext.User.IsInRole("Admin");

        // للسماح بالوصول في بيئة التطوير فقط
        return true; // يجب تغيير هذا في الإنتاج
    }
}

// في Program.cs عند تكوين Dashboard
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new [] { new HangfireAuthorizationFilter() }
});
```

## الخلاصة

المهام الخلفية ضرورية لبناء تطبيقات ويب سريعة الاستجابة وفعالة. سواء كنت تستخدم `IHostedService` للمهام المستمرة البسيطة أو Hangfire لإدارة المهام المجدولة والمعقدة، فإن دمج المهام الخلفية في تطبيقك سيحسن بشكل كبير من أدائه وتجربة المستخدم. في الدرس التالي، سنتناول كيفية ضمان جودة الكود من خلال اختبارات الوحدة (Unit Testing).
