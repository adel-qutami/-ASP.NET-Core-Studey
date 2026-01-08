# 10-cashing-strategies.md

# استراتيجيات التخزين المؤقت (In-Memory & Redis)

## المقدمة

التخزين المؤقت (Caching) هو أسلوب أساسي لتحسين أداء تطبيقات الويب وتقليل الحمل على الموارد الخلفية (مثل قواعد البيانات والخدمات الخارجية). من خلال تخزين البيانات التي يتم الوصول إليها بشكل متكرر في الذاكرة أو في مخزن بيانات سريع، يمكن للتطبيق تقديم الاستجابات بشكل أسرع بكثير. ASP.NET Core يوفر دعماً مدمجاً لأنواع مختلفة من التخزين المؤقت.

## 1. التخزين المؤقت في الذاكرة (In-Memory Caching)

التخزين المؤقت في الذاكرة هو أبسط أنواع التخزين المؤقت وأكثرها شيوعاً. يتم تخزين البيانات في ذاكرة الخادم الذي يستضيف التطبيق. إنه فعال جداً للبيانات التي يتم الوصول إليها بشكل متكرر ولا تتغير كثيراً.

### أ. إعداد In-Memory Caching

يتم إضافة خدمة التخزين المؤقت في الذاكرة إلى حاوية DI في `Program.cs`:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// ... إضافة خدمات أخرى ...

builder.Services.AddMemoryCache(); // إضافة خدمة التخزين المؤقت في الذاكرة

var app = builder.Build();
// ...
```

### ب. استخدام `IMemoryCache`

يمكنك حقن `IMemoryCache` في المتحكمات أو الخدمات لاستخدامه:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Memory;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMemoryCache _memoryCache;
    private readonly IProductService _productService; // خدمة لجلب المنتجات من قاعدة البيانات

    public ProductsController(IMemoryCache memoryCache, IProductService productService)
    {
        _memoryCache = memoryCache;
        _productService = productService;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        const string cacheKey = "AllProducts";

        // محاولة الحصول على المنتجات من الذاكرة المؤقتة
        if (_memoryCache.TryGetValue(cacheKey, out IEnumerable<Product>? products))
        {
            return Ok(products);
        }

        // إذا لم تكن موجودة، جلبها من الخدمة وتخزينها مؤقتاً
        products = await _productService.GetAllProductsAsync();

        var cacheEntryOptions = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromSeconds(60)) // إزالة بعد 60 ثانية من عدم الاستخدام
            .SetAbsoluteExpiration(TimeSpan.FromMinutes(10)); // إزالة بعد 10 دقائق كحد أقصى

        _memoryCache.Set(cacheKey, products, cacheEntryOptions);

        return Ok(products);
    }
}
```

**خيارات `MemoryCacheEntryOptions`:**

*   `SetAbsoluteExpiration`: يحدد وقتاً مطلقاً لإزالة العنصر من الذاكرة المؤقتة، بغض النظر عن النشاط.
*   `SetSlidingExpiration`: يحدد وقتاً لإزالة العنصر إذا لم يتم الوصول إليه خلال تلك الفترة. إذا تم الوصول إليه، يتم إعادة تعيين المؤقت.
*   `SetPriority`: يحدد أولوية العنصر عند إزالة العناصر بسبب ضغط الذاكرة.

## 2. التخزين المؤقت الموزع (Distributed Caching)

التخزين المؤقت في الذاكرة مناسب للتطبيقات التي تعمل على خادم واحد. ولكن في بيئات موزعة (مثل المزارع الخادمية أو Kubernetes)، حيث يمكن أن يكون هناك عدة نسخ من التطبيق، نحتاج إلى تخزين مؤقت موزع لضمان أن جميع النسخ ترى نفس البيانات المخزنة مؤقتاً. Redis هو خيار شائع جداً للتخزين المؤقت الموزع.

### أ. إعداد Distributed Caching (باستخدام Redis)

1.  **تثبيت حزمة NuGet:**

    ```bash
    dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
    ```

2.  **تكوين Redis في `Program.cs`:**

    ```csharp
    // Program.cs
    var builder = WebApplication.CreateBuilder(args);

    // ... إضافة خدمات أخرى ...

    builder.Services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = builder.Configuration.GetConnectionString("RedisConnection");
        options.InstanceName = "MyApiCache_"; // بادئة للمفاتيح في Redis
    });

    var app = builder.Build();
    // ...
    ```

3.  **إضافة سلسلة اتصال Redis في `appsettings.json`:**

    ```json
    {
      "ConnectionStrings": {
        "DefaultConnection": "...",
        "RedisConnection": "localhost:6379" // أو عنوان خادم Redis الخاص بك
      },
      // ...
    }
    ```

### ب. استخدام `IDistributedCache`

