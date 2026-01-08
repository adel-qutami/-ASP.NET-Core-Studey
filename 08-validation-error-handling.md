# 08-validation-error-handling.md

# التحقق من البيانات ومعالجة الأخطاء

## المقدمة

تطبيقات الويب تتعامل باستمرار مع مدخلات المستخدم، ومن الضروري التحقق من صحة هذه المدخلات (Validation) لضمان سلامة البيانات ومنع الثغرات الأمنية. بالإضافة إلى ذلك، يجب أن يتعامل التطبيق مع الأخطاء (Error Handling) بشكل رشيق وودود للمستخدم، بدلاً من عرض رسائل خطأ تقنية قد تكشف عن تفاصيل حساسة للنظام.

## 1. التحقق من صحة البيانات (Data Validation)

ASP.NET Core يوفر آليات قوية للتحقق من صحة البيانات على مستوى النموذج (Model) باستخدام سمات (Data Annotations) أو Fluent Validation.

### أ. Data Annotations

هي سمات يمكنك تطبيقها مباشرة على خصائص النموذج لتحديد قواعد التحقق.

```csharp
using System.ComponentModel.DataAnnotations;

public class ProductDto
{
    [Required(ErrorMessage = "اسم المنتج مطلوب.")]
    [StringLength(100, MinimumLength = 3, ErrorMessage = "يجب أن يكون طول الاسم بين 3 و 100 حرف.")]
    public string Name { get; set; } = string.Empty;

    [Range(0.01, 10000.00, ErrorMessage = "يجب أن يكون السعر بين 0.01 و 10000.")]
    public decimal Price { get; set; }

    [EmailAddress(ErrorMessage = "صيغة البريد الإلكتروني غير صحيحة.")]
    public string? ManufacturerEmail { get; set; }

    [Url(ErrorMessage = "صيغة الرابط غير صحيحة.")]
    public string? Website { get; set; }
}
```

**التحقق التلقائي في المتحكمات:**

عند استخدام `[ApiController]`، يقوم ASP.NET Core تلقائياً بالتحقق من صحة النموذج. إذا فشل التحقق، فإنه يُرجع استجابة `400 Bad Request` مع تفاصيل الأخطاء.

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateProduct([FromBody] ProductDto productDto)
    {
        // لا حاجة للتحقق يدوياً هنا، [ApiController] يقوم بذلك تلقائياً
        // إذا كان ModelState غير صالح، سيتم إرجاع 400 Bad Request
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState); // هذا السطر لن يتم الوصول إليه عادةً مع [ApiController]
        }

        // ... حفظ المنتج ...
        return Ok("Product created successfully!");
    }
}
```

### ب. Fluent Validation

للقواعد الأكثر تعقيداً أو لفصل قواعد التحقق عن النموذج، يمكنك استخدام مكتبة Fluent Validation. 

1.  **تثبيت حزمة NuGet:**

    ```bash
    dotnet add package FluentValidation.AspNetCore
    ```

2.  **تعريف Validator:**

    ```csharp
    using FluentValidation;

    public class ProductDtoValidator : AbstractValidator<ProductDto>
    {
        public ProductDtoValidator()
        {
            RuleFor(product => product.Name)
                .NotEmpty().WithMessage("اسم المنتج مطلوب.")
                .Length(3, 100).WithMessage("يجب أن يكون طول الاسم بين 3 و 100 حرف.");

            RuleFor(product => product.Price)
                .GreaterThan(0).WithMessage("يجب أن يكون السعر أكبر من صفر.");

            RuleFor(product => product.ManufacturerEmail)
                .EmailAddress().When(product => !string.IsNullOrEmpty(product.ManufacturerEmail))
                .WithMessage("صيغة البريد الإلكتروني غير صحيحة.");
        }
    }
    ```

3.  **تسجيل Fluent Validation في `Program.cs`:**

    ```csharp
    // Program.cs
    using FluentValidation.AspNetCore;

    var builder = WebApplication.CreateBuilder(args);

    builder.Services.AddControllers()
        .AddFluentValidation(fv => fv.RegisterValidatorsFromAssemblyContaining<ProductDtoValidator>());

    // ...
    ```

## 2. معالجة الأخطاء (Error Handling)

يجب أن يتعامل التطبيق مع الاستثناءات (Exceptions) والأخطاء بطريقة مركزية لتوفير تجربة مستخدم متسقة ومنع تسرب المعلومات الحساسة.

### أ. استخدام `UseExceptionHandler`

في بيئة الإنتاج، يمكنك استخدام `UseExceptionHandler` لعرض صفحة خطأ عامة أو إرجاع استجابة JSON موحدة.

```csharp
// Program.cs
var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // للمطورين
}
else
{
    app.UseExceptionHandler("/Error"); // للمستخدمين النهائيين
    app.UseHsts();
}

