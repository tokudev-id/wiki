---
title: Error Handling dan Logging
description: Standar penanganan error dan logging di aplikasi .NET menggunakan Serilog, mencakup exception handling, structured logging, dan praktik terbaik untuk monitoring aplikasi production.
published: true
date: 2025-12-29T00:00:00.000Z
tags: logging, error-handling, serilog
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Error Handling dan Logging

Dokumen ini menjelaskan standar penanganan error dan logging di aplikasi .NET. Logging yang baik adalah kunci untuk debugging dan monitoring aplikasi di production.

## Prinsip Error Handling

### 1. Fail Fast

Validasi input sedini mungkin dan gagalkan dengan pesan yang jelas.

```csharp
public async Task<PelangganDto> BuatAsync(BuatPelangganRequest request)
{
    // Validasi di awal - fail fast
    if (string.IsNullOrWhiteSpace(request.Nama))
        throw new ValidationException("Nama wajib diisi");

    if (string.IsNullOrWhiteSpace(request.Email))
        throw new ValidationException("Email wajib diisi");

    // Lanjutkan jika valid
    // ...
}
```

### 2. Gunakan Exception yang Tepat

```csharp
// Hierarchy exception yang direkomendasikan
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception inner) : base(message, inner) { }
}

public class NotFoundException : DomainException
{
    public NotFoundException(string message) : base(message) { }

    public static NotFoundException ForEntity<T>(object id) =>
        new($"{typeof(T).Name} dengan ID '{id}' tidak ditemukan");
}

public class ValidationException : DomainException
{
    public IReadOnlyDictionary<string, string[]> Errors { get; }

    public ValidationException(string message) : base(message)
    {
        Errors = new Dictionary<string, string[]>();
    }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("Satu atau lebih kesalahan validasi terjadi")
    {
        Errors = new Dictionary<string, string[]>(errors);
    }
}

public class ConflictException : DomainException
{
    public ConflictException(string message) : base(message) { }
}

public class ForbiddenException : DomainException
{
    public ForbiddenException(string message) : base(message) { }
}

public class BusinessRuleException : DomainException
{
    public BusinessRuleException(string message) : base(message) { }
}
```

### 3. Jangan Sembunyikan Exception

```csharp
// BURUK - Menyembunyikan exception
try
{
    await ProsesDataAsync();
}
catch (Exception)
{
    // Diam-diam gagal, tidak ada yang tahu
}

// BURUK - Catch semua lalu return null
try
{
    return await DapatkanDataAsync();
}
catch (Exception)
{
    return null;  // Caller tidak tahu ada error
}

// BAIK - Tangkap spesifik, log, dan rethrow atau handle dengan benar
try
{
    return await DapatkanDataAsync();
}
catch (SqlException ex) when (ex.Number == 1205) // Deadlock
{
    _logger.LogWarning(ex, "Deadlock terdeteksi, mencoba ulang...");
    throw new RetryableException("Database deadlock", ex);
}
catch (SqlException ex)
{
    _logger.LogError(ex, "Database error saat mengambil data");
    throw;  // Rethrow untuk ditangani di level atas
}
```

## Global Exception Handling

### Exception Handler Middleware

```csharp
// Middleware/GlobalExceptionHandler.cs
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    private readonly IHostEnvironment _env;

    public GlobalExceptionHandler(
        ILogger<GlobalExceptionHandler> logger,
        IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (statusCode, title, detail) = MapException(exception);

        _logger.LogError(exception,
            "Exception terjadi: {ExceptionType} - {Message}",
            exception.GetType().Name,
            exception.Message);

        httpContext.Response.StatusCode = statusCode;
        httpContext.Response.ContentType = "application/problem+json";

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = detail,
            Instance = httpContext.Request.Path
        };

        // Tambahkan stack trace di development
        if (_env.IsDevelopment())
        {
            problemDetails.Extensions["stackTrace"] = exception.StackTrace;
        }

        // Tambahkan validation errors jika ada
        if (exception is ValidationException validationEx)
        {
            problemDetails.Extensions["errors"] = validationEx.Errors;
        }

        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }

    private static (int StatusCode, string Title, string Detail) MapException(Exception exception)
    {
        return exception switch
        {
            ValidationException => (400, "Validasi Gagal", exception.Message),
            NotFoundException => (404, "Tidak Ditemukan", exception.Message),
            ForbiddenException => (403, "Akses Ditolak", exception.Message),
            ConflictException => (409, "Konflik", exception.Message),
            UnauthorizedAccessException => (401, "Tidak Terautentikasi", "Silakan login terlebih dahulu"),
            BusinessRuleException => (422, "Aturan Bisnis Dilanggar", exception.Message),
            OperationCanceledException => (499, "Request Dibatalkan", "Client membatalkan request"),
            _ => (500, "Kesalahan Internal", "Terjadi kesalahan yang tidak terduga")
        };
    }
}

// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
```

