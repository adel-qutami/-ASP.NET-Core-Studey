# 04-dependency-injection.md

# حقن التبعية (Dependency Injection) وإدارة الخدمات

## المقدمة

حقن التبعية (Dependency Injection - DI) هو نمط تصميم برمجي (Design Pattern) يهدف إلى تقليل الاقتران (Coupling) بين مكونات التطبيق وزيادة قابلية الاختبار (Testability) والصيانة (Maintainability). ASP.NET Core مبني بشكل أساسي على DI، ويوفر حاوية DI مدمجة (Built-in DI Container) لإدارة دورة حياة الخدمات (Services) وحقنها تلقائياً عند الحاجة.

## 1. ما هو حقن التبعية؟

بدلاً من أن تقوم الفئة بإنشاء تبعياتها (Dependencies) بنفسها، يتم توفير هذه التبعيات لها من الخارج. هذا يعني أن الفئة لا تعرف كيف يتم إنشاء تبعياتها، بل تعرف فقط كيفية استخدامها. هذا الفصل بين الاهتمامات (Separation of Concerns) يؤدي إلى كود أكثر مرونة.

### مثال بدون DI

```csharp
public class DataRepository
{
    public void SaveData(string data) { /* ... */ }
}

public class BusinessLogic
{
    private DataRepository _repository;

    public BusinessLogic()
    {
        _repository = new DataRepository(); // الفئة تنشئ تبعيتها بنفسها
    }

    public void ProcessData(string data)
    {
        // ... منطق العمل ...
        _repository.SaveData(data);
    }
}
```

في هذا المثال، `BusinessLogic` مرتبطة بشكل وثيق بـ `DataRepository`. إذا أردنا تغيير `DataRepository` أو اختبار `BusinessLogic` بشكل منفصل، سيكون الأمر صعباً.

### مثال مع DI (Constructor Injection)

```csharp
public interface IDataRepository
{
    void SaveData(string data);
}

public class DataRepository : IDataRepository
{
    public void SaveData(string data) { /* ... */ }
}

public class BusinessLogic
{
    private IDataRepository _repository;

    public BusinessLogic(IDataRepository repository) // التبعية يتم حقنها عبر الـ Constructor
    {
        _repository = repository;
    }

    public void ProcessData(string data)
    {
        // ... منطق العمل ...
        _repository.SaveData(data);
    }
}
```

هنا، `BusinessLogic` تعتمد على واجهة `IDataRepository` بدلاً من تطبيق معين. يتم تمرير `IDataRepository` إلى Constructor، مما يجعل `BusinessLogic` قابلة للاختبار بسهولة (يمكن تمرير Mock Object) وأكثر مرونة.

## 2. حاوية حقن التبعية في ASP.NET Core

ASP.NET Core يوفر حاوية DI مدمجة تُعرف باسم Service Provider. يتم تسجيل الخدمات (Implementations of Interfaces) في هذه الحاوية، ثم تقوم الحاوية بإنشاء الكائنات وحقنها تلقائياً عند الحاجة.

يتم تسجيل الخدمات عادةً في ملف `Program.cs` (أو `Startup.cs` في الإصدارات القديمة) باستخدام كائن `IServiceCollection`.

### أنواع دورة حياة الخدمات (Service Lifetimes)

عند تسجيل خدمة، يجب تحديد دورة حياتها، والتي تحدد متى يتم إنشاء مثيل جديد من الخدمة ومتى يتم التخلص منه.

| نوع دورة الحياة | الوصف | متى يتم إنشاء المثيل؟ |
| --- | --- | --- |
| **Singleton** | يتم إنشاء مثيل واحد فقط للخدمة طوال عمر التطبيق بأكمله. | عند أول طلب للخدمة، أو عند بدء التطبيق إذا تم تحديد ذلك. |
| **Scoped** | يتم إنشاء مثيل واحد للخدمة لكل طلب HTTP. | لكل طلب HTTP جديد. |
| **Transient** | يتم إنشاء مثيل جديد للخدمة في كل مرة يتم فيها طلبها. | في كل مرة يتم فيها طلب الخدمة. |

