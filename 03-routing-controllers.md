# 03-routing-controllers.md

# التوجيه والمتحكمات (Routing & Controllers)

## المقدمة

في ASP.NET Core Web API، يعتبر التوجيه (Routing) والمتحكمات (Controllers) العمود الفقري لكيفية معالجة الطلبات الواردة من العملاء (Clients) وإرجاع الاستجابات. التوجيه يحدد كيف يتم مطابقة عناوين URL مع الأكواد البرمجية المناسبة، بينما المتحكمات هي الفئات التي تحتوي على المنطق الفعلي لمعالجة هذه الطلبات.

## 1. المتحكمات (Controllers)

المتحكم هو فئة عامة (Public Class) تُستخدم لمعالجة طلبات HTTP. يجب أن ترث هذه الفئة من `ControllerBase` (لـ Web API) أو `Controller` (لـ MVC). تحتوي المتحكمات على دوال (Actions) عامة تُعرف باسم Action Methods، والتي يتم استدعاؤها استجابة لطلبات HTTP.

### مثال على متحكم بسيط

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;
using System.Linq;

namespace MyApiProject.Controllers
{
    [ApiController] // يشير إلى أن هذا المتحكم يستخدم لـ API
    [Route("api/[controller]")] // يحدد قالب التوجيه لهذا المتحكم
    public class ProductsController : ControllerBase
    {
        private static readonly List<string> Products = new List<string>
        {
            "Laptop", "Mouse", "Keyboard", "Monitor"
        };

        // GET api/products
        [HttpGet]
        public ActionResult<IEnumerable<string>> Get()
        {
            return Ok(Products);
        }

        // GET api/products/5
        [HttpGet("{id}")]
        public ActionResult<string> Get(int id)
        {
            if (id < 0 || id >= Products.Count)
            {
                return NotFound(); // 404 Not Found
            }
            return Ok(Products[id]);
        }

        // POST api/products
        [HttpPost]
        public ActionResult Post([FromBody] string newProduct)
        {
            Products.Add(newProduct);
            return CreatedAtAction(nameof(Get), new { id = Products.Count - 1 }, newProduct); // 201 Created
        }

        // PUT api/products/5
        [HttpPut("{id}")]
        public ActionResult Put(int id, [FromBody] string updatedProduct)
        {
            if (id < 0 || id >= Products.Count)
            {
                return NotFound();
            }
            Products[id] = updatedProduct;
            return NoContent(); // 204 No Content
        }

