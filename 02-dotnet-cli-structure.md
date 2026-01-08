# 02-dotnet-cli-structure.md

# هيكلية المشروع وأدوات CLI

## المقدمة

فهم هيكلية مشروع ASP.NET Core وكيفية استخدام أدوات سطر الأوامر (.NET CLI) أمر أساسي لإدارة المشاريع بكفاءة. يتيح لك .NET CLI إنشاء المشاريع، إدارتها، بنائها، ونشرها دون الحاجة إلى بيئة تطوير متكاملة (IDE) بالكامل، مما يجعله أداة قوية للمطورين.

## 1. هيكلية المشروع الأساسية

عند إنشاء مشروع ASP.NET Core جديد، يتم إنشاء مجموعة من الملفات والمجلدات الافتراضية. فهم الغرض من كل منها يساعد في تنظيم الكود وتحديد مكان إضافة الميزات الجديدة.

```
MyAspNetCoreProject/
├── .vscode/               # إعدادات Visual Studio Code (اختياري)
├── Controllers/           # المتحكمات (Controllers) التي تعالج طلبات HTTP
│   └── HomeController.cs
├── Models/                # نماذج البيانات (Data Models)
│   └── ErrorViewModel.cs
├── Views/                 # ملفات العرض (Views) لتطبيقات MVC/Razor Pages
│   ├── Home/
│   │   └── Index.cshtml
│   └── Shared/
│       └── _Layout.cshtml
├── wwwroot/               # الملفات الثابتة (Static Files) مثل CSS, JS, صور
│   ├── css/
│   ├── js/
│   └── lib/
├── appsettings.json       # ملفات إعدادات التطبيق الأساسية
├── appsettings.Development.json # إعدادات خاصة ببيئة التطوير
├── Program.cs             # نقطة الدخول الرئيسية للتطبيق (Startup)
├── MyAspNetCoreProject.csproj # ملف المشروع: تعريف التبعيات، إصدار .NET، إلخ.
├── Startup.cs             # (في الإصدارات القديمة) تهيئة الخدمات والـ Middleware
└── Properties/
    └── launchSettings.json # إعدادات تشغيل المشروع (منافذ، متغيرات بيئة)
```

**ملاحظات هامة:**

*   **`Program.cs`**: في .NET 6 وما بعده، تم دمج منطق `Startup.cs` و `Program.cs` في ملف واحد لتبسيط نقطة الدخول. هذا يستخدم ما يسمى بـ "Minimal APIs" و "Top-level statements".
*   **`wwwroot`**: هذا المجلد هو الجذر الافتراضي للملفات الثابتة التي يتم تقديمها مباشرة للمتصفح.
*   **`.csproj`**: ملف XML يصف المشروع، تبعياته، وإعدادات البناء. يمكنك تحريره يدوياً لإضافة حزم NuGet أو تغيير إعدادات البناء.

## 2. .NET Command Line Interface (CLI)

.NET CLI هي أداة قوية تسمح لك بالتفاعل مع مشاريع .NET من سطر الأوامر. إليك بعض الأوامر الأساسية التي ستحتاجها بشكل متكرر:

### الأوامر الأساسية لـ .NET CLI

| الأمر | الوصف | مثال |
| --- | --- | --- |
| `dotnet new` | إنشاء مشروع أو ملف جديد من قالب. | `dotnet new webapi -n MyApi` |
| `dotnet build` | بناء المشروع (تجميع الكود). | `dotnet build` |
| `dotnet run` | بناء وتشغيل المشروع. | `dotnet run` |
| `dotnet watch run` | بناء وتشغيل المشروع ومراقبته للتغييرات. | `dotnet watch run` |
| `dotnet publish` | تجميع التطبيق للنشر. | `dotnet publish -c Release -o app/publish` |
| `dotnet add package` | إضافة حزمة NuGet إلى المشروع. | `dotnet add package Microsoft.EntityFrameworkCore` |
| `dotnet remove package` | إزالة حزمة NuGet من المشروع. | `dotnet remove package Microsoft.EntityFrameworkCore` |
| `dotnet restore` | استعادة التبعيات (يتم تلقائياً مع `build` و `run`). | `dotnet restore` |
| `dotnet test` | تشغيل اختبارات الوحدة. | `dotnet test` |

### أمثلة عملية على استخدام CLI

#### إنشاء حل (Solution) ومشاريع متعددة

في المشاريع الكبيرة، من الشائع تنظيم الكود في حلول تحتوي على مشاريع متعددة (مثل مشروع للـ API، مشروع للمنطق التجاري، مشروع للبيانات).

```bash
# 1. إنشاء مجلد للحل
mkdir MySolution
cd MySolution

# 2. إنشاء ملف الحل
dotnet new sln -n MySolution

# 3. إنشاء مشروع Web API وإضافته للحل
mkdir MyApiProject
cd MyApiProject
dotnet new webapi -n MyApiProject
cd ..
dotnet sln add MyApiProject/MyApiProject.csproj

# 4. إنشاء مشروع مكتبة فئات (Class Library) وإضافته للحل
mkdir MyBusinessLogic
cd MyBusinessLogic
dotnet new classlib -n MyBusinessLogic
cd ..
dotnet sln add MyBusinessLogic/MyBusinessLogic.csproj

# 5. إضافة مرجع من مشروع API إلى مشروع المكتبة
dotnet add MyApiProject/MyApiProject.csproj reference MyBusinessLogic/MyBusinessLogic.csproj

# 6. بناء الحل بالكامل
dotnet build
```

#### نشر التطبيق (Publishing)

لنشر تطبيقك، يمكنك استخدام الأمر `dotnet publish`. هذا الأمر يقوم بتجميع كل ما يلزم لتشغيل تطبيقك في مجلد واحد.

```bash
# نشر التطبيق في وضع Release إلى مجلد 'publish'
dotnet publish -c Release -o ./publish

# تشغيل التطبيق المنشور
dotnet ./publish/MyApiProject.dll
```

## 3. ملف `launchSettings.json`

يوجد هذا الملف في مجلد `Properties` ويحتوي على إعدادات خاصة بكيفية تشغيل مشروعك في بيئات مختلفة (مثل IIS Express، أو مباشرة من التطبيق). يمكنك تحديد منافذ HTTP/HTTPS، متغيرات البيئة، وأوامر التشغيل هنا.

```json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

**نقاط رئيسية:**

*   `applicationUrl`: يحدد المنافذ التي سيستمع إليها التطبيق.
*   `environmentVariables`: يسمح بتحديد متغيرات البيئة الخاصة بملف التعريف (Profile) هذا، مثل `ASPNETCORE_ENVIRONMENT` الذي يحدد بيئة التشغيل (Development, Staging, Production).
*   `launchUrl`: يحدد المسار الذي سيتم فتحه في المتصفح عند تشغيل التطبيق (غالباً ما يكون `swagger` لواجهات برمجة التطبيقات).

## الخلاصة

إتقان .NET CLI وفهم هيكلية المشروع يمنحك تحكماً كاملاً في عملية تطوير ASP.NET Core. هذه الأدوات ضرورية لأتمتة المهام، إدارة التبعيات، ونشر التطبيقات بكفاءة. في الدرس التالي، سنتعمق في كيفية بناء واجهات برمجة التطبيقات (Web APIs) باستخدام التوجيه والمتحكمات.