### تسجيل الخدمات في `Program.cs`

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

// إضافة الخدمات إلى الحاوية
builder.Services.AddControllers();
// تسجيل خدمة Singleton
builder.Services.AddSingleton<IDataRepository, DataRepository>();
// تسجيل خدمة Scoped
builder.Services.AddScoped<IBusinessLogic, BusinessLogic>();
// تسجيل خدمة Transient
builder.Services.AddTransient<IOperationService, OperationService>();

var app = builder.Build();

// ... تكوين الـ Middleware ...

app.MapControllers();

app.Run();
```

**شرح:**

*   `AddSingleton<TInterface, TImplementation>()`: يسجل `DataRepository` كتطبيق لـ `IDataRepository`، وسيتم إنشاء مثيل واحد فقط منه.
*   `AddScoped<TInterface, TImplementation>()`: يسجل `BusinessLogic` كتطبيق لـ `IBusinessLogic`، وسيتم إنشاء مثيل جديد لكل طلب HTTP.
*   `AddTransient<TInterface, TImplementation>()`: يسجل `OperationService` كتطبيق لـ `IOperationService`، وسيتم إنشاء مثيل جديد في كل مرة يتم فيها طلب الخدمة.

## 3. استخدام الخدمات المحقونة (Consuming Injected Services)

بمجرد تسجيل الخدمات في حاوية DI، يمكنك طلبها في Constructors للفئات الأخرى (مثل المتحكمات، الخدمات الأخرى، أو الـ Middleware) وسيتم حقنها تلقائياً.

### حقن التبعية في المتحكمات (Controller Injection)

```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyApiProject.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class MyController : ControllerBase
    {
        private readonly IBusinessLogic _businessLogic;
        private readonly IDataRepository _dataRepository;

        // يتم حقن التبعيات تلقائياً هنا
        public MyController(IBusinessLogic businessLogic, IDataRepository dataRepository)
        {
            _businessLogic = businessLogic;
            _dataRepository = dataRepository;
        }

        [HttpGet]
        public IActionResult Get()
        {
            _businessLogic.ProcessData("some data");
            var data = _dataRepository.GetData(); // مثال
            return Ok(data);
        }
    }
}
```

### حقن التبعية في الخدمات الأخرى

يمكنك أيضاً حقن الخدمات في خدمات أخرى، مما يسمح ببناء سلسلة من التبعيات المعقدة.

```csharp
public class AnotherService : IAnotherService
{
    private readonly IOperationService _operationService;

    public AnotherService(IOperationService operationService)
    {
        _operationService = operationService;
    }

    public void DoSomething()
    {
        _operationService.Execute();
    }
}
```

## 4. مزايا حقن التبعية

*   **قابلية الاختبار (Testability):** يسهل اختبار الفئات بشكل منفصل عن طريق تمرير Mock Objects للتبعيات.
*   **قابلية الصيانة (Maintainability):** يقلل من الاقتران، مما يجعل تغيير أو استبدال التبعيات أسهل.
*   **قابلية التوسع (Extensibility):** يسهل إضافة وظائف جديدة أو تغيير السلوك دون تعديل الفئات التي تعتمد عليها.
*   **إعادة استخدام الكود (Code Reusability):** يمكن إعادة استخدام الخدمات في أجزاء مختلفة من التطبيق.

## الخلاصة

حقن التبعية هو مفهوم أساسي في ASP.NET Core يجب على كل مطور إتقانه. فهم كيفية تسجيل الخدمات، اختيار دورة الحياة المناسبة، وحقن التبعيات سيساعدك على بناء تطبيقات قوية، مرنة، وقابلة للصيانة. في الدرس التالي، سنتناول مفهوم الـ Middleware وكيفية بناء Request Pipeline في ASP.NET Core.
