# CI/CD dan DevOps

Dokumen ini menjelaskan praktik CI/CD dan DevOps untuk proyek .NET di Unictive.

## Docker

### Dockerfile untuk .NET

```dockerfile
# Dockerfile
# Multi-stage build untuk ukuran image minimal

# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj dan restore (untuk caching layer)
COPY ["src/API/API.csproj", "API/"]
COPY ["src/Application/Application.csproj", "Application/"]
COPY ["src/Domain/Domain.csproj", "Domain/"]
COPY ["src/Infrastructure/Infrastructure.csproj", "Infrastructure/"]
RUN dotnet restore "API/API.csproj"

# Copy semua source dan build
COPY src/ .
WORKDIR /src/API
RUN dotnet build "API.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "API.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Security: Run as non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# Copy published app
COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Expose port
EXPOSE 8080

# Set environment
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

ENTRYPOINT ["dotnet", "API.dll"]
```

### Docker Compose untuk Development

```yaml
# docker-compose.yml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=AppDb;User=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true
      - Redis__ConnectionString=redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "YourStrong@Passw0rd" -Q "SELECT 1"
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network

  seq:
    image: datalust/seq:latest
    environment:
      - ACCEPT_EULA=Y
    ports:
      - "5341:80"
    volumes:
      - seq-data:/data
    networks:
      - app-network

volumes:
  sqlserver-data:
  redis-data:
  seq-data:

networks:
  app-network:
    driver: bridge
```

### Docker Build vs Compose

| Aspek | `docker build` | `docker-compose` |
|-------|----------------|------------------|
| Tujuan | Build single image | Orchestrate multiple containers |
| Use case | CI/CD build | Local development, testing |
| Network | Manual setup | Automatic network creation |
| Dependencies | Manual ordering | `depends_on` directive |
| Environment | Build-time args | Runtime environment |

```bash
# Build image saja
docker build -t myapp:latest .

# Build dan jalankan dengan compose
docker-compose up --build

# Jalankan di background
docker-compose up -d

# Stop dan hapus containers
docker-compose down

# Stop, hapus containers dan volumes
docker-compose down -v
```

## CI/CD Pipeline

### Bitbucket Pipelines

```yaml
# bitbucket-pipelines.yml
image: mcr.microsoft.com/dotnet/sdk:8.0

definitions:
  caches:
    dotnet: ~/.nuget/packages

  steps:
    - step: &build-test
        name: Build dan Test
        caches:
          - dotnet
        script:
          - dotnet restore
          - dotnet build --no-restore --configuration Release
          - dotnet test --no-build --configuration Release --logger "trx;LogFileName=test-results.trx" --collect:"XPlat Code Coverage"
        artifacts:
          - "**/TestResults/**"

    - step: &security-scan
        name: Security Scan
        script:
          - dotnet tool install --global security-scan
          - security-scan ./src --export=report.json
        artifacts:
          - report.json

    - step: &build-docker
        name: Build Docker Image
        services:
          - docker
        script:
          - docker build -t $DOCKER_REGISTRY/$BITBUCKET_REPO_SLUG:$BITBUCKET_COMMIT .
          - docker push $DOCKER_REGISTRY/$BITBUCKET_REPO_SLUG:$BITBUCKET_COMMIT

    - step: &deploy-staging
        name: Deploy to Staging
        deployment: staging
        script:
          - pipe: atlassian/ssh-run:0.4.1
            variables:
              SSH_USER: $STAGING_SSH_USER
              SERVER: $STAGING_SERVER
              COMMAND: |
                cd /opt/app
                docker-compose pull
                docker-compose up -d

    - step: &deploy-production
        name: Deploy to Production
        deployment: production
        trigger: manual
        script:
          - pipe: atlassian/ssh-run:0.4.1
            variables:
              SSH_USER: $PROD_SSH_USER
              SERVER: $PROD_SERVER
              COMMAND: |
                cd /opt/app
                docker-compose pull
                docker-compose up -d

pipelines:
  default:
    - step: *build-test

  branches:
    develop:
      - step: *build-test
      - step: *security-scan
      - step: *build-docker
      - step: *deploy-staging

    main:
      - step: *build-test
      - step: *security-scan
      - step: *build-docker
      - step: *deploy-staging
      - step: *deploy-production

  pull-requests:
    "**":
      - step: *build-test
      - step: *security-scan
```

