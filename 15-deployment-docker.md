# 15-deployment-docker.md

# النشر باستخدام Docker و CI/CD

## المقدمة

بعد تطوير تطبيق ASP.NET Core، تأتي خطوة نشره (Deployment) لجعله متاحاً للمستخدمين. في بيئات التطوير الحديثة، أصبح **Docker** و **Continuous Integration/Continuous Deployment (CI/CD)** أدوات أساسية لتبسيط عملية النشر، ضمان الاتساق بين البيئات، وتحقيق التسليم المستمر. هذا الدرس سيغطي كيفية تغليف تطبيقك باستخدام Docker ونشره باستخدام مبادئ CI/CD.

## 1. Docker وتغليف التطبيقات (Containerization)

Docker هو منصة تسمح لك بتغليف تطبيقاتك وتبعياتها في حاويات (Containers) معزولة. هذه الحاويات قابلة للنقل (Portable) ويمكن تشغيلها بشكل متسق في أي بيئة تدعم Docker، مما يحل مشكلة "يعمل على جهازي ولكن لا يعمل على خادم الإنتاج".

### أ. مفاهيم Docker الأساسية

*   **Image (صورة):** قالب للقراءة فقط يحتوي على تعليمات لإنشاء حاوية. تتضمن الكود، وقت التشغيل، المكتبات، ومتغيرات البيئة.
*   **Container (حاوية):** مثيل قابل للتشغيل من الصورة. هو التطبيق الخاص بك الذي يعمل في بيئة معزولة.
*   **Dockerfile:** ملف نصي يحتوي على جميع الأوامر التي يمكن للمستخدم استدعاؤها في سطر الأوامر لإنشاء صورة Docker.

### ب. إنشاء Dockerfile لتطبيق ASP.NET Core

في جذر مشروع ASP.NET Core (بجانب ملف `.csproj`)، أنشئ ملفاً باسم `Dockerfile` (بدون امتداد):

```dockerfile
# Dockerfile

# المرحلة الأولى: البناء (Build Stage)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApiProject/MyApiProject.csproj", "MyApiProject/"]
RUN dotnet restore "MyApiProject/MyApiProject.csproj"
COPY . .
WORKDIR "/src/MyApiProject"
RUN dotnet build "MyApiProject.csproj" -c Release -o /app/build

# المرحلة الثانية: النشر (Publish Stage)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS publish
WORKDIR /app
COPY --from=build /app/build .
ENTRYPOINT ["dotnet", "MyApiProject.dll"]
```

**شرح Dockerfile:**

*   **`FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build`**: يبدأ بصورة .NET SDK (8.0) التي تحتوي على الأدوات اللازمة للبناء. `AS build` يعطي هذه المرحلة اسماً.
*   **`WORKDIR /src`**: يحدد دليل العمل داخل الحاوية.
*   **`COPY`**: ينسخ الملفات من المضيف إلى الحاوية. ننسخ ملف `.csproj` أولاً للسماح لـ Docker بتخزين خطوة `dotnet restore` مؤقتاً إذا لم يتغير ملف المشروع.
*   **`RUN dotnet restore`**: يستعيد تبعيات NuGet.
*   **`RUN dotnet build`**: يبني المشروع في وضع `Release`.
*   **`FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS publish`**: يبدأ بمرحلة جديدة باستخدام صورة .NET ASPNET Runtime (8.0) فقط. هذه الصورة أصغر بكثير ولا تحتوي على أدوات البناء، مما يقلل من حجم الصورة النهائية.
*   **`COPY --from=build /app/build .`**: ينسخ مخرجات البناء من المرحلة `build` إلى المرحلة `publish`.
*   **`ENTRYPOINT ["dotnet", "MyApiProject.dll"]`**: يحدد الأمر الذي سيتم تنفيذه عند بدء تشغيل الحاوية.

### ج. بناء صورة Docker وتشغيل حاوية

من سطر الأوامر في جذر المشروع (حيث يوجد `Dockerfile`):

```bash
# بناء الصورة (استبدل myapiproject باسم صورتك)
docker build -t myapiproject .

# تشغيل الحاوية (ربط المنفذ 8080 على المضيف بالمنفذ 80 في الحاوية)
docker run -p 8080:80 myapiproject
```

الآن، يجب أن يكون تطبيقك متاحاً على `http://localhost:8080`.

## 2. CI/CD (Continuous Integration / Continuous Deployment)

CI/CD هو ممارسة تطوير برمجيات تهدف إلى أتمتة مراحل بناء، اختبار، ونشر التطبيقات. يضمن CI/CD أن الكود يتم دمجه واختباره ونشره بشكل متكرر وموثوق.

### أ. مفاهيم CI/CD الأساسية

*   **Continuous Integration (CI):** دمج تغييرات الكود بشكل متكرر في مستودع مركزي، ثم بناء واختبار الكود تلقائياً.
*   **Continuous Delivery (CD):** توسيع CI لضمان أن الكود يمكن نشره في أي وقت بعد اجتياز جميع الاختبارات.
*   **Continuous Deployment (CD):** أتمتة عملية النشر بالكامل، بحيث يتم نشر كل تغيير يجتاز الاختبارات تلقائياً إلى الإنتاج.

