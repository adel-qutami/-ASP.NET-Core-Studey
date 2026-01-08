# 07-authentication-authorization.md

# الهوية، JWT، وصلاحيات الوصول

## المقدمة

الأمان هو جانب حاسم في أي تطبيق ويب. في ASP.NET Core، يتم التعامل مع أمان المستخدمين بشكل أساسي من خلال مفهومين رئيسيين: **المصادقة (Authentication)** و**التفويض (Authorization)**. المصادقة هي عملية التحقق من هوية المستخدم (من أنت؟)، بينما التفويض هو عملية تحديد ما إذا كان المستخدم المصادق عليه لديه الإذن للقيام بإجراء معين (ماذا يمكنك أن تفعل؟).

## 1. المصادقة (Authentication)

المصادقة هي عملية التحقق من أن المستخدم هو من يدعي أنه هو. في ASP.NET Core، يتم ذلك عادةً باستخدام مخططات المصادقة (Authentication Schemes).

### أنواع المصادقة الشائعة في Web API

*   **JWT Bearer Authentication:** الأكثر شيوعاً لواجهات برمجة التطبيقات (APIs) الحديثة. يعتمد على إرسال رمز مميز (Token) في كل طلب بعد تسجيل الدخول الأولي.
*   **API Key Authentication:** يستخدم مفتاح API سري يتم إرساله مع كل طلب. أبسط ولكنه أقل أماناً من JWT في معظم الحالات.

### إعداد JWT Bearer Authentication

1.  **تثبيت حزم NuGet:**

    ```bash
    dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
    ```

2.  **تكوين JWT في `Program.cs`:**

    ```csharp
    using Microsoft.AspNetCore.Authentication.JwtBearer;
    using Microsoft.IdentityModel.Tokens;
    using System.Text;

    var builder = WebApplication.CreateBuilder(args);

    // ... إضافة خدمات أخرى ...

    builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = builder.Configuration["Jwt:Issuer"],
                ValidAudience = builder.Configuration["Jwt:Audience"],
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
            };
        });

    builder.Services.AddAuthorization(); // إضافة خدمة التفويض

    var app = builder.Build();

    // ... تكوين الـ Middleware ...

    // يجب أن يأتي بعد UseRouting وقبل MapControllers
    app.UseAuthentication();
    app.UseAuthorization();

    app.MapControllers();

    app.Run();
    ```

3.  **إضافة إعدادات JWT في `appsettings.json`:**

    ```json
    {
      "Jwt": {
        "Key": "YourSuperSecretKeyWhichShouldBeAtLeast32CharactersLong",
        "Issuer": "YourIssuerDomain.com",
        "Audience": "YourAudienceDomain.com"
      },
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*"
    }
    ```

### إنشاء رمز JWT (Token Generation)

عادةً ما يتم إنشاء الرمز المميز بعد تسجيل دخول المستخدم بنجاح. إليك مثال على دالة لإنشاء رمز JWT:

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.IdentityModel.Tokens;
using System.Text;

public class JwtTokenGenerator
{
    private readonly IConfiguration _configuration;

    public JwtTokenGenerator(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GenerateToken(string userId, string username, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, userId),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(ClaimTypes.Name, username)
        };

        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var expires = DateTime.Now.AddDays(Convert.ToDouble(_configuration["Jwt:ExpireDays"] ?? "7"));

        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: expires,
            signingCredentials: creds
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

## 2. التفويض (Authorization)

التفويض هو عملية تحديد ما إذا كان المستخدم المصادق عليه لديه الأذونات اللازمة للوصول إلى مورد أو تنفيذ إجراء معين. في ASP.NET Core، يتم التفويض باستخدام سمات (Attributes) أو سياسات (Policies).

### أ. التفويض بالسمات (Role-based Authorization)

يمكنك تقييد الوصول إلى المتحكمات أو الدوال بناءً على أدوار المستخدمين باستخدام السمة `[Authorize]`. 

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace MyApiProject.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [Authorize] // يتطلب مصادقة للوصول إلى أي دالة في هذا المتحكم
    public class AdminController : ControllerBase
    {
        // يمكن الوصول إليها فقط من قبل المستخدمين المصادق عليهم
        [HttpGet("dashboard")]
        public IActionResult GetAdminDashboard() { return Ok("Admin Dashboard Data"); }

        [HttpGet("users")]
        [Authorize(Roles = "Administrator,Manager")] // يتطلب دور Administrator أو Manager
        public IActionResult GetUsers() { return Ok("List of Users"); }