يمكنك حقن `IDistributedCache` في المتحكمات أو الخدمات لاستخدامه. يتم التعامل مع البيانات كسلاسل بايت، لذا ستحتاج إلى تسلسلها (Serialize) وإلغاء تسلسلها (Deserialize).

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IDistributedCache _distributedCache;
    private readonly IProductService _productService;

    public ProductsController(IDistributedCache distributedCache, IProductService productService)
    {
        _distributedCache = distributedCache;
        _productService = productService;
    }

    [HttpGet("distributed")]
    public async Task<ActionResult<IEnumerable<Product>>> GetProductsDistributed()
    {
        const string cacheKey = "AllProductsDistributed";

        // محاولة الحصول على المنتجات من الذاكرة المؤقتة الموزعة
        string? cachedProductsJson = await _distributedCache.GetStringAsync(cacheKey);
        if (!string.IsNullOrEmpty(cachedProductsJson))
        {
            var products = JsonSerializer.Deserialize<IEnumerable<Product>>(cachedProductsJson);
            return Ok(products);
        }

        // إذا لم تكن موجودة، جلبها من الخدمة وتخزينها مؤقتاً
        var productsFromDb = await _productService.GetAllProductsAsync();
        var productsJson = JsonSerializer.Serialize(productsFromDb);

        var cacheEntryOptions = new DistributedCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromSeconds(60)) // إزالة بعد 60 ثانية من عدم الاستخدام
            .SetAbsoluteExpiration(TimeSpan.Minutes(10)); // إزالة بعد 10 دقائق كحد أقصى

        await _distributedCache.SetStringAsync(cacheKey, productsJson, cacheEntryOptions);

        return Ok(productsFromDb);
    }
}
```

## 3. التخزين المؤقت للاستجابة (Response Caching)

ASP.NET Core يوفر أيضاً Middleware للتخزين المؤقت للاستجابة، والذي يمكنه تخزين استجابات HTTP كاملة. هذا مفيد جداً لنقاط النهاية التي تُرجع نفس البيانات لجميع المستخدمين ولا تتغير كثيراً.

### أ. إعداد Response Caching

1.  **إضافة خدمة Response Caching في `Program.cs`:**

    ```csharp
    // Program.cs
    var builder = WebApplication.CreateBuilder(args);

    // ... إضافة خدمات أخرى ...

    builder.Services.AddResponseCaching();

    var app = builder.Build();

    // ... تكوين الـ Middleware ...

    app.UseResponseCaching(); // يجب أن يأتي قبل UseRouting

    app.UseRouting();
    // ...
    ```

2.  **استخدام السمة `[ResponseCache]`:**

    ```csharp
    using Microsoft.AspNetCore.Mvc;

    [ApiController]
    [Route("api/[controller]")]
    public class CachedDataController : ControllerBase
    {
        [HttpGet]
        [ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any, NoStore = false)]
        public IActionResult GetCachedData()
        {
            // هذه الاستجابة سيتم تخزينها مؤقتاً لمدة 60 ثانية
            return Ok($"This data was generated at {DateTime.Now}");
        }

        [HttpGet("private")]
        [ResponseCache(Duration = 30, Location = ResponseCacheLocation.Client, NoStore = false)]
        public IActionResult GetPrivateCachedData()
        {
            // هذه الاستجابة سيتم تخزينها مؤقتاً في متصفح العميل فقط
            return Ok($"Private data generated at {DateTime.Now}");
        }
    }
    ```

**خيارات `ResponseCache`:**

*   `Duration`: المدة بالثواني التي يجب أن يتم فيها تخزين الاستجابة مؤقتاً.
*   `Location`: يحدد أين يمكن تخزين الاستجابة مؤقتاً (Any, Client, None).
*   `NoStore`: يمنع تخزين الاستجابة مؤقتاً.

## 4. اعتبارات هامة عند استخدام Caching

*   **Invalidation (إبطال التخزين المؤقت):** كيف ستتعامل مع البيانات التي تتغير؟ يجب أن يكون لديك استراتيجية لإبطال البيانات المخزنة مؤقتاً عند تحديث البيانات الأصلية.
*   **Cache Stampede:** عندما تنتهي صلاحية عنصر مخزن مؤقتاً ويحاول العديد من الطلبات إعادة تحميله في نفس الوقت، مما يضع حملاً كبيراً على الموارد الخلفية.
*   **Consistency (الاتساق):** هل تحتاج البيانات المخزنة مؤقتاً إلى أن تكون متسقة تماماً مع البيانات الأصلية؟ قد يكون هناك تأخير بسيط في التحديث.
*   **Memory Usage:** التخزين المؤقت في الذاكرة يستهلك ذاكرة الخادم. يجب مراقبة استخدام الذاكرة لتجنب مشاكل الأداء.

## الخلاصة

التخزين المؤقت هو أداة قوية لتحسين أداء تطبيقات ASP.NET Core. سواء كنت تستخدم التخزين المؤقت في الذاكرة للبيانات المحلية، أو التخزين المؤقت الموزع مثل Redis للبيئات الموزعة، أو التخزين المؤقت للاستجابة لتقليل الحمل على الخادم، فإن فهم هذه الاستراتيجيات وتطبيقها بشكل صحيح سيؤدي إلى تطبيقات أسرع وأكثر استجابة. في الدرس التالي، سنتناول كيفية تنفيذ المهام الخلفية (Background Tasks) في ASP.NET Core.