### GitHub Actions (Alternatif)

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  DOTNET_VERSION: "8.0.x"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v3
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --verbosity normal --collect:"XPlat Code Coverage"

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: "**/coverage.cobertura.xml"

  security-scan:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - uses: actions/checkout@v4

      - name: Run security scan
        uses: security-scan/action@v1

  build-docker:
    runs-on: ubuntu-latest
    needs: [build-test, security-scan]
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-docker
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - name: Deploy to Staging
        run: |
          # Deploy script

  deploy-production:
    runs-on: ubuntu-latest
    needs: build-docker
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Deploy to Production
        run: |
          # Deploy script
```

## CI/CD Checklist

### Pre-Deployment Checklist

```markdown
## Sebelum Deploy ke Staging

- [ ] Semua unit tests passing
- [ ] Semua integration tests passing
- [ ] Code coverage >= 80%
- [ ] No critical security vulnerabilities
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Health check endpoint working

## Sebelum Deploy ke Production

- [ ] Staging sudah ditest minimal 24 jam
- [ ] Load testing completed (jika ada perubahan signifikan)
- [ ] Rollback plan documented
- [ ] Database backup verified
- [ ] Stakeholders notified
- [ ] Monitoring alerts configured
- [ ] On-call engineer assigned
```

### Deployment Strategies

#### Rolling Update

```yaml
# kubernetes deployment dengan rolling update
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

#### Blue-Green Deployment

```bash
# 1. Deploy ke green environment
docker-compose -f docker-compose.green.yml up -d

# 2. Test green environment
curl http://green.internal/health

# 3. Switch traffic (nginx/load balancer)
# Update upstream dari blue ke green

# 4. Stop blue setelah verified
docker-compose -f docker-compose.blue.yml down
```

## Container Guidelines

### Best Practices

```dockerfile
# 1. Gunakan specific version, bukan 'latest'
FROM mcr.microsoft.com/dotnet/aspnet:8.0.1 AS base

# 2. Multi-stage build untuk ukuran minimal
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
# ... build steps
FROM base AS final
COPY --from=build /app .

# 3. Jalankan sebagai non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# 4. Gunakan .dockerignore
# .dockerignore:
# **/.git
# **/bin
# **/obj
# **/.vs
# **/*.user
# **/node_modules

# 5. Set resource limits
# docker-compose.yml:
# deploy:
#   resources:
#     limits:
#       cpus: '2'
#       memory: 2G

# 6. Health checks
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1
```

### Security Checklist untuk Containers

- [ ] Base image dari trusted source
- [ ] Tidak ada secrets di image (gunakan environment variables)
- [ ] Run as non-root user
- [ ] Minimal packages installed
- [ ] Regular image scanning (Trivy, Snyk)
- [ ] Read-only filesystem jika memungkinkan

## Release Management

### Semantic Versioning

```
MAJOR.MINOR.PATCH

1.0.0 -> 1.0.1  (patch: bug fix)
1.0.1 -> 1.1.0  (minor: new feature, backward compatible)
1.1.0 -> 2.0.0  (major: breaking changes)
```

### Git Tags untuk Release

```bash
# Tag release
git tag -a v1.2.0 -m "Release 1.2.0: Add payment feature"
git push origin v1.2.0

# List tags
git tag -l "v1.*"
```

### Release Notes Template

```markdown
# Release v1.2.0

## Fitur Baru
- Tambah integrasi pembayaran dengan Midtrans
- Dashboard analitik untuk admin

## Perbaikan
- Fix validasi email yang salah
- Perbaiki performance query pelanggan

## Breaking Changes
- API endpoint `/api/v1/orders` diganti menjadi `/api/v2/orders`
- Field `customer_id` sekarang wajib di create order

## Migrasi
1. Jalankan migrasi database: `dotnet ef database update`
2. Update environment variable: `PAYMENT_GATEWAY_KEY`

## Known Issues
- Laporan PDF membutuhkan waktu lama untuk data besar
```

## Referensi

- [Docker Documentation](https://docs.docker.com/)
- [Bitbucket Pipelines](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-bitbucket-pipelines/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [.NET Docker Images](https://hub.docker.com/_/microsoft-dotnet)

---

*Terakhir diperbarui: Desember 2025*
