# Wiki Unictive

Proyek Wiki menggunakan Wiki.js untuk dokumentasi standar pengembangan .NET.

## Prasyarat

- Docker dan Docker Compose
- Minimal 1GB RAM untuk container

## Cara Menjalankan

### 1. Konfigurasi Environment

```bash
cp .env.example .env
```

Edit file `.env` dan sesuaikan kata sandi database jika diperlukan.

### 2. Jalankan Container

```bash
docker compose up -d
```

### 3. Akses Wiki.js

Buka browser dan akses: http://localhost:3000

Pada akses pertama, Anda akan diminta untuk:
1. Membuat akun administrator
2. Mengonfigurasi pengaturan dasar wiki

## Mengimpor Dokumentasi

Setelah Wiki.js berjalan, Anda dapat mengimpor dokumentasi dari folder `docs/`:

1. Login ke Wiki.js sebagai administrator
2. Buat halaman baru untuk setiap dokumen
3. Salin konten dari file markdown

### Daftar Dokumentasi

| Kategori | File | Deskripsi |
|----------|------|-----------|
| **Beranda** | `beranda.md` | Halaman utama dokumentasi |
| **Onboarding** | `onboarding-guide.md` | Panduan untuk pengembang baru |
| **Standar Kode** | `standar-kode-dotnet.md` | Standar penulisan kode .NET |
| **Konvensi** | `konvensi-penamaan.md` | Konvensi penamaan variabel, kelas, dll |
| **API** | `best-practice-api-service.md` | Praktik terbaik API Service |
| **Frontend** | `blazor-razor-pages.md` | Panduan Blazor dan Razor Pages |
| **Arsitektur** | `repository-service-pattern.md` | Pola Repository dan Service |
| **DDD** | `architecture-ddd.md` | Domain-Driven Design, Clean Architecture |
| **Database** | `database-caching.md` | EF Core, indexing, Redis caching |
| **Testing** | `testing-guidelines.md` | xUnit, integration tests, mocking |
| **Logging** | `error-handling-logging.md` | Error handling, Serilog |
| **Security** | `security-awareness.md` | OWASP Top 10, keamanan aplikasi |
| **Resilience** | `resilience-patterns.md` | Polly, Circuit Breaker, Retry |
| **CI/CD** | `cicd-devops.md` | Docker, pipelines, deployment |
| **Observability** | `observability.md` | OpenTelemetry, monitoring |

## Perintah Docker

```bash
# Menjalankan container
docker compose up -d

# Melihat log
docker compose logs -f

# Menghentikan container
docker compose down

# Menghentikan dan menghapus volume (reset data)
docker compose down -v

# Update image Wiki.js
docker compose pull wiki
docker compose up --force-recreate -d
```

## Backup Database

```bash
# Backup
docker exec wiki-db pg_dump -U wikijs wiki > backup.sql

# Restore
docker exec -i wiki-db psql -U wikijs wiki < backup.sql
```

## Struktur Proyek

```
Wiki/
├── docker-compose.yml    # Konfigurasi Docker Compose
├── .env.example          # Template environment variables
├── README.md             # Dokumentasi proyek ini
└── docs/                 # File dokumentasi markdown (15 files)
    ├── beranda.md                    # Halaman utama
    ├── onboarding-guide.md           # Panduan onboarding
    ├── standar-kode-dotnet.md        # Standar kode
    ├── konvensi-penamaan.md          # Konvensi penamaan
    ├── best-practice-api-service.md  # API best practices
    ├── blazor-razor-pages.md         # Blazor & Razor
    ├── repository-service-pattern.md # Repository pattern
    ├── architecture-ddd.md           # Architecture & DDD
    ├── database-caching.md           # Database & caching
    ├── testing-guidelines.md         # Testing
    ├── error-handling-logging.md     # Error handling
    ├── security-awareness.md         # Security
    ├── resilience-patterns.md        # Resilience
    ├── cicd-devops.md                # CI/CD
    └── observability.md              # Observability
```

## Referensi

- [Wiki.js Documentation](https://docs.requarks.io/)
- [Docker Documentation](https://docs.docker.com/)
- [Microsoft .NET Documentation](https://docs.microsoft.com/en-us/dotnet/)
