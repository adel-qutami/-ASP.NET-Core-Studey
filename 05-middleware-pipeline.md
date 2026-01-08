# 05-middleware-pipeline.md

# فهم الـ Middleware ودورة حياة الطلب

## المقدمة

في ASP.NET Core، يتم معالجة كل طلب HTTP يمر عبر التطبيق بواسطة سلسلة من المكونات تسمى **Middleware**. هذه المكونات تتحد لتشكيل **Request Pipeline** (خط أنابيب الطلب). كل Middleware يمكنه معالجة الطلب قبل تمريره إلى المكون التالي، أو بعد عودة الاستجابة من المكونات اللاحقة، أو حتى إنهاء الطلب بالكامل.

## 1. ما هو الـ Middleware؟

الـ Middleware هو برنامج صغير يتم تجميعه في سلسلة لمعالجة طلبات HTTP والاستجابات. كل مكون Middleware:

*   يختار ما إذا كان سيمرر الطلب إلى المكون التالي في الـ Pipeline.
*   يمكنه إجراء عمليات قبل وبعد استدعاء المكون التالي.
*   يمكنه تعديل كائن الطلب (HttpRequest) وكائن الاستجابة (HttpResponse).

### هيكلية الـ Middleware

```
Request
  ↓
Middleware 1 (مثلاً: Logging)
  ↓
Middleware 2 (مثلاً: Authentication)
  ↓
Middleware 3 (مثلاً: Routing)
  ↓
Endpoint (Controller/Minimal API)
  ↑
Middleware 3 (بعد المعالجة)
  ↑
Middleware 2 (بعد المعالجة)
  ↑
Middleware 1 (بعد المعالجة)
  ↑
Response
```

## 2. تكوين الـ Request Pipeline

يتم تكوين الـ Request Pipeline في ملف `Program.cs` (أو `Startup.cs` في الإصدارات القديمة) باستخدام كائن `WebApplication` (أو `IApplicationBuilder`). يتم ترتيب مكونات الـ Middleware حسب الترتيب الذي يتم إضافتها به، وهذا الترتيب مهم جداً.

### أمثلة على تكوين الـ Middleware

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

// إضافة الخدمات إلى حاوية DI (قبل بناء الـ app)
builder.Services.AddControllers();
builder.Services.AddSwaggerGen(); // لـ Swagger/OpenAPI

var app = builder.Build();

// تكوين الـ Request Pipeline (ترتيب الـ Middleware مهم!)

// 1. Middleware لمعالجة الاستثناءات (يجب أن يكون الأول)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // صفحة استثناءات مفصلة في وضع التطوير
    app.UseSwagger(); // تمكين Swagger UI في وضع التطوير
    app.UseSwaggerUI();
}
else
{
    app.UseExceptionHandler("/Error"); // صفحة خطأ عامة في وضع الإنتاج
    app.UseHsts(); // إضافة رأس HSTS للأمان
}

// 2. Middleware لإعادة توجيه HTTP إلى HTTPS
app.UseHttpsRedirection();

// 3. Middleware لتقديم الملفات الثابتة (مثل CSS, JS, صور من wwwroot)
app.UseStaticFiles();

// 4. Middleware للتوجيه (Routing)
app.UseRouting();

// 5. Middleware للمصادقة (Authentication) - يجب أن يأتي بعد UseRouting وقبل UseAuthorization
app.UseAuthentication();

// 6. Middleware للتفويض (Authorization) - يجب أن يأتي بعد UseAuthentication
app.UseAuthorization();

// 7. Middleware لربط الطلبات بنقاط النهاية (Endpoints) - Controllers, Minimal APIs
app.MapControllers(); // لـ Web API Controllers
// app.MapRazorPages(); // لـ Razor Pages
// app.MapGet("/", () => "Hello World!"); // مثال على Minimal API

