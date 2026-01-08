# 09-logging-monitoring.md

# التسجيل والمراقبة (Logging & Monitoring)

## المقدمة

التسجيل (Logging) والمراقبة (Monitoring) هما عنصران حيويان لأي تطبيق برمجي، خاصة في بيئات الإنتاج. يسمحان لك بفهم سلوك تطبيقك، تتبع المشاكل، تشخيص الأخطاء، وقياس الأداء. ASP.NET Core يوفر نظام تسجيل قوي ومرن يمكن تكوينه بسهولة، بالإضافة إلى أدوات لمراقبة صحة التطبيق.

## 1. التسجيل (Logging) في ASP.NET Core

ASP.NET Core لديه نظام تسجيل مدمج (Built-in Logging) يدعم العديد من مزودي التسجيل (Logging Providers) مثل Console, Debug, EventSource, EventLog، ويمكن توسيعه لدعم مزودين خارجيين مثل Serilog, NLog, Log4Net.

### أ. استخدام `ILogger<T>`

يمكنك حقن واجهة `ILogger<T>` في أي فئة (عبر حقن التبعية) لتسجيل الرسائل. `T` هنا هو اسم الفئة التي تقوم بالتسجيل، مما يساعد في تحديد مصدر الرسالة.

```csharp
using Microsoft.Extensions.Logging;

public class MyService
{
    private readonly ILogger<MyService> _logger;

    public MyService(ILogger<MyService> logger)
    {
        _logger = logger;
    }

    public void DoWork()
    {
        _logger.LogInformation("Starting DoWork method.");
        try
        {
            // ... بعض العمل ...
            _logger.LogWarning("Potential issue detected in DoWork.");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An error occurred during DoWork.");
        }
        _logger.LogDebug("DoWork method finished.");
    }
}
```

### ب. مستويات التسجيل (Log Levels)

تحدد مستويات التسجيل أهمية الرسالة. يمكنك تكوين الحد الأدنى للمستوى الذي سيتم تسجيله لكل مزود أو لكل فئة.

| المستوى | الوصف |
| --- | --- |
| **Trace** | رسائل مفصلة جداً، قد تحتوي على بيانات حساسة. |
| **Debug** | معلومات مفيدة لتصحيح الأخطاء أثناء التطوير. |
| **Information** | تتبع عام لسير التطبيق، مفيد للمراقبة. |
| **Warning** | حدث غير طبيعي أو غير متوقع، قد يشير إلى مشكلة محتملة. |
| **Error** | خطأ في العملية الحالية أو المكون. |
| **Critical** | فشل كارثي يتطلب اهتماماً فورياً. |

### ج. تكوين التسجيل في `appsettings.json`

يمكنك التحكم في مستويات التسجيل من خلال ملف `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "MyApiProject.Controllers": "Debug" // لتسجيل رسائل Debug من المتحكمات
    },
    "Console": {
      "LogLevel": {
        "Default": "Information"
      }
    }
  },
  "AllowedHosts": "*"
}
```

### د. استخدام Serilog (كمثال لمزود خارجي)

Serilog هو مكتبة تسجيل شائعة توفر ميزات غنية مثل التسجيل المنظم (Structured Logging) والقدرة على الكتابة إلى مصادر متعددة (Sinks) مثل الملفات، قواعد البيانات، وخدمات المراقبة السحابية.

1.  **تثبيت حزم NuGet:**

    ```bash
    dotnet add package Serilog.AspNetCore
    dotnet add package Serilog.Sinks.Console
    dotnet add package Serilog.Sinks.File
    ```

2.  **تكوين Serilog في `Program.cs`:**

    ```csharp
    using Serilog;

    var builder = WebApplication.CreateBuilder(args);

    // تكوين Serilog
    builder.Host.UseSerilog((context, configuration) =>
        configuration.ReadFrom.Configuration(context.Configuration)
                     .WriteTo.Console()
                     .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day));

    // ... إضافة خدمات أخرى ...

    var app = builder.Build();

    // ... تكوين الـ Middleware ...

    app.Run();
    ```

