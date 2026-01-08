# 06-data-access-efcore.md

# التعامل مع قواعد البيانات عبر EF Core

## المقدمة

Entity Framework Core (EF Core) هو ORM (Object-Relational Mapper) خفيف الوزن ومتعدد المنصات من Microsoft. يسمح للمطورين بالتعامل مع قواعد البيانات باستخدام كائنات .NET (C#) بدلاً من كتابة استعلامات SQL يدوياً. يقلل EF Core من الكود المتكرر (boilerplate code) ويسهل التفاعل مع قواعد البيانات في تطبيقات ASP.NET Core.

## 1. مفاهيم أساسية في EF Core

*   **DbContext:** هو الفئة الرئيسية في EF Core. يمثل جلسة (Session) مع قاعدة البيانات ويستخدم للاستعلام عن البيانات وحفظها. كل مثيل من `DbContext` يمثل وحدة عمل (Unit of Work).
*   **Entities (الكيانات):** هي فئات C# بسيطة تمثل جداول في قاعدة البيانات. كل خاصية (Property) في الكيان عادةً ما تتوافق مع عمود (Column) في الجدول.
*   **Migrations (الترحيلات):** هي طريقة لإدارة تغييرات مخطط قاعدة البيانات (Database Schema) بمرور الوقت. تسمح لك بتعريف التغييرات في الكود ثم تطبيقها على قاعدة البيانات.
*   **Providers (الموفرون):** EF Core يدعم قواعد بيانات مختلفة (SQL Server, SQLite, PostgreSQL, MySQL) من خلال موفرين خاصين بكل قاعدة بيانات.

## 2. إعداد EF Core في مشروع ASP.NET Core

### أ. تثبيت حزم NuGet

للبدء، تحتاج إلى تثبيت حزم NuGet المناسبة لقاعدة البيانات التي ستستخدمها. على سبيل المثال، لـ SQL Server:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools # لأوامر Migrations
```

### ب. تعريف الكيانات (Entities)

لنقم بإنشاء كيان بسيط يمثل منتجاً:

```csharp
// Models/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string? Description { get; set; }
}
```

### ج. إنشاء DbContext

قم بإنشاء فئة ترث من `DbContext` وأضف خصائص `DbSet<TEntity>` لكل كيان تريد التعامل معه:

```csharp
// Data/ApplicationDbContext.cs
using Microsoft.EntityFrameworkCore;
using MyApiProject.Models;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; } = null!;

    // يمكنك إضافة تكوينات إضافية للكيانات هنا (Fluent API)
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>().Property(p => p.Price).HasColumnType("decimal(18,2)");
    }
}
```

### د. تسجيل DbContext في حاوية DI

في `Program.cs`، قم بتسجيل `ApplicationDbContext` كخدمة واستخدم سلسلة الاتصال (Connection String):

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// ... إضافة خدمات أخرى ...

// قراءة سلسلة الاتصال من appsettings.json
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

// تسجيل DbContext
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString)); // استخدم UseSqlite, UseNpgsql, UseMySql حسب قاعدة بياناتك

var app = builder.Build();

// ... تكوين الـ Middleware ...
```

وفي ملف `appsettings.json`، أضف سلسلة الاتصال:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyApiDb;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
}
```

## 3. إدارة مخطط قاعدة البيانات باستخدام Migrations

### أ. إنشاء Migration

افتح سطر الأوامر في مجلد المشروع وقم بتشغيل الأمر التالي:

```bash
dotnet ef migrations add InitialCreate
```

سيقوم هذا الأمر بإنشاء مجلد `Migrations` يحتوي على ملف يصف التغييرات التي سيتم تطبيقها على قاعدة البيانات.

### ب. تطبيق Migration على قاعدة البيانات

بعد إنشاء Migration، يمكنك تطبيقها على قاعدة البيانات لإنشاء المخطط أو تحديثه:

```bash
dotnet ef database update
```

### ج. تحديث Migration

إذا قمت بتغيير كياناتك (مثلاً، أضفت خاصية جديدة إلى `Product`)، كرر الخطوات:

1.  `dotnet ef migrations add AddProductDescription` (استبدل `AddProductDescription` باسم وصفي).
2.  `dotnet ef database update`.

## 4. عمليات CRUD الأساسية باستخدام EF Core

الآن بعد أن تم إعداد EF Core، يمكننا استخدامه لإجراء عمليات CRUD (Create, Read, Update, Delete) في المتحكمات أو الخدمات.

### أ. إضافة بيانات (Create)

```csharp
// في متحكم أو خدمة
public async Task<Product> AddProduct(Product product)
{
    _context.Products.Add(product);
    await _context.SaveChangesAsync(); // حفظ التغييرات في قاعدة البيانات
    return product;
}
```

### ب. قراءة بيانات (Read)

```csharp
// الحصول على جميع المنتجات
public async Task<IEnumerable<Product>> GetAllProducts()
{
    return await _context.Products.ToListAsync();
}

// الحصول على منتج بواسطة ID
public async Task<Product?> GetProductById(int id)
{
    return await _context.Products.FindAsync(id); // يبحث أولاً في الذاكرة ثم في قاعدة البيانات
    // أو
    // return await _context.Products.FirstOrDefaultAsync(p => p.Id == id);
}
```

### ج. تحديث بيانات (Update)

```csharp
public async Task<bool> UpdateProduct(Product product)
{
    _context.Entry(product).State = EntityState.Modified;
    try
    {
        await _context.SaveChangesAsync();
        return true;
    }
    catch (DbUpdateConcurrencyException)
    {
        if (!_context.Products.Any(e => e.Id == product.Id))
        {
            return false; // المنتج غير موجود
        }
        throw;
    }
}
```

### د. حذف بيانات (Delete)

```csharp
public async Task<bool> DeleteProduct(int id)
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
    {
        return false;
    }

    _context.Products.Remove(product);
    await _context.SaveChangesAsync();
    return true;
}
```

## 5. الاستعلامات المعقدة (Advanced Queries)

EF Core يدعم LINQ، مما يسمح بكتابة استعلامات معقدة بطريقة برمجية:

```csharp
public async Task<IEnumerable<Product>> GetProductsByPriceRange(decimal minPrice, decimal maxPrice)
{
    return await _context.Products
                         .Where(p => p.Price >= minPrice && p.Price <= maxPrice)
                         .OrderBy(p => p.Name)
                         .ToListAsync();
}

public async Task<IEnumerable<object>> GetProductNamesAndPrices()
{
    return await _context.Products
                         .Select(p => new { p.Name, p.Price })
                         .ToListAsync();
}
```

## الخلاصة

Entity Framework Core هو أداة قوية ومرنة للتعامل مع قواعد البيانات في ASP.NET Core. من خلال فهم مفاهيمه الأساسية، كيفية إعداده، وإجراء عمليات CRUD، ستتمكن من بناء طبقة بيانات قوية لتطبيقاتك. في الدرس التالي، سنتعمق في جوانب الأمان، تحديداً المصادقة (Authentication) والتفويض (Authorization).