// ...

// متحكم بسيط لمعالجة الأخطاء
// Controllers/ErrorController.cs
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ErrorController : ControllerBase
{
    [Route("/")]
    public IActionResult Error()
    {
        var context = HttpContext.Features.Get<IExceptionHandlerFeature>();
        var exception = context?.Error;

        // يمكنك تسجيل الخطأ هنا
        // _logger.LogError(exception, "An unhandled exception occurred.");

        return Problem(
            detail: exception?.Message,
            title: "An error occurred while processing your request.",
            statusCode: StatusCodes.Status500InternalServerError);
    }
}
```

### ب. Middleware مخصص لمعالجة الأخطاء

لتحكم أكبر، يمكنك إنشاء Middleware مخصص لمعالجة الأخطاء وتحويل الاستثناءات إلى استجابات HTTP مناسبة.

```csharp
// Middleware/ErrorHandlingMiddleware.cs
using Microsoft.AspNetCore.Http;
using System.Net;
using System.Text.Json;

public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;

    public ErrorHandlingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;

        var result = JsonSerializer.Serialize(new { message = "An unexpected error occurred.", detail = exception.Message });
        return context.Response.WriteAsync(result);
    }
}

// في Program.cs
// app.UseMiddleware<ErrorHandlingMiddleware>(); // يجب أن يكون في بداية الـ pipeline
```

### ج. Problem Details for HTTP APIs

ASP.NET Core يدعم معيار RFC 7807 (Problem Details for HTTP APIs) لتوفير تفاصيل موحدة للأخطاء في استجابات JSON. هذا مفيد بشكل خاص لواجهات برمجة التطبيقات.

```csharp
// مثال على ProblemDetails من ASP.NET Core
return Problem(
    detail: "تفاصيل إضافية عن الخطأ",
    title: "عنوان الخطأ",
    statusCode: StatusCodes.Status400BadRequest,
    type: "https://example.com/probs/out-of-credit", // URI لنوع الخطأ
    instance: HttpContext.Request.Path // URI للمورد الذي تسبب في الخطأ
);
```

## 3. تسجيل الأخطاء (Logging Errors)

بالإضافة إلى معالجة الأخطاء، من الضروري تسجيلها (Logging) لمراقبة صحة التطبيق وتصحيح الأخطاء. ASP.NET Core لديه نظام تسجيل مدمج يمكن تكوينه للعمل مع مزودي تسجيل مختلفين (مثل Console, Debug, Azure Application Insights, Serilog).

```csharp
// في أي فئة تحتاج إلى تسجيل
private readonly ILogger<MyClass> _logger;

public MyClass(ILogger<MyClass> logger)
{
    _logger = logger;
}

public void DoSomethingRisky()
{
    try
    {
        // ...
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "An error occurred in DoSomethingRisky.");
        throw; // إعادة رمي الاستثناء ليتم معالجته بواسطة الـ Middleware
    }
}
```

## الخلاصة

التحقق من صحة البيانات ومعالجة الأخطاء هما جزء لا يتجزأ من بناء تطبيقات ويب قوية وموثوقة. من خلال تطبيق Data Annotations أو Fluent Validation، واستخدام آليات معالجة الأخطاء المركزية مثل `UseExceptionHandler` أو Custom Middleware، وتسجيل الأخطاء بشكل فعال، يمكنك تحسين جودة تطبيقك وتجربة المستخدم بشكل كبير. في الدرس التالي، سنتناول التسجيل والمراقبة بشكل أعمق.
