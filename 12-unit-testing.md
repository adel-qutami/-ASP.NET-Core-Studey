# 12-unit-testing.md

# اختبارات الوحدة (Unit Testing) في ASP.NET Core

## المقدمة

اختبارات الوحدة (Unit Tests) هي جزء أساسي من عملية تطوير البرمجيات الحديثة. تهدف إلى اختبار أصغر وحدة قابلة للاختبار في التطبيق (عادةً ما تكون دالة أو فئة) بشكل منفصل عن بقية النظام. تساعد اختبارات الوحدة في ضمان صحة الكود، اكتشاف الأخطاء مبكراً، وتوفير الثقة عند إجراء تغييرات أو إعادة هيكلة (Refactoring). في ASP.NET Core، تُستخدم مكتبات مثل **xUnit** لإطار الاختبار و **Moq** لإنشاء الكائنات الوهمية (Mocks).

## 1. لماذا اختبارات الوحدة؟

*   **اكتشاف الأخطاء مبكراً:** تساعد في تحديد المشاكل في مرحلة التطوير بدلاً من الإنتاج.
*   **تحسين جودة الكود:** تشجع على كتابة كود نظيف، قابل للفصل (Decoupled)، وقابل للاختبار.
*   **توفير الثقة:** تمنح المطورين الثقة في أن التغييرات الجديدة لا تكسر الوظائف الموجودة.
*   **توثيق السلوك:** تعمل الاختبارات كتوثيق حي لسلوك الكود.
*   **تسهيل إعادة الهيكلة:** تسمح بإعادة هيكلة الكود بأمان مع التأكد من أن الوظائف الأساسية لا تزال تعمل.

## 2. إعداد مشروع الاختبار

للبدء، ستحتاج إلى إنشاء مشروع اختبار منفصل في الحل الخاص بك.

### أ. إنشاء مشروع اختبار xUnit

افتح سطر الأوامر في مجلد الحل الخاص بك:

```bash
# في مجلد الحل (Solution Folder)

dotnet new xunit -n MyApiProject.Tests
dotnet sln add MyApiProject.Tests/MyApiProject.Tests.csproj

# إضافة مرجع من مشروع الاختبار إلى المشروع الذي ستختبره
dotnet add MyApiProject.Tests/MyApiProject.Tests.csproj reference MyApiProject/MyApiProject.csproj
```

### ب. تثبيت حزم NuGet لـ Moq

Moq هي مكتبة شائعة لإنشاء الكائنات الوهمية (Mocking Framework) في .NET. تسمح لك بإنشاء نسخ وهمية من التبعيات (Dependencies) حتى تتمكن من اختبار الفئة المستهدفة بمعزل عن تبعياتها الحقيقية.

```bash
dotnet add MyApiProject.Tests/MyApiProject.Tests.csproj package Moq
```

## 3. كتابة اختبارات الوحدة (Unit Tests)

لنفرض أن لدينا خدمة بسيطة `ProductService` تعتمد على `IRepository<Product>`:

```csharp
// MyApiProject/Services/IProductService.cs
public interface IProductService
{
    Task<IEnumerable<Product>> GetAllProductsAsync();
    Task<Product?> GetProductByIdAsync(int id);
    Task<Product> AddProductAsync(Product product);
}

// MyApiProject/Services/ProductService.cs
public class ProductService : IProductService
{
    private readonly IRepository<Product> _productRepository;

    public ProductService(IRepository<Product> productRepository)
    {
        _productRepository = productRepository;
    }

    public async Task<IEnumerable<Product>> GetAllProductsAsync()
    {
        return await _productRepository.GetAllAsync();
    }

    public async Task<Product?> GetProductByIdAsync(int id)
    {
        if (id <= 0) return null; // مثال على التحقق من المدخلات
        return await _productRepository.GetByIdAsync(id);
    }

    public async Task<Product> AddProductAsync(Product product)
    {
        if (string.IsNullOrWhiteSpace(product.Name)) throw new ArgumentException("Product name cannot be empty.");
        return await _productRepository.AddAsync(product);
    }
}

// MyApiProject/Data/IRepository.cs (واجهة عامة للتعامل مع البيانات)
public interface IRepository<T>
{
    Task<IEnumerable<T>> GetAllAsync();
    Task<T?> GetByIdAsync(int id);
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}
```

الآن، لنكتب اختبارات وحدة لـ `ProductService`:

```csharp
// MyApiProject.Tests/ProductServiceTests.cs
using Xunit;
using Moq;
using MyApiProject.Models;
using MyApiProject.Services;
using MyApiProject.Data;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

public class ProductServiceTests
{
    [Fact] // سمة تشير إلى أن هذه دالة اختبار
    public async Task GetAllProductsAsync_ReturnsAllProducts()
    {
        // Arrange (التحضير): إعداد الكائنات الوهمية والبيانات
        var mockRepository = new Mock<IRepository<Product>>();
        var products = new List<Product> { new Product { Id = 1, Name = "Test1" }, new Product { Id = 2, Name = "Test2" } };
        mockRepository.Setup(repo => repo.GetAllAsync()).ReturnsAsync(products);

        var service = new ProductService(mockRepository.Object);

        // Act (التنفيذ): استدعاء الدالة المراد اختبارها
        var result = await service.GetAllProductsAsync();

        // Assert (التحقق): التأكد من أن النتيجة صحيحة
        Assert.NotNull(result);
        Assert.Equal(2, result.Count());
        Assert.Contains(result, p => p.Name == "Test1");
    }

    [Fact]
    public async Task GetProductByIdAsync_ReturnsProduct_WhenProductExists()
    {
        // Arrange
        var mockRepository = new Mock<IRepository<Product>>();
        var product = new Product { Id = 1, Name = "Test Product" };
        mockRepository.Setup(repo => repo.GetByIdAsync(1)).ReturnsAsync(product);

        var service = new ProductService(mockRepository.Object);

        // Act
        var result = await service.GetProductByIdAsync(1);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(1, result.Id);
        Assert.Equal("Test Product", result.Name);
    }

    [Fact]
    public async Task GetProductByIdAsync_ReturnsNull_WhenProductDoesNotExist()
    {
        // Arrange
        var mockRepository = new Mock<IRepository<Product>>();
        mockRepository.Setup(repo => repo.GetByIdAsync(It.IsAny<int>())).ReturnsAsync((Product?)null);

        var service = new ProductService(mockRepository.Object);

        // Act
        var result = await service.GetProductByIdAsync(99);

        // Assert
        Assert.Null(result);
    }

    [Theory] // سمة لاختبار نفس الدالة ببيانات مختلفة
    [InlineData(0)]
    [InlineData(-5)]
    public async Task GetProductByIdAsync_ReturnsNull_WhenIdIsInvalid(int invalidId)
    {
        // Arrange
        var mockRepository = new Mock<IRepository<Product>>();
        var service = new ProductService(mockRepository.Object);

        // Act
        var result = await service.GetProductByIdAsync(invalidId);

        // Assert
        Assert.Null(result);
    }

    [Fact]
    public async Task AddProductAsync_AddsProductSuccessfully()
    {
        // Arrange
        var mockRepository = new Mock<IRepository<Product>>();
        var newProduct = new Product { Id = 0, Name = "New Product" }; // ID 0 قبل الإضافة
        var addedProduct = new Product { Id = 3, Name = "New Product" }; // ID 3 بعد الإضافة
        mockRepository.Setup(repo => repo.AddAsync(It.IsAny<Product>()))
                      .ReturnsAsync((Product p) => { p.Id = 3; return p; }); // محاكاة إضافة وتعيين ID

        var service = new ProductService(mockRepository.Object);

        // Act
        var result = await service.AddProductAsync(newProduct);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(3, result.Id); // التأكد من تعيين ID
        mockRepository.Verify(repo => repo.AddAsync(It.IsAny<Product>()), Times.Once); // التأكد من استدعاء الدالة مرة واحدة
    }

    [Fact]
    public async Task AddProductAsync_ThrowsException_WhenProductNameIsEmpty()
    {
        // Arrange
        var mockRepository = new Mock<IRepository<Product>>();
        var invalidProduct = new Product { Id = 0, Name = "" };
        var service = new ProductService(mockRepository.Object);

        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(() => service.AddProductAsync(invalidProduct));
    }
}
```

## 4. تشغيل الاختبارات

يمكنك تشغيل الاختبارات من سطر الأوامر باستخدام .NET CLI:

```bash
dotnet test MyApiProject.Tests/MyApiProject.Tests.csproj
```

أو من داخل Visual Studio / Visual Studio Code باستخدام Test Explorer.

## 5. مبادئ اختبارات الوحدة الجيدة (FIRST Principles)

لضمان فعالية اختبارات الوحدة، يجب أن تتبع مبادئ FIRST:

*   **F**ast (سريعة): يجب أن تعمل الاختبارات بسرعة لكي يتم تشغيلها بشكل متكرر.
*   **I**solated (معزولة): يجب أن يكون كل اختبار مستقلاً عن الآخرين.
*   **R**epeatable (قابلة للتكرار): يجب أن تعطي الاختبارات نفس النتائج في كل مرة يتم تشغيلها.
*   **S**elf-validating (ذاتية التحقق): يجب أن تكون نتائج الاختبارات (نجاح/فشل) واضحة ولا تتطلب تدخلاً يدوياً.
*   **T**imely (في الوقت المناسب): يجب كتابة الاختبارات في نفس وقت كتابة الكود المنتج أو قبله (Test-Driven Development - TDD).

## 6. Mocking vs Stubbing

*   **Mocking:** تستخدم للتحقق من التفاعلات (Interactions) مع التبعيات. أنت تهتم بما إذا كانت دالة معينة على التبعية قد تم استدعاؤها، وكم مرة، وبأي معلمات. (مثال: `mockRepository.Verify(...)`).
*   **Stubbing:** تستخدم لتوفير استجابات محددة للتبعيات. أنت تهتم بالنتيجة التي ترجعها التبعية وليس بكيفية استدعائها. (مثال: `mockRepository.Setup(...).ReturnsAsync(...)`).

## الخلاصة

اختبارات الوحدة هي استثمار حاسم في جودة واستقرار تطبيقات ASP.NET Core. من خلال استخدام xUnit و Moq، يمكنك بناء مجموعة قوية من الاختبارات التي تضمن أن الكود الخاص بك يعمل كما هو متوقع، مما يقلل من الأخطاء ويزيد من ثقتك في التغييرات المستقبلية. في الدرس التالي، سنتناول هندسة البرمجيات النظيفة (Clean Architecture) لتنظيم المشاريع الكبيرة.