        [HttpGet("reports")]
        [Authorize(Roles = "Administrator")] // يتطلب دور Administrator فقط
        public IActionResult GetReports() { return Ok("Sensitive Reports"); }
    }

    [ApiController]
    [Route("api/[controller]")]
    public class PublicController : ControllerBase
    {
        [HttpGet("data")]
        [AllowAnonymous] // يسمح بالوصول بدون مصادقة (يتجاوز [Authorize] على مستوى المتحكم إن وجد)
        public IActionResult GetPublicData() { return Ok("Public Data"); }
    }
}
```

### ب. التفويض بالسياسات (Policy-based Authorization)

تسمح لك السياسات بتعريف قواعد تفويض أكثر تعقيداً وقابلية لإعادة الاستخدام. يمكن أن تعتمد السياسات على أدوار، مطالبات (Claims)، أو متطلبات مخصصة.

1.  **تعريف السياسة في `Program.cs`:**

    ```csharp
    // في Program.cs
    builder.Services.AddAuthorization(options =>
    {
        options.AddPolicy("RequireAdminRole", policy => policy.RequireRole("Administrator"));
        options.AddPolicy("RequireManagerOrAdmin", policy =>
            policy.RequireAssertion(context =>
                context.User.IsInRole("Administrator") ||
                context.User.IsInRole("Manager")));
        options.AddPolicy("RequireAgeOver21", policy =>
            policy.RequireClaim("DateOfBirth", claim =>
            {
                var dob = Convert.ToDateTime(claim.Value);
                return dob.AddYears(21) <= DateTime.Today;
            }));
    });
    ```

2.  **استخدام السياسة في المتحكمات:**

    ```csharp
    [ApiController]
    [Route("api/[controller]")]
    public class PolicyController : ControllerBase
    {
        [HttpGet("admin-only")]
        [Authorize(Policy = "RequireAdminRole")]
        public IActionResult GetAdminData() { return Ok("Admin Data via Policy"); }

        [HttpGet("manager-or-admin")]
        [Authorize(Policy = "RequireManagerOrAdmin")]
        public IActionResult GetManagerOrAdminData() { return Ok("Manager or Admin Data"); }

        [HttpGet("age-restricted")]
        [Authorize(Policy = "RequireAgeOver21")]
        public IActionResult GetAgeRestrictedData() { return Ok("Content for over 21s"); }
    }
    ```

## 3. ASP.NET Core Identity

ASP.NET Core Identity هو نظام عضوية كامل الميزات يوفر واجهة برمجة تطبيقات لإدارة المستخدمين، كلمات المرور، الأدوار، والمطالبات. إنه مثالي لتطبيقات الويب التي تتطلب إدارة مستخدمين كاملة، ويمكن دمجه مع JWT للمصادقة.

### ميزات Identity:

*   إدارة المستخدمين (تسجيل، تسجيل دخول، تغيير كلمة مرور).
*   إدارة الأدوار (Roles).
*   دعم المصادقة متعددة العوامل (Multi-factor authentication).
*   دعم لمقدمي تسجيل الدخول الخارجيين (مثل Google, Facebook).

### إعداد ASP.NET Core Identity (مختصر)

1.  **تثبيت حزم NuGet:**

    ```bash
    dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
    dotnet add package Microsoft.AspNetCore.Identity.UI
    ```

2.  **تكوين Identity في `Program.cs`:**

    ```csharp
    // في Program.cs
    builder.Services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(connectionString));

    builder.Services.AddDefaultIdentity<IdentityUser>(options => options.SignIn.RequireConfirmedAccount = true)
        .AddRoles<IdentityRole>() // لإضافة دعم الأدوار
        .AddEntityFrameworkStores<ApplicationDbContext>();

    // ... ثم استخدم UseAuthentication و UseAuthorization كما هو موضح أعلاه
    ```

## الخلاصة

الأمان هو جانب لا يمكن التهاون به في تطوير الويب. ASP.NET Core يوفر أدوات قوية ومرنة للتعامل مع المصادقة والتفويض، سواء كنت تستخدم JWT لـ APIs أو ASP.NET Core Identity لتطبيقات الويب الكاملة. فهم هذه المفاهيم وتطبيقها بشكل صحيح سيضمن حماية تطبيقك وبيانات المستخدمين. في الدرس التالي، سنتناول كيفية التحقق من صحة البيانات ومعالجة الأخطاء بشكل فعال.