### ب. أدوات CI/CD الشائعة

*   **GitHub Actions:** أداة CI/CD مدمجة في GitHub، سهلة الاستخدام ومناسبة للمشاريع الصغيرة والكبيرة.
*   **Azure DevOps Pipelines:** حل شامل من Microsoft يدعم CI/CD لمختلف المنصات والتقنيات.
*   **GitLab CI/CD:** مدمج في GitLab، يوفر ميزات قوية لأتمتة دورة حياة التطوير.
*   **Jenkins:** خادم أتمتة مفتوح المصدر، مرن وقابل للتوسيع.

### ج. مثال على GitHub Actions لـ ASP.NET Core و Docker

في مستودع GitHub الخاص بك، أنشئ ملفاً في المسار `.github/workflows/main.yml`:

```yaml
# .github/workflows/main.yml
name: Build and Deploy ASP.NET Core Docker Image

on:
  push:
    branches:
      - main # تشغيل هذا Workflow عند الدفع إلى فرع main
  pull_request:
    branches:
      - main # تشغيل هذا Workflow عند فتح Pull Request على فرع main

env:
  DOTNET_VERSION: '8.0.x'
  PROJECT_PATH: 'MyApiProject/MyApiProject.csproj' # مسار ملف المشروع الخاص بك
  DOCKER_IMAGE_NAME: 'myapiproject' # اسم صورة Docker

jobs:
  build-and-push-docker-image:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Restore dependencies
      run: dotnet restore ${{ env.PROJECT_PATH }}

    - name: Build
      run: dotnet build ${{ env.PROJECT_PATH }} --no-restore --configuration Release

    - name: Test
      run: dotnet test --no-build --verbosity normal

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKER_USERNAME }}/${{ env.DOCKER_IMAGE_NAME }}:latest
        file: ./Dockerfile
```

**شرح Workflow:**

*   **`on: push` و `pull_request`**: يحدد متى سيتم تشغيل هذا Workflow (عند الدفع إلى `main` أو فتح `pull_request`).
*   **`env`**: يحدد متغيرات البيئة التي يمكن استخدامها في جميع المهام.
*   **`jobs`**: يحتوي على المهام التي سيتم تنفيذها.
*   **`build-and-push-docker-image`**: اسم المهمة.
*   **`runs-on: ubuntu-latest`**: يحدد البيئة التي ستعمل عليها المهمة.
*   **`actions/checkout@v4`**: يجلب الكود من المستودع.
*   **`actions/setup-dotnet@v4`**: يقوم بإعداد بيئة .NET.
*   **`dotnet restore`, `dotnet build`, `dotnet test`**: ينفذ الأوامر الأساسية لـ CI.
*   **`docker/login-action@v3`**: يسجل الدخول إلى Docker Hub باستخدام بيانات الاعتماد المخزنة كـ GitHub Secrets.
*   **`docker/build-push-action@v5`**: يبني صورة Docker ويدفعها إلى Docker Hub.

**ملاحظات هامة:**

*   يجب عليك إضافة `DOCKER_USERNAME` و `DOCKER_PASSWORD` كـ Secrets في إعدادات مستودع GitHub الخاص بك.
*   هذا المثال يقوم بالبناء والدفع إلى Docker Hub. في سيناريو الإنتاج، قد تحتاج إلى نشر الحاوية إلى خدمة سحابية (مثل Azure Container Apps, AWS ECS, Kubernetes).

## 3. النشر إلى الخدمات السحابية (Cloud Deployment)

بعد تغليف تطبيقك في Docker، يصبح نشره على الخدمات السحابية أسهل بكثير. معظم موفري الخدمات السحابية يدعمون تشغيل حاويات Docker.

### أمثلة على خدمات النشر السحابية:

*   **Azure App Service / Azure Container Apps:** حلول سهلة الاستخدام لنشر تطبيقات الويب والحاويات على Azure.
*   **AWS Elastic Beanstalk / AWS ECS / AWS EKS:** خدمات مرنة وقابلة للتوسع لنشر التطبيقات والحاويات على AWS.
*   **Google Cloud Run / Google Kubernetes Engine:** حلول مماثلة على Google Cloud.
*   **Kubernetes:** نظام تنسيق (Orchestration) للحاويات، مثالي لإدارة التطبيقات الموزعة على نطاق واسع.

## الخلاصة

Docker و CI/CD هما أدوات لا غنى عنها في دورة حياة تطوير البرمجيات الحديثة. من خلال تغليف تطبيق ASP.NET Core الخاص بك في حاويات Docker وأتمتة عمليات البناء والاختبار والنشر باستخدام أدوات CI/CD مثل GitHub Actions، يمكنك تحقيق تسليم مستمر، تقليل الأخطاء، وضمان بيئة نشر متسقة وموثوقة. هذا يختتم خارطة الطريق الخاصة بنا، ونتمنى لك رحلة موفقة في عالم تطوير ASP.NET Core!
