# 01-csharp-essentials.md

# أساسيات C# المتقدمة للمطورين

## المقدمة

C# هي لغة البرمجة الأساسية لتطوير تطبيقات ASP.NET Core. بينما يغطي هذا الدليل تطوير الويب، فإن الفهم العميق لميزات C# الحديثة أمر بالغ الأهمية لكتابة كود نظيف، فعال، وقابل للصيانة. سنركز هنا على المفاهيم التي تُستخدم بكثرة في سياق ASP.NET Core.

## 1. LINQ (Language Integrated Query)

LINQ هي مجموعة من التقنيات في C# التي توفر إمكانية الاستعلام عن البيانات من مصادر مختلفة (مثل الكائنات، قواعد البيانات، XML) باستخدام بناء جملة موحد. تجعل LINQ التعامل مع مجموعات البيانات أكثر سهولة وقراءة.

### أمثلة على LINQ

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int CategoryId { get; set; }
}

public class LinqExamples
{
    public static void Run()
    {
        List<Product> products = new List<Product>
        {
            new Product { Id = 1, Name = "Laptop", Price = 1200, CategoryId = 1 },
            new Product { Id = 2, Name = "Mouse", Price = 25, CategoryId = 2 },
            new Product { Id = 3, Name = "Keyboard", Price = 75, CategoryId = 2 },
            new Product { Id = 4, Name = "Monitor", Price = 300, CategoryId = 1 },
            new Product { Id = 5, Name = "Webcam", Price = 50, CategoryId = 2 }
        };

        // استعلام للحصول على المنتجات بسعر أكبر من 100
        var expensiveProducts = products.Where(p => p.Price > 100).ToList();
        Console.WriteLine("Expensive Products:");
        expensiveProducts.ForEach(p => Console.WriteLine($"- {p.Name} ({p.Price:C})"));

        // استعلام لتجميع المنتجات حسب الفئة وحساب متوسط السعر
        var categoryAvgPrices = products.GroupBy(p => p.CategoryId)
                                        .Select(g => new { CategoryId = g.Key, AveragePrice = g.Average(p => p.Price) })
                                        .ToList();
        Console.WriteLine("\nCategory Average Prices:");
        categoryAvgPrices.ForEach(c => Console.WriteLine($"- Category {c.CategoryId}: {c.AveragePrice:C}"));
    }
}
```

## 2. Asynchronous Programming (Async/Await)

تطوير الويب الحديث يتطلب تطبيقات سريعة الاستجابة. البرمجة غير المتزامنة تسمح للتطبيق بإجراء عمليات طويلة الأمد (مثل استعلامات قواعد البيانات أو طلبات الشبكة) دون حظر مؤشر الترابط الرئيسي (Main Thread)، مما يحسن تجربة المستخدم وأداء الخادم.

### أمثلة على Async/Await

```csharp
using System;
using System.Threading.Tasks;

public class AsyncExamples
{
    public static async Task Run()
    {
        Console.WriteLine("Starting long running operation...");
        string result = await GetLongRunningResultAsync();
        Console.WriteLine($"Operation completed: {result}");
    }

    private static async Task<string> GetLongRunningResultAsync()
    {
        // محاكاة عملية تستغرق وقتاً طويلاً (مثل استعلام قاعدة بيانات)
        await Task.Delay(2000); 
        return "Data fetched successfully!";
    }
}
```

## 3. Record Types

تم تقديم Record Types في C# 9 لتبسيط إنشاء أنواع البيانات التي تهدف إلى تخزين البيانات بشكل أساسي (Data-centric types). توفر Records مساواة قيمية (Value Equality)، وطباعة تلقائية (ToString)، وميزات أخرى تقلل من الكود المتكرر (Boilerplate Code).

### أمثلة على Record Types

```csharp
using System;

public record Person(string FirstName, string LastName);

public class RecordExamples
{
    public static void Run()
    {
        Person person1 = new Person("John", "Doe");
        Person person2 = new Person("John", "Doe");
        Person person3 = new Person("Jane", "Smith");

        Console.WriteLine($"Person 1: {person1}"); // ToString تلقائي
        Console.WriteLine($"Person 1 equals Person 2: {person1 == person2}"); // مساواة قيمية
        Console.WriteLine($"Person 1 equals Person 3: {person1 == person3}");
    }
}
```

## 4. Pattern Matching

تتيح Pattern Matching فحص بنية البيانات أو الكائنات لتحديد ما إذا كانت تفي بمتطلبات معينة، ثم استخراج البيانات منها. هذا يجعل الكود أكثر وضوحاً واختصاراً عند التعامل مع أنواع مختلفة أو حالات متعددة.

### أمثلة على Pattern Matching

```csharp
using System;

public class Shape { }
public class Circle : Shape { public double Radius { get; set; } }
public class Rectangle : Shape { public double Length { get; set; } public double Width { get; set; } }

public class PatternMatchingExamples
{
    public static void Run()
    {
        Shape circle = new Circle { Radius = 5 };
        Shape rectangle = new Rectangle { Length = 10, Width = 4 };
        Shape unknown = new Shape();

        PrintShapeInfo(circle);
        PrintShapeInfo(rectangle);
        PrintShapeInfo(unknown);
    }

    public static void PrintShapeInfo(Shape shape)
    {
        if (shape is Circle c)
        {
            Console.WriteLine($"Circle with radius {c.Radius}");
        }
        else if (shape is Rectangle r)
        {
            Console.WriteLine($"Rectangle with length {r.Length} and width {r.Width}");
        }
        else if (shape is null)
        {
            Console.WriteLine("Null shape");
        }
        else
        {
            Console.WriteLine("Unknown shape");
        }
    }
}
```

## 5. Nullable Reference Types

تم تقديم Nullable Reference Types في C# 8 للمساعدة في تقليل أخطاء `NullReferenceException` عن طريق السماح للمطورين بالتعبير عن نيتهم حول ما إذا كان المتغير المرجعي يمكن أن يكون `null` أم لا. يتم تمكينها عادةً في ملف المشروع (`.csproj`).

### تمكين Nullable Reference Types

```xml
<!-- في ملف .csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable> <!-- هنا يتم التمكين -->
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

### أمثلة على Nullable Reference Types

```csharp
using System;

#nullable enable // يمكن تمكينها على مستوى الملف أيضاً

public class User
{
    public string Name { get; set; } = string.Empty;
    public string? Email { get; set; } // يمكن أن يكون null
}

public class NullableExamples
{
    public static void Run()
    {
        User user1 = new User { Name = "Alice" };
        User user2 = new User { Name = "Bob", Email = "bob@example.com" };

        Console.WriteLine($"User 1 Name: {user1.Name}, Email: {user1.Email ?? "N/A"}");
        Console.WriteLine($"User 2 Name: {user2.Name}, Email: {user2.Email}");

        // سيتم تحذيرك من قبل المحول البرمجي إذا حاولت الوصول إلى user1.Email مباشرة دون التحقق من null
        // string email = user1.Email; // تحذير: قد يكون null

        if (user1.Email is not null)
        {
            Console.WriteLine($"User 1 Email (checked): {user1.Email}");
        }
    }
}
```

## الخلاصة

إتقان هذه الميزات المتقدمة في C# سيمكنك من كتابة كود ASP.NET Core أكثر قوة، كفاءة، وأماناً. ستصادف هذه المفاهيم بشكل متكرر في مشاريع الويب الحديثة، وفهمها سيجعل عملية التعلم والتطوير أكثر سلاسة. في الدرس التالي، سنتناول هيكلية مشروع ASP.NET Core وأدوات CLI.