app.Run();
```

**ترتيب الـ Middleware مهم:**

*   الـ Middleware الذي يتعامل مع الأخطاء (مثل `UseDeveloperExceptionPage` أو `UseExceptionHandler`) يجب أن يكون في بداية الـ Pipeline لالتقاط أي استثناءات تحدث في الـ Middleware اللاحقة.
*   الـ Middleware الذي يخدم الملفات الثابتة (`UseStaticFiles`) يجب أن يأتي مبكراً لتجنب معالجة الطلبات للملفات الثابتة بواسطة الـ Middleware الأكثر تعقيداً.
*   الـ Middleware الخاص بالمصادقة والتفويض (`UseAuthentication`, `UseAuthorization`) يجب أن يأتي بعد `UseRouting` وقبل `MapControllers` (أو `MapEndpoints`) لضمان تحديد المسار قبل تطبيق قواعد الأمان.

## 3. الـ Middleware المدمج (Built-in Middleware)

ASP.NET Core يوفر العديد من مكونات الـ Middleware الجاهزة للاستخدام:

| Middleware | الوصف |
| --- | --- |
| `UseDeveloperExceptionPage` | يعرض صفحة خطأ مفصلة للمطورين. |
| `UseExceptionHandler` | يعالج الاستثناءات في بيئة الإنتاج. |
| `UseHttpsRedirection` | يعيد توجيه طلبات HTTP إلى HTTPS. |
| `UseStaticFiles` | يخدم الملفات الثابتة من مجلد `wwwroot`. |
| `UseRouting` | يطابق عناوين URL مع نقاط النهاية. |
| `UseAuthentication` | يقوم بالمصادقة على المستخدمين. |
| `UseAuthorization` | يتحقق من صلاحيات المستخدمين. |
| `UseCors` | يضيف دعم CORS (Cross-Origin Resource Sharing). |
| `UseResponseCaching` | يضيف دعم التخزين المؤقت للاستجابات. |
| `UseSession` | يضيف دعم إدارة الجلسات. |

## 4. كتابة Custom Middleware

يمكنك كتابة مكونات Middleware مخصصة لتلبية احتياجات تطبيقك الخاصة. هناك طريقتان رئيسيتان:

### أ. Middleware باستخدام `app.Use()` (Inline Middleware)

هذه الطريقة مناسبة للـ Middleware البسيط الذي لا يتطلب حقن تبعية معقدة.

```csharp
app.Use(async (context, next) =>
{
    // منطق يتم تنفيذه قبل تمرير الطلب
    Console.WriteLine($"Request received for: {context.Request.Path}");

    await next(); // تمرير الطلب إلى الـ Middleware التالي

    // منطق يتم تنفيذه بعد عودة الاستجابة
    Console.WriteLine($"Response sent for: {context.Request.Path} with status: {context.Response.StatusCode}");
});
```

### ب. Middleware كفئة منفصلة (Class-based Middleware)

هذه الطريقة مفضلة للـ Middleware الأكثر تعقيداً، وتسمح بحقن التبعية.

1.  **إنشاء الفئة:**

    ```csharp
    using Microsoft.AspNetCore.Http;
    using System.Threading.Tasks;

    public class MyCustomMiddleware
    {
        private readonly RequestDelegate _next;

        public MyCustomMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            // منطق يتم تنفيذه قبل تمرير الطلب
            Console.WriteLine($"Custom Middleware: Before next - {context.Request.Path}");

            await _next(context); // تمرير الطلب إلى الـ Middleware التالي

            // منطق يتم تنفيذه بعد عودة الاستجابة
            Console.WriteLine($"Custom Middleware: After next - {context.Response.StatusCode}");
        }
    }
    ```

2.  **إضافة الفئة إلى الـ Pipeline:**

    ```csharp
    // في Program.cs
    app.UseMiddleware<MyCustomMiddleware>();
    
    // أو كـ Extension Method لسهولة الاستخدام
    public static class MyCustomMiddlewareExtensions
    {
        public static IApplicationBuilder UseMyCustomMiddleware(
            this IApplicationBuilder builder)
        {
            return builder.UseMiddleware<MyCustomMiddleware>();
        }
    }
    // ثم استخدامها هكذا:
    // app.UseMyCustomMiddleware();
    ```

## الخلاصة

الـ Middleware والـ Request Pipeline هما مفاهيم أساسية في ASP.NET Core تتحكم في كيفية معالجة طلبات HTTP. فهم كيفية عملها، ترتيبها، وكيفية كتابة Middleware مخصص سيمكنك من بناء تطبيقات ويب قوية، مرنة، وقابلة للتوسع. في الدرس التالي، سنتعمق في كيفية التعامل مع قواعد البيانات باستخدام Entity Framework Core.
