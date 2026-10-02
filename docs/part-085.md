# Part 085: CI/CD กับ GitHub Actions

## เนื้อหาใน Part นี้
- CI/CD pipeline คืออะไร
- GitHub Actions workflow
- Build, Test, Deploy steps
- Docker build in CI
- Deploy to Azure/cloud
- โปรแกรมตัวอย่าง: Complete CI/CD pipeline

---

## 1. CI/CD คืออะไร

**CI (Continuous Integration)** = รวม code จากหลายคนเข้าด้วยกันบ่อยๆ พร้อม automated tests

**CD (Continuous Delivery)** = ทุก commit พร้อม deploy ไปยัง staging โดยอัตโนมัติ

**CD (Continuous Deployment)** = Deploy ไป production อัตโนมัติทุก commit ที่ผ่าน tests

```
Developer ──push──► GitHub ──trigger──► CI Pipeline
                                          │
                                    ┌─────▼─────┐
                                    │   Build   │
                                    └─────┬─────┘
                                          │
                                    ┌─────▼─────┐
                                    │   Test    │
                                    └─────┬─────┘
                                          │
                                    ┌─────▼─────┐
                                    │  Security │
                                    │   Scan    │
                                    └─────┬─────┘
                                          │
                                    ┌─────▼─────┐
                                    │   Docker  │
                                    │   Build   │
                                    └─────┬─────┘
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                        ┌─────▼─────┐          ┌─────▼─────┐
                        │  Deploy   │          │  Deploy   │
                        │  Staging  │          │Production │
                        └───────────┘          └───────────┘
```

---

## 2. GitHub Actions Basics

### Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI Pipeline           # ชื่อ workflow

on:                         # Triggers
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:        # Manual trigger

env:                        # Global environment variables
  DOTNET_VERSION: '9.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:                       # Jobs รันแบบ parallel (โดย default)
  build:
    runs-on: ubuntu-latest  # Runner
    steps:
    - uses: actions/checkout@v4  # Action
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    - name: Build
      run: dotnet build --configuration Release
```

### Triggers

```yaml
on:
  # Push to specific branches
  push:
    branches: [main, 'release/**']
    paths:
      - 'src/**'
      - '*.csproj'
    tags:
      - 'v*'

  # Pull requests
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  # Schedule (cron)
  schedule:
    - cron: '0 2 * * *'  # ทุกวัน 02:00 UTC

  # Manual trigger
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
      version:
        description: 'Version tag'
        required: false
        type: string
```

---

## 3. Complete CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '9.0.x'
  
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # Cancel previous run ถ้า new push มา

jobs:
  # ─── Build & Test ─────────────────────────────────
  test:
    name: Build & Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Cache NuGet packages
      uses: actions/cache@v4
      with:
        path: ~/.nuget/packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
        restore-keys: |
          ${{ runner.os }}-nuget-
    
    - name: Restore
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore --configuration Release
    
    - name: Test
      run: |
        dotnet test --no-build --configuration Release \
          --collect:"XPlat Code Coverage" \
          --results-directory ./TestResults \
          --logger "trx;LogFileName=test-results.trx"
      env:
        ConnectionStrings__DefaultConnection: "Host=localhost;Database=testdb;Username=postgres;Password=postgres"
        Redis__ConnectionString: "localhost:6379"
    
    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: ./TestResults/*.trx
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v4
      with:
        files: ./TestResults/**/coverage.cobertura.xml
        fail_ci_if_error: false

  # ─── Code Analysis ────────────────────────────────
  analyze:
    name: Code Analysis
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Install dotnet-format
      run: dotnet tool install -g dotnet-format
    
    - name: Check formatting
      run: dotnet format --verify-no-changes --severity warn
    
    - name: Run .NET Analyzers
      run: dotnet build --configuration Release /p:TreatWarningsAsErrors=true
    
    # OWASP Dependency Check
    - name: Dependency vulnerability scan
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'myapp'
        path: '.'
        format: 'HTML'
    
    - name: Upload vulnerability report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: vulnerability-report
        path: reports/

  # ─── Security Scan ────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: csharp
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Build
      run: dotnet build --configuration Release
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
```

