# Dokumentasi Standar Pengembangan .NET

Selamat datang di dokumentasi standar pengembangan .NET Unictive. Dokumentasi ini berisi panduan lengkap mengenai standar kode, konvensi penamaan, dan praktik terbaik dalam pengembangan aplikasi menggunakan teknologi .NET.

## Daftar Isi

### Onboarding & Tim
- [Panduan Onboarding](./onboarding-guide.md) - Panduan untuk pengembang baru

### Standar Kode & Konvensi
- [Standar Kode .NET](./standar-kode-dotnet.md) - Prinsip penulisan kode
- [Konvensi Penamaan](./konvensi-penamaan.md) - Aturan penamaan variabel, kelas, dll

### Pengembangan Backend
- [Praktik Terbaik API Service](./best-practice-api-service.md) - RESTful API, validasi, pagination
- [Blazor dan Razor Pages](./blazor-razor-pages.md) - Frontend dengan .NET
- [Pola Repository dan Service](./repository-service-pattern.md) - Arsitektur data layer
- [Arsitektur dan DDD](./architecture-ddd.md) - Domain-Driven Design, Clean Architecture
- [Database dan Caching](./database-caching.md) - EF Core, indexing, Redis

### Testing & Kualitas
- [Panduan Testing](./testing-guidelines.md) - xUnit, integration tests, mocking
- [Error Handling dan Logging](./error-handling-logging.md) - Penanganan error, Serilog

### Keamanan & Ketahanan
- [Kesadaran Keamanan](./security-awareness.md) - OWASP Top 10, best practices
- [Pola Ketahanan](./resilience-patterns.md) - Polly, Circuit Breaker, Retry

### DevOps & Operasi
- [CI/CD dan DevOps](./cicd-devops.md) - Docker, pipelines, deployment
- [Observability](./observability.md) - OpenTelemetry, logging, monitoring

## Tujuan Dokumentasi

Dokumentasi ini bertujuan untuk:

- **Menjaga Konsistensi** - Memastikan seluruh tim pengembang mengikuti standar yang sama
- **Meningkatkan Keterbacaan** - Kode yang konsisten lebih mudah dibaca dan dipahami
- **Mempermudah Pemeliharaan** - Kode yang terstandar lebih mudah untuk dipelihara dan dikembangkan
- **Mempercepat Onboarding** - Anggota tim baru dapat lebih cepat memahami basis kode
- **Meningkatkan Keamanan** - Menghindari kerentanan umum dengan praktik yang benar
- **Memastikan Keandalan** - Aplikasi yang tahan terhadap kegagalan

## Teknologi yang Dicakup

- .NET 8 / .NET 9
- ASP.NET Core
- Blazor (Server dan WebAssembly)
- Razor Pages
- Entity Framework Core
- Web API
- Docker & Kubernetes
- OpenTelemetry
- Polly
- Redis

## Referensi

Dokumentasi ini disusun berdasarkan:

- [Microsoft .NET Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [.NET Runtime Coding Style](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md)
- [ASP.NET Core Engineering Guidelines](https://github.com/dotnet/aspnetcore/wiki/Engineering-guidelines)
- [OWASP Top 10](https://owasp.org/Top10/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Polly Documentation](https://www.pollydocs.org/)

---

*Terakhir diperbarui: Desember 2025*