        // DELETE api/products/5
        [HttpDelete("{id}")]
        public ActionResult Delete(int id)
        {
            if (id < 0 || id >= Products.Count)
            {
                return NotFound();
            }
            Products.RemoveAt(id);
            return NoContent();
        }
    }
}
```

**شرح السمات (Attributes):**

*   `[ApiController]`: يضيف سلوكيات خاصة بالـ API مثل التحقق التلقائي من صحة النموذج (Model Validation) واستنتاج مصادر الربط (Binding Sources).
*   `[Route("api/[controller]")]`: يحدد قالب التوجيه للمتحكم. `[controller]` هو Placeholder يتم استبداله باسم المتحكم (بدون لاحقة `Controller`). في هذا المثال، سيكون المسار الأساسي هو `api/products`.
*   `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`: تحدد نوع طلب HTTP الذي تستجيب له الدالة. يمكن أيضاً تحديد قالب مسار إضافي داخل هذه السمات، مثل `[HttpGet("{id}")]`.
*   `[FromBody]`: يشير إلى أن القيمة يجب أن تأتي من جسم الطلب (Request Body).

## 2. التوجيه (Routing)

التوجيه هو عملية مطابقة عنوان URL لطلب HTTP وارد مع Action Method محدد في المتحكم. في ASP.NET Core، يتم التوجيه عادةً باستخدام سمات (Attributes) مباشرة على المتحكمات والدوال.

### أنواع التوجيه

1.  **Attribute Routing (التوجيه بالسمات):** هو الأسلوب المفضل والأكثر شيوعاً في Web API. يتم تحديد المسارات مباشرة فوق المتحكمات والدوال باستخدام سمات مثل `[Route]`, `[HttpGet]`, إلخ.

    ```csharp
    [Route("api/[controller]")] // قالب على مستوى المتحكم
    public class UsersController : ControllerBase
    {
        [HttpGet] // GET api/users
        public IActionResult GetAllUsers() { ... }

        [HttpGet("{id}")] // GET api/users/{id}
        public IActionResult GetUserById(int id) { ... }

        [HttpPost("register")] // POST api/users/register
        public IActionResult RegisterUser([FromBody] User user) { ... }
    }
    ```

2.  **Conventional Routing (التوجيه التقليدي):** يستخدم بشكل أساسي في تطبيقات MVC حيث يتم تعريف قوالب المسارات في `Program.cs` (أو `Startup.cs` سابقاً). يمكن استخدامه أيضاً في Web API ولكن Attribute Routing أكثر مرونة ووضوحاً.

    ```csharp
    // في Program.cs
    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");
    
    // أو لـ API فقط
    app.MapControllers(); // يستخدم Attribute Routing بشكل افتراضي
    ```

### ربط المعلمات (Parameter Binding)

ASP.NET Core يقوم تلقائياً بربط قيم من الطلب (مثل المسار، الاستعلام، الجسم) إلى معلمات Action Method. إليك كيفية عمله:

| المصدر | الوصف | مثال |
| --- | --- | --- |
| **المسار (Route)** | من أجزاء عنوان URL. | `[HttpGet("{id}")] public ActionResult Get(int id)` |
| **الاستعلام (Query)** | من سلسلة الاستعلام (Query String). | `public ActionResult Get([FromQuery] string name)` |
| **الرأس (Header)** | من رؤوس طلب HTTP. | `public ActionResult Get([FromHeader] string userAgent)` |
| **الجسم (Body)** | من جسم طلب HTTP (لـ POST, PUT). | `public ActionResult Post([FromBody] Product product)` |
| **الخدمات (Services)** | من حقن التبعية (DI). | `public ActionResult Get([FromServices] IProductService service)` |

**ملاحظات:**

*   إذا لم يتم تحديد سمة ربط المصدر، يحاول ASP.NET Core استنتاج المصدر بناءً على نوع المعلمة وموقعها.
*   لأنواع القيم البسيطة (int, string, bool)، يتم البحث في المسار ثم الاستعلام ثم الرأس.
*   لأنواع الكائنات المعقدة، يتم البحث في الجسم (Body) افتراضياً.

## 3. أنواع الاستجابات (Action Results)

Action Methods في المتحكمات تُرجع عادةً كائنات من نوع `IActionResult` أو `ActionResult<T>`. هذه الأنواع توفر مرونة في إرجاع استجابات HTTP مختلفة (مثل 200 OK, 201 Created, 404 Not Found, 400 Bad Request).

### أمثلة على `IActionResult`

| نوع الاستجابة | الوصف | كود الحالة | مثال |
| --- | --- | --- | --- |
| `Ok()` | نجاح مع محتوى. | 200 OK | `return Ok(data);` |
| `NotFound()` | المورد غير موجود. | 404 Not Found | `return NotFound();` |
| `BadRequest()` | طلب غير صالح. | 400 Bad Request | `return BadRequest("Invalid input");` |
| `NoContent()` | نجاح بدون محتوى. | 204 No Content | `return NoContent();` |
| `CreatedAtAction()` | تم إنشاء مورد جديد. | 201 Created | `return CreatedAtAction(nameof(Get), new { id = newId }, newItem);` |
| `Unauthorized()` | غير مصرح به. | 401 Unauthorized | `return Unauthorized();` |
| `Forbid()` | ممنوع (لعدم وجود صلاحيات). | 403 Forbidden | `return Forbid();` |

### `ActionResult<T>`

تم تقديمه في ASP.NET Core 2.1 لتوفير مرونة أكبر. يمكن أن يُرجع إما نوع `T` مباشرة (وسيتم تحويله إلى 200 OK) أو أي نوع من `IActionResult`.

```csharp
[HttpGet("{id}")]
public ActionResult<Product> GetProduct(int id)
{
    var product = _productService.GetProductById(id);
    if (product == null)
    {
        return NotFound(); // يرجع 404 Not Found
    }
    return product; // يرجع 200 OK مع كائن المنتج
}
```

## الخلاصة

التوجيه والمتحكمات هما الأساس الذي تبنى عليه واجهات برمجة التطبيقات في ASP.NET Core. إتقان كيفية تعريف المسارات، ربط المعلمات، وإرجاع الاستجابات الصحيحة أمر بالغ الأهمية لبناء API فعال وقابل للصيانة. في الدرس التالي، سنتعمق في مفهوم حقن التبعية (Dependency Injection) وكيفية استخدامه لتصميم تطبيقات مرنة وقابلة للاختبار.