---

## 4. Docker Build in CI

```yaml
# .github/workflows/docker.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          BUILD_VERSION=${{ github.sha }}
          BUILD_DATE=${{ github.event.head_commit.timestamp }}

    - name: Security scan image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
        format: 'table'
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
```

---

## 5. Deploy to Azure

### Azure Container Apps Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  workflow_run:
    workflows: ["Docker Build & Push"]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging  # ต้อง approve ใน GitHub
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Azure Login
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy to Azure Container Apps (Staging)
      uses: azure/container-apps-deploy-action@v1
      with:
        appSourcePath: ${{ github.workspace }}
        acrName: myregistry
        containerAppName: myapp-staging
        resourceGroup: myapp-rg
        imageToDeploy: ghcr.io/${{ github.repository }}:${{ github.sha }}
        environmentVariables: |
          ASPNETCORE_ENVIRONMENT=Staging
    
    - name: Run smoke tests
      run: |
        sleep 30  # Wait for deployment
        curl -f https://staging.myapp.com/health || exit 1
        curl -f https://staging.myapp.com/health/ready || exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # ต้อง manual approve
    
    steps:
    - name: Azure Login
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy to Azure Container Apps (Production)
      uses: azure/container-apps-deploy-action@v1
      with:
        acrName: myregistry
        containerAppName: myapp-prod
        resourceGroup: myapp-rg
        imageToDeploy: ghcr.io/${{ github.repository }}:${{ github.sha }}
    
    - name: Verify deployment
      run: |
        sleep 60
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.myapp.com/health)
        if [ "$STATUS" != "200" ]; then
          echo "Health check failed with status $STATUS"
          exit 1
        fi
    
    - name: Notify Slack on success
      if: success()
      uses: slackapi/slack-github-action@v1.26.0
      with:
        payload: |
          {
            "text": "✅ Deployed to production: ${{ github.sha }}"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    
    - name: Notify Slack on failure
      if: failure()
      uses: slackapi/slack-github-action@v1.26.0
      with:
        payload: |
          {
            "text": "❌ Production deployment failed: ${{ github.sha }}"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 6. โปรแกรมตัวอย่าง: Complete CI/CD Pipeline

### โครงสร้าง Repository

```
myapp/
├── .github/
│   └── workflows/
│       ├── ci.yml          # CI: build, test, analyze
│       ├── docker.yml      # Build & push Docker image
│       ├── deploy.yml      # Deploy to environments
│       └── release.yml     # Create GitHub release
├── src/
│   └── MyApp.API/
├── tests/
│   ├── MyApp.UnitTests/
│   └── MyApp.IntegrationTests/
├── k8s/                    # Kubernetes manifests
├── Dockerfile
├── docker-compose.yml
└── Makefile
```

### Full Pipeline Workflow

```yaml
# .github/workflows/pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, 'feature/**', 'fix/**']
  pull_request:
    branches: [main]
  release:
    types: [published]

env:
  DOTNET_VERSION: '9.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ════════ STAGE 1: Quality Gates ════════
  
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    - name: Check formatting
      run: dotnet format --verify-no-changes

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    - name: Restore
      run: dotnet restore
    - name: Test
      run: |
        dotnet test tests/MyApp.UnitTests/ \
          --no-restore \
          --collect:"XPlat Code Coverage" \
          --results-directory ./TestResults/unit
    - uses: actions/upload-artifact@v4
      with:
        name: unit-test-results
        path: TestResults/unit/

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: integrationtestdb
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 5s --health-timeout 5s --health-retries 5
    
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    - name: Test
      run: |
        dotnet test tests/MyApp.IntegrationTests/ \
          --no-restore \
          --results-directory ./TestResults/integration
      env:
        ConnectionStrings__DefaultConnection: "Host=localhost;Database=integrationtestdb;Username=postgres;Password=postgres"
    - uses: actions/upload-artifact@v4
      with:
        name: integration-test-results
        path: TestResults/integration/

  # ════════ STAGE 2: Build Docker ════════

  docker-build:
    name: Build & Push Docker
    needs: [lint, unit-tests, integration-tests]
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: docker/setup-buildx-action@v3
    
    - uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=
          type=ref,event=branch
          type=semver,pattern={{version}},event=tag
    
    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Scan image for vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        severity: 'HIGH,CRITICAL'
        exit-code: '1'

  # ════════ STAGE 3: Deploy ════════

  deploy-staging:
    name: Deploy to Staging
    needs: docker-build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to K8s (Staging)
      uses: azure/k8s-deploy@v4
      with:
        action: deploy
        strategy: rolling
        manifests: k8s/
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.docker-build.outputs.image-tag }}
        namespace: staging
    
    - name: Smoke test
      run: |
        sleep 30
        response=$(curl -s -o /dev/null -w "%{http_code}" https://staging.myapp.com/health)
        [ "$response" = "200" ] || (echo "Smoke test failed: $response" && exit 1)
    
    - name: Run E2E tests
      run: |
        dotnet test tests/MyApp.E2ETests/ \
          --results-directory ./TestResults/e2e
      env:
        E2E_BASE_URL: https://staging.myapp.com

  deploy-production:
    name: Deploy to Production
    needs: [docker-build, deploy-staging]
    runs-on: ubuntu-latest
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to K8s (Production)
      uses: azure/k8s-deploy@v4
      with:
        action: deploy
        strategy: canary
        percentage: 20  # Start with 20% canary
        manifests: k8s/
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.docker-build.outputs.image-tag }}
        namespace: production
    
    - name: Monitor canary
      run: |
        sleep 300  # Monitor for 5 minutes
        # Check error rates etc.
    
    - name: Promote canary
      uses: azure/k8s-deploy@v4
      with:
        action: promote
        strategy: canary
        manifests: k8s/
        namespace: production

  # ════════ STAGE 4: Release Notes ════════

  create-release:
    name: Create Release Notes
    needs: deploy-production
    runs-on: ubuntu-latest
    if: github.event_name == 'release'
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Full history for changelog
    
    - name: Generate changelog
      uses: orhun/git-cliff-action@v3
      with:
        config: cliff.toml
        args: --latest --strip header
      id: git-cliff
    
    - name: Update release notes
      uses: actions/github-script@v7
      with:
        script: |
          await github.rest.repos.updateRelease({
            owner: context.repo.owner,
            repo: context.repo.repo,
            release_id: context.payload.release.id,
            body: `${{ steps.git-cliff.outputs.content }}`
          });
```

### ASP.NET Core Health Checks สำหรับ CI

```csharp
// tests/MyApp.IntegrationTests/HealthCheckTests.cs
public class HealthCheckTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public HealthCheckTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Use in-memory database for tests
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
                if (descriptor != null)
                    services.Remove(descriptor);
                
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
            });
        });
    }

    [Fact]
    public async Task HealthCheck_ReturnsHealthy()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/health/live");
        
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var content = await response.Content.ReadAsStringAsync();
        content.Should().Contain("Healthy");
    }

    [Fact]
    public async Task ReadinessCheck_WithDatabase_ReturnsHealthy()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/health/ready");
        
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

### Makefile สำหรับ Local Development

```makefile
# Makefile

.PHONY: build test docker-build docker-push deploy

# Variables
REGISTRY = ghcr.io
IMAGE_NAME = myorg/myapp
VERSION = $(shell git describe --tags --always)

# Build
build:
	dotnet build --configuration Release

# Test
test:
	dotnet test --configuration Release \
		--collect:"XPlat Code Coverage" \
		--results-directory ./TestResults

test-watch:
	dotnet watch test --project tests/MyApp.UnitTests/

# Docker
docker-build:
	docker build -t $(REGISTRY)/$(IMAGE_NAME):$(VERSION) .
	docker tag $(REGISTRY)/$(IMAGE_NAME):$(VERSION) $(REGISTRY)/$(IMAGE_NAME):latest

docker-push:
	docker push $(REGISTRY)/$(IMAGE_NAME):$(VERSION)
	docker push $(REGISTRY)/$(IMAGE_NAME):latest

# CI simulation
ci: build test docker-build
	@echo "CI pipeline completed successfully!"

# Format
format:
	dotnet format

# Clean
clean:
	find . -name 'bin' -type d -exec rm -rf {} +
	find . -name 'obj' -type d -exec rm -rf {} +
	rm -rf TestResults/
```

---

## 7. GitHub Actions Secrets และ Environments

### Setting up Secrets

```bash
# ใช้ GitHub CLI ตั้ง secrets
gh secret set AZURE_CREDENTIALS < azure-credentials.json
gh secret set SLACK_WEBHOOK_URL --body "https://hooks.slack.com/..."
gh secret set DB_PASSWORD --body "MyStrongPass123!"

# Per-environment secrets
gh secret set DB_PASSWORD --env staging --body "StagingPass123!"
gh secret set DB_PASSWORD --env production --body "ProdPass456!"
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      dotnet-version:
        required: false
        type: string
        default: '9.0.x'
      test-project:
        required: true
        type: string
    secrets:
      db-password:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ inputs.dotnet-version }}
    - name: Run tests
      run: dotnet test ${{ inputs.test-project }}
      env:
        DB_PASSWORD: ${{ secrets.db-password }}