3.  **تكوين Serilog في `appsettings.json`:**

    ```json
    {
      "Serilog": {
        "MinimumLevel": {
          "Default": "Information",
          "Override": {
            "Microsoft": "Warning",
            "System": "Warning"
          }
        },
        "WriteTo": [
          { "Name": "Console" },
          { "Name": "File", "Args": { "path": "logs/log-.txt", "rollingInterval": "Day" } }
        ]
      },
      "AllowedHosts": "*"
    }
    ```

## 2. مراقبة صحة التطبيق (Health Checks)

ASP.NET Core Health Checks هي ميزة تسمح لك بمراقبة صحة تطبيقك والخدمات التي يعتمد عليها (مثل قواعد البيانات، خدمات خارجية). يمكن استخدامها بواسطة أدوات المراقبة الخارجية أو موازنات التحميل (Load Balancers) لتحديد ما إذا كان التطبيق يعمل بشكل صحيح.

### أ. إعداد Health Checks

1.  **إضافة خدمة Health Checks في `Program.cs`:**

    ```csharp
    // Program.cs
    var builder = WebApplication.CreateBuilder(args);

    // ... إضافة خدمات أخرى ...

    builder.Services.AddHealthChecks()
        .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"), name: "SQL Server");
        // يمكنك إضافة فحوصات أخرى مثل:
        // .AddRedis("localhost:6379", name: "Redis Cache");
        // .AddUrlGroup(new Uri("https://google.com"), name: "Google");

    var app = builder.Build();

    // ... تكوين الـ Middleware ...

    // إضافة Health Check Endpoint
    app.MapHealthChecks("/health"); // نقطة نهاية بسيطة
    app.MapHealthChecks("/health-detailed", new HealthCheckOptions
    {
        ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse // لعرض تفاصيل أكثر
    });

    app.Run();
    ```

2.  **تثبيت حزمة `AspNetCore.HealthChecks.UI.Client` (لـ `UIResponseWriter`):**

    ```bash
    dotnet add package AspNetCore.HealthChecks.UI.Client
    ```

### ب. أنواع Health Checks

*   **Liveness Checks:** تحدد ما إذا كان التطبيق قيد التشغيل. إذا فشل، يجب إعادة تشغيل التطبيق.
*   **Readiness Checks:** تحدد ما إذا كان التطبيق جاهزاً لاستقبال الطلبات. إذا فشل، يجب إزالة التطبيق من موازنة التحميل مؤقتاً.

### ج. استخدام Health Checks

بعد الإعداد، يمكنك الوصول إلى نقاط النهاية (Endpoints) الخاصة بالـ Health Checks من خلال المتصفح أو أدوات المراقبة:

*   `GET /health`: سيعود بـ `HTTP 200 OK` إذا كانت جميع الفحوصات ناجحة، أو `HTTP 503 Service Unavailable` إذا فشل أي منها.
*   `GET /health-detailed`: سيعرض تقريراً مفصلاً بحالة كل فحص.

## 3. مقاييس الأداء (Performance Metrics)

بالإضافة إلى التسجيل وفحوصات الصحة، من المهم جمع مقاييس الأداء (Metrics) لمراقبة سلوك التطبيق بمرور الوقت. يمكن لـ ASP.NET Core دمجها مع أنظمة مراقبة مثل Prometheus, Grafana, Azure Application Insights.

### أمثلة على المقاييس التي يجب مراقبتها:

*   **CPU Usage:** استخدام المعالج.
*   **Memory Usage:** استخدام الذاكرة.
*   **Request Rate:** عدد الطلبات في الثانية.
*   **Response Time:** متوسط وقت الاستجابة للطلبات.
*   **Error Rate:** نسبة الأخطاء إلى إجمالي الطلبات.
*   **Database Query Times:** أوقات استعلامات قاعدة البيانات.

## الخلاصة

التسجيل والمراقبة هما أدوات لا غنى عنها لضمان استقرار وأداء تطبيقات ASP.NET Core. من خلال استخدام نظام التسجيل المدمج أو مكتبات خارجية مثل Serilog، وتطبيق Health Checks، وجمع مقاييس الأداء، يمكنك الحصول على رؤية شاملة لتطبيقك واتخاذ قرارات مستنيرة لتحسينه. في الدرس التالي، سنتناول استراتيجيات التخزين المؤقت لتحسين الأداء.