## Logging dengan Serilog

### Setup Serilog

```csharp
// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/log-.txt",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30)
    .WriteTo.Seq("http://localhost:5341")  // Seq server
    .CreateLogger();

try
{
    Log.Information("Aplikasi dimulai");

    var builder = WebApplication.CreateBuilder(args);
    builder.Host.UseSerilog();

    // ... konfigurasi lain

    var app = builder.Build();
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
        };
    });

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Aplikasi gagal dimulai");
}
finally
{
    Log.CloseAndFlush();
}
```

### appsettings.json untuk Serilog

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File", "Serilog.Sinks.Seq"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "theme": "Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme::Code, Serilog.Sinks.Console"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      },
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "http://localhost:5341"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithEnvironmentName"]
  }
}
```

## Structured Logging Best Practices

### Jangan Gunakan String Interpolation

```csharp
// BURUK - String interpolation
_logger.LogInformation($"Pelanggan {pelanggan.Id} berhasil dibuat oleh {userId}");

// BAIK - Message template dengan parameter
_logger.LogInformation(
    "Pelanggan {PelangganId} berhasil dibuat oleh {UserId}",
    pelanggan.Id,
    userId);
```

### Gunakan Log Level yang Tepat

```csharp
public class PelangganService
{
    private readonly ILogger<PelangganService> _logger;

    public async Task<PelangganDto> BuatAsync(BuatPelangganRequest request)
    {
        // TRACE - Detail untuk debugging mendalam
        _logger.LogTrace("Memulai BuatAsync dengan request: {@Request}", request);

        // DEBUG - Informasi debugging
        _logger.LogDebug("Memeriksa apakah email {Email} sudah terdaftar", request.Email);

        // INFORMATION - Alur bisnis normal
        _logger.LogInformation("Pelanggan baru {Email} berhasil didaftarkan dengan ID {PelangganId}",
            request.Email, pelanggan.Id);

        // WARNING - Situasi tidak normal tapi bukan error
        _logger.LogWarning("Percobaan login gagal untuk {Email}, percobaan ke-{Attempt}",
            email, attemptCount);

        // ERROR - Error yang ditangani
        _logger.LogError(exception, "Gagal mengirim email konfirmasi ke {Email}", email);

        // CRITICAL - Error fatal yang menyebabkan sistem berhenti
        _logger.LogCritical(exception, "Koneksi database terputus, aplikasi akan berhenti");
    }
}
```

### Log Levels Guidelines

| Level | Kapan Digunakan | Contoh |
|-------|-----------------|--------|
| **Trace** | Detail debugging paling rinci | Masuk/keluar method, nilai variabel |
| **Debug** | Info debugging untuk development | Query yang dieksekusi, cache hit/miss |
| **Information** | Alur bisnis normal | User login, pesanan dibuat, payment berhasil |
| **Warning** | Situasi tidak normal | Retry terjadi, resource hampir habis |
| **Error** | Error yang ditangani | Gagal kirim email, API timeout |
| **Critical** | Error fatal | Database down, out of memory |

### Enrichment dan Context

```csharp
// Tambahkan context ke semua log dalam scope
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["OrderId"] = order.Id,
    ["CustomerId"] = customer.Id,
    ["CorrelationId"] = correlationId
}))
{
    _logger.LogInformation("Memproses pesanan");
    await ProcessOrderAsync(order);
    _logger.LogInformation("Pesanan selesai diproses");
}