# ใช้งาน reusable workflow
# .github/workflows/ci.yml
jobs:
  unit-tests:
    uses: ./.github/workflows/reusable-test.yml
    with:
      test-project: tests/MyApp.UnitTests/
  
  integration-tests:
    uses: ./.github/workflows/reusable-test.yml
    with:
      test-project: tests/MyApp.IntegrationTests/
    secrets:
      db-password: ${{ secrets.TEST_DB_PASSWORD }}
```

---

## Exercises / Project Tasks

### Exercise 1: Basic CI
สร้าง CI workflow ที่:
- Build และ test ทุก push
- Report test coverage
- Fail ถ้า coverage < 80%

### Exercise 2: Docker CI
เพิ่มขั้นตอน Docker:
- Build Docker image ทุก PR
- Push ไป registry เมื่อ merge to main
- Scan สำหรับ vulnerabilities

### Exercise 3: Full Pipeline
สร้าง pipeline ครบ:
- Build → Test → Docker → Deploy Staging → Deploy Production
- Environment protection rules
- Notification เมื่อ fail

### Exercise 4: Release Management
สร้าง release workflow:
- Triggered โดย git tag
- Generate changelog
- Create GitHub release
- Deploy to production

---

## สรุป

- **CI** ทดสอบ code ทุกครั้งที่มี push ป้องกัน integration issues
- **CD** deploy อัตโนมัติลด manual errors และเพิ่มความเร็ว
- **GitHub Actions** ใช้ YAML ที่อ่านง่าย มี marketplace actions มากมาย
- **Services** ใน workflow ช่วยรัน dependencies เช่น database สำหรับ integration tests
- **Environments** ควบคุมว่า workflow ต้อง approve ก่อน deploy
- **Secrets** เก็บ sensitive data ปลอดภัย
- **Reusable workflows** ลด duplication ระหว่าง workflows

---

## Part ถัดไป

**Part 086: Azure Fundamentals กับ .NET** - เรียนรู้บริการสำคัญของ Azure และการใช้งานกับ .NET

---

*Part 085/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