// Semua log dalam scope akan memiliki OrderId, CustomerId, CorrelationId
```

### Request Correlation

```csharp
// Middleware untuk correlation ID
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationIdHeader = "X-Correlation-ID";

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers[CorrelationIdHeader] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}
```

## Sensitive Data Protection

### Jangan Log Data Sensitif

```csharp
// BURUK - Log data sensitif
_logger.LogInformation("User login: {Email}, Password: {Password}", email, password);
_logger.LogDebug("Credit card: {CardNumber}", cardNumber);

// BAIK - Mask atau jangan log sama sekali
_logger.LogInformation("User login: {Email}", email);
_logger.LogDebug("Payment processed for card ending in {LastFour}", cardNumber[^4..]);
```

### Destructuring dengan Filtering

```csharp
// Konfigurasi Serilog untuk filter sensitive data
Log.Logger = new LoggerConfiguration()
    .Destructure.ByTransforming<BuatPelangganRequest>(r => new
    {
        r.Nama,
        r.Email,
        Password = "***REDACTED***"
    })
    .CreateLogger();
```

## Error Response Format

### Gunakan Problem Details (RFC 7807)

```csharp
// Response untuk validation error
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Validasi Gagal",
  "status": 400,
  "detail": "Satu atau lebih kesalahan validasi terjadi",
  "instance": "/api/pelanggan",
  "errors": {
    "Email": ["Email wajib diisi", "Format email tidak valid"],
    "Nama": ["Nama wajib diisi"]
  }
}

// Response untuk not found
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Tidak Ditemukan",
  "status": 404,
  "detail": "Pelanggan dengan ID '123' tidak ditemukan",
  "instance": "/api/pelanggan/123"
}

// Response untuk server error (production)
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
  "title": "Kesalahan Internal",
  "status": 500,
  "detail": "Terjadi kesalahan yang tidak terduga. Silakan coba lagi nanti.",
  "instance": "/api/pesanan",
  "traceId": "00-abc123-def456-00"
}
```

## Logging di Production

### Apa yang Harus Di-log

| Kategori | Log? | Level | Contoh |
|----------|------|-------|--------|
| Request masuk | Ya | Info | HTTP GET /api/pelanggan |
| Request selesai | Ya | Info | 200 OK in 45ms |
| Business events | Ya | Info | Pesanan #123 dibuat |
| Errors | Ya | Error | Gagal proses pembayaran |
| Security events | Ya | Warning | Login gagal 5x |
| Performance anomalies | Ya | Warning | Query > 5 detik |
| Debug info | Tidak* | Debug | Nilai variabel |
| PII/Sensitive | Tidak | - | Password, CC number |

*Di production, set minimum level ke Information

### Log Aggregation

```yaml
# docker-compose.yml untuk logging stack
version: "3.8"

services:
  seq:
    image: datalust/seq:latest
    environment:
      ACCEPT_EULA: Y
    ports:
      - "5341:80"
    volumes:
      - seq-data:/data

  # Alternatif: ELK Stack
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"

volumes:
  seq-data:
```

## Checklist Error Handling

- [ ] Semua exception custom mewarisi dari base exception
- [ ] Global exception handler terkonfigurasi
- [ ] Tidak ada empty catch blocks
- [ ] Tidak ada log data sensitif
- [ ] Structured logging digunakan (bukan string interpolation)
- [ ] Correlation ID ada di setiap request
- [ ] Log level sesuai dengan severity
- [ ] Problem Details format digunakan untuk error response
- [ ] Log aggregation terkonfigurasi di production

## Referensi

- [Serilog Documentation](https://serilog.net/)
- [Logging in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/)
- [RFC 7807 - Problem Details](https://tools.ietf.org/html/rfc7807)
- [Seq Documentation](https://docs.datalust.co/docs)

---

*Terakhir diperbarui: Desember 2025*
