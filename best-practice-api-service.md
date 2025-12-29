---
title: Praktik Terbaik API Service
description: Panduan praktik terbaik membangun API Service dengan ASP.NET Core, mencakup arsitektur, keamanan, validasi, penanganan kesalahan, pagination, versioning, dan dokumentasi Swagger.
published: true
date: 2025-12-29T00:00:00.000Z
tags: api, rest, best-practices
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Praktik Terbaik API Service

Dokumen ini menjelaskan praktik terbaik dalam membangun API Service menggunakan ASP.NET Core. Panduan ini mencakup arsitektur, keamanan, validasi, penanganan kesalahan, dan optimisasi kinerja.

## Arsitektur API

### Struktur Proyek yang Direkomendasikan

```
src/
├── NamaProyek.API/
│   ├── Controllers/
│   ├── Filters/
│   ├── Middleware/
│   └── Program.cs
├── NamaProyek.Application/
│   ├── Commands/
│   ├── Queries/
│   ├── Services/
│   ├── DTOs/
│   └── Validators/
├── NamaProyek.Domain/
│   ├── Entities/
│   ├── Enums/
│   ├── Events/
│   └── Interfaces/
└── NamaProyek.Infrastructure/
    ├── Repositories/
    ├── Data/
    └── ExternalServices/
```

### Gunakan Minimal APIs atau Controller

#### Minimal APIs (Direkomendasikan untuk API Sederhana)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.MapGet("/api/pelanggan", async (IPelangganService service) =>
{
    var pelanggan = await service.DapatkanSemuaAsync();
    return Results.Ok(pelanggan);
});

app.MapGet("/api/pelanggan/{id}", async (int id, IPelangganService service) =>
{
    var pelanggan = await service.DapatkanByIdAsync(id);
    return pelanggan is null ? Results.NotFound() : Results.Ok(pelanggan);
});

app.MapPost("/api/pelanggan", async (BuatPelangganRequest request, IPelangganService service) =>
{
    var pelanggan = await service.BuatAsync(request);
    return Results.Created($"/api/pelanggan/{pelanggan.Id}", pelanggan);
});

app.Run();
```

#### Controller-Based APIs (Direkomendasikan untuk API Kompleks)

```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class PelangganController : ControllerBase
{
    private readonly IPelangganService _pelangganService;
    private readonly ILogger<PelangganController> _logger;

    public PelangganController(
        IPelangganService pelangganService,
        ILogger<PelangganController> logger)
    {
        _pelangganService = pelangganService;
        _logger = logger;
    }

    /// <summary>
    /// Mendapatkan daftar semua pelanggan.
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<PelangganDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<PelangganDto>>> DapatkanSemua(
        CancellationToken cancellationToken)
    {
        var pelanggan = await _pelangganService.DapatkanSemuaAsync(cancellationToken);
        return Ok(pelanggan);
    }

    /// <summary>
    /// Mendapatkan pelanggan berdasarkan ID.
    /// </summary>
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(PelangganDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<PelangganDto>> DapatkanById(
        int id,
        CancellationToken cancellationToken)
    {
        var pelanggan = await _pelangganService.DapatkanByIdAsync(id, cancellationToken);

        if (pelanggan is null)
        {
            return NotFound();
        }

        return Ok(pelanggan);
    }

    /// <summary>
    /// Membuat pelanggan baru.
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(PelangganDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<PelangganDto>> Buat(
        [FromBody] BuatPelangganRequest request,
        CancellationToken cancellationToken)
    {
        var pelanggan = await _pelangganService.BuatAsync(request, cancellationToken);
        return CreatedAtAction(nameof(DapatkanById), new { id = pelanggan.Id }, pelanggan);
    }
}
```

## Desain Endpoint RESTful

### Konvensi Penamaan URL

```csharp
// BAIK - Gunakan kata benda jamak
GET    /api/pelanggan           // Daftar pelanggan
GET    /api/pelanggan/{id}      // Detail pelanggan
POST   /api/pelanggan           // Buat pelanggan
PUT    /api/pelanggan/{id}      // Update pelanggan (penuh)
PATCH  /api/pelanggan/{id}      // Update pelanggan (parsial)
DELETE /api/pelanggan/{id}      // Hapus pelanggan

// Relasi nested
GET    /api/pelanggan/{id}/pesanan              // Pesanan milik pelanggan
POST   /api/pelanggan/{id}/pesanan              // Buat pesanan untuk pelanggan
GET    /api/pelanggan/{id}/pesanan/{pesananId}  // Detail pesanan

// BURUK
GET    /api/getPelanggan        // Jangan gunakan kata kerja di URL
POST   /api/createPelanggan     // HTTP method sudah menunjukkan aksi
GET    /api/pelanggan/getAll    // Redundan
```

### Kode Status HTTP yang Tepat

```csharp
public class PelangganController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> DapatkanSemua()
    {
        var data = await _service.DapatkanSemuaAsync();
        return Ok(data);  // 200 OK
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> DapatkanById(int id)
    {
        var data = await _service.DapatkanByIdAsync(id);
        if (data is null)
            return NotFound();  // 404 Not Found

        return Ok(data);  // 200 OK
    }

    [HttpPost]
    public async Task<IActionResult> Buat([FromBody] BuatRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);  // 400 Bad Request

        var hasil = await _service.BuatAsync(request);
        return CreatedAtAction(nameof(DapatkanById), new { id = hasil.Id }, hasil);  // 201 Created
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateRequest request)
    {
        var ada = await _service.AdaAsync(id);
        if (!ada)
            return NotFound();  // 404 Not Found

        await _service.UpdateAsync(id, request);
        return NoContent();  // 204 No Content
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Hapus(int id)
    {
        var ada = await _service.AdaAsync(id);
        if (!ada)
            return NotFound();  // 404 Not Found

        await _service.HapusAsync(id);
        return NoContent();  // 204 No Content
    }
}
```

## Validasi Input

### Gunakan FluentValidation

```csharp
// Validator
public class BuatPelangganRequestValidator : AbstractValidator<BuatPelangganRequest>
{
    public BuatPelangganRequestValidator()
    {
        RuleFor(x => x.Nama)
            .NotEmpty().WithMessage("Nama wajib diisi")
            .MaximumLength(100).WithMessage("Nama maksimal 100 karakter");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email wajib diisi")
            .EmailAddress().WithMessage("Format email tidak valid")
            .MaximumLength(255).WithMessage("Email maksimal 255 karakter");

        RuleFor(x => x.NomorTelepon)
            .Matches(@"^(\+62|62|0)[0-9]{9,12}$")
            .When(x => !string.IsNullOrEmpty(x.NomorTelepon))
            .WithMessage("Format nomor telepon tidak valid");

        RuleFor(x => x.TanggalLahir)
            .LessThan(DateTime.Today)
            .When(x => x.TanggalLahir.HasValue)
            .WithMessage("Tanggal lahir harus sebelum hari ini");
    }
}

// Registrasi di Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<BuatPelangganRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

### Data Annotations (Alternatif)

```csharp
public class BuatPelangganRequest
{
    [Required(ErrorMessage = "Nama wajib diisi")]
    [StringLength(100, ErrorMessage = "Nama maksimal 100 karakter")]
    public string Nama { get; set; } = string.Empty;

    [Required(ErrorMessage = "Email wajib diisi")]
    [EmailAddress(ErrorMessage = "Format email tidak valid")]
    [StringLength(255, ErrorMessage = "Email maksimal 255 karakter")]
    public string Email { get; set; } = string.Empty;

    [Phone(ErrorMessage = "Format nomor telepon tidak valid")]
    public string? NomorTelepon { get; set; }
}
```

## Penanganan Kesalahan Global

### Exception Handling Middleware

```csharp
public class GlobalExceptionHandlerMiddleware : IMiddleware
{
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;

    public GlobalExceptionHandlerMiddleware(ILogger<GlobalExceptionHandlerMiddleware> logger)
    {
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        try
        {
            await next(context);
        }
        catch (Exception exception)
        {
            _logger.LogError(exception, "Terjadi kesalahan yang tidak tertangani");
            await HandleExceptionAsync(context, exception);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/problem+json";

        var (statusCode, title, detail) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Tidak Ditemukan", exception.Message),
            ValidationException validationEx => (StatusCodes.Status400BadRequest, "Validasi Gagal", validationEx.Message),
            UnauthorizedAccessException => (StatusCodes.Status401Unauthorized, "Tidak Terotorisasi", "Akses ditolak"),
            ForbiddenException => (StatusCodes.Status403Forbidden, "Dilarang", exception.Message),
            ConflictException => (StatusCodes.Status409Conflict, "Konflik", exception.Message),
            _ => (StatusCodes.Status500InternalServerError, "Kesalahan Server", "Terjadi kesalahan internal")
        };

        context.Response.StatusCode = statusCode;

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = detail,
            Instance = context.Request.Path
        };

        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}

// Registrasi di Program.cs
builder.Services.AddTransient<GlobalExceptionHandlerMiddleware>();

var app = builder.Build();
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
```

### Custom Exceptions

```csharp
public abstract class ApplicationException : Exception
{
    protected ApplicationException(string message) : base(message) { }
}

public class NotFoundException : ApplicationException
{
    public NotFoundException(string message) : base(message) { }

    public static NotFoundException ForEntity<T>(object id)
        => new($"{typeof(T).Name} dengan ID '{id}' tidak ditemukan");
}

public class ValidationException : ApplicationException
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("Satu atau lebih kesalahan validasi terjadi")
    {
        Errors = errors;
    }
}

public class ConflictException : ApplicationException
{
    public ConflictException(string message) : base(message) { }
}

public class ForbiddenException : ApplicationException
{
    public ForbiddenException(string message) : base(message) { }
}
```

## Keamanan API

### Otentikasi JWT

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.Zero
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

### Otorisasi Berbasis Policy

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanManagePelanggan", policy =>
        policy.RequireClaim("permission", "pelanggan:manage"));

    options.AddPolicy("MinimumAge", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
});

// Controller
[Authorize(Policy = "CanManagePelanggan")]
[HttpPost]
public async Task<IActionResult> BuatPelanggan([FromBody] BuatPelangganRequest request)
{
    // ...
}
```

### Rate Limiting

```csharp
// Program.cs (.NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                QueueLimit = 0,
                Window = TimeSpan.FromMinutes(1)
            }));

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsync(
            "Terlalu banyak permintaan. Silakan coba lagi nanti.", token);
    };
});

var app = builder.Build();
app.UseRateLimiter();
```

### Validasi Input untuk Keamanan

```csharp
// Selalu sanitasi dan validasi input
public class PelangganService : IPelangganService
{
    public async Task<PelangganDto> BuatAsync(BuatPelangganRequest request)
    {
        // Sanitasi input HTML/script
        var namaBersih = HtmlEncoder.Default.Encode(request.Nama);

        // Validasi format
        if (!IsValidEmail(request.Email))
        {
            throw new ValidationException("Format email tidak valid");
        }

        // Gunakan parameterized queries (EF Core sudah melakukannya)
        var pelanggan = new Pelanggan
        {
            Nama = namaBersih,
            Email = request.Email.ToLowerInvariant()
        };

        // ...
    }
}
```

## Pagination dan Filtering

### Implementasi Pagination

```csharp
// Request
public class PagedRequest
{
    private const int MaxPageSize = 100;
    private int _pageSize = 10;

    public int PageNumber { get; set; } = 1;

    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = value > MaxPageSize ? MaxPageSize : value;
    }
}

// Response
public class PagedResponse<T>
{
    public IEnumerable<T> Data { get; set; } = Enumerable.Empty<T>();
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public int TotalRecords { get; set; }
    public bool HasPrevious => PageNumber > 1;
    public bool HasNext => PageNumber < TotalPages;
}

// Extension method
public static class QueryableExtensions
{
    public static async Task<PagedResponse<T>> ToPagedResponseAsync<T>(
        this IQueryable<T> query,
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        var totalRecords = await query.CountAsync(cancellationToken);
        var data = await query
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(cancellationToken);

        return new PagedResponse<T>
        {
            Data = data,
            PageNumber = pageNumber,
            PageSize = pageSize,
            TotalRecords = totalRecords,
            TotalPages = (int)Math.Ceiling(totalRecords / (double)pageSize)
        };
    }
}

// Penggunaan di Controller
[HttpGet]
public async Task<ActionResult<PagedResponse<PelangganDto>>> DapatkanSemua(
    [FromQuery] PagedRequest request,
    CancellationToken cancellationToken)
{
    var response = await _dbContext.Pelanggan
        .AsNoTracking()
        .OrderBy(p => p.Nama)
        .Select(p => new PelangganDto(p.Id, p.Nama, p.Email))
        .ToPagedResponseAsync(request.PageNumber, request.PageSize, cancellationToken);

    return Ok(response);
}
```

### Filtering dan Sorting

```csharp
public class PelangganFilter : PagedRequest
{
    public string? Nama { get; set; }
    public string? Email { get; set; }
    public bool? IsAktif { get; set; }
    public string? SortBy { get; set; }
    public bool SortDescending { get; set; }
}

// Service
public async Task<PagedResponse<PelangganDto>> DapatkanDenganFilterAsync(
    PelangganFilter filter,
    CancellationToken cancellationToken)
{
    var query = _dbContext.Pelanggan.AsNoTracking();

    // Apply filters
    if (!string.IsNullOrWhiteSpace(filter.Nama))
    {
        query = query.Where(p => p.Nama.Contains(filter.Nama));
    }

    if (!string.IsNullOrWhiteSpace(filter.Email))
    {
        query = query.Where(p => p.Email.Contains(filter.Email));
    }

    if (filter.IsAktif.HasValue)
    {
        query = query.Where(p => p.IsAktif == filter.IsAktif.Value);
    }

    // Apply sorting
    query = filter.SortBy?.ToLowerInvariant() switch
    {
        "nama" => filter.SortDescending
            ? query.OrderByDescending(p => p.Nama)
            : query.OrderBy(p => p.Nama),
        "email" => filter.SortDescending
            ? query.OrderByDescending(p => p.Email)
            : query.OrderBy(p => p.Email),
        "tanggaldibuat" => filter.SortDescending
            ? query.OrderByDescending(p => p.TanggalDibuat)
            : query.OrderBy(p => p.TanggalDibuat),
        _ => query.OrderBy(p => p.Id)
    };

    return await query
        .Select(p => new PelangganDto(p.Id, p.Nama, p.Email))
        .ToPagedResponseAsync(filter.PageNumber, filter.PageSize, cancellationToken);
}
```

## Versioning API

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
});

// Controller dengan versioning
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class PelangganController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> DapatkanSemua() { }
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class PelangganV2Controller : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> DapatkanSemua() { }
}
```

## Dokumentasi dengan Swagger

```csharp
// Program.cs
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "API Unictive",
        Version = "v1",
        Description = "API untuk aplikasi Unictive",
        Contact = new OpenApiContact
        {
            Name = "Tim Pengembang",
            Email = "dev@unictive.com"
        }
    });

    // Menambahkan dokumentasi dari XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);

    // Menambahkan autentikasi JWT
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "Masukkan token JWT dengan format: Bearer {token}",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

## Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>("database")
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!)
    .AddCheck<CustomHealthCheck>("custom");

var app = builder.Build();
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description
            })
        });
        await context.Response.WriteAsync(result);
    }
});
```

## Logging

### Jangan Gunakan String Interpolation untuk Log

```csharp
// BAIK - Menggunakan structured logging
_logger.LogInformation("Pelanggan {PelangganId} berhasil dibuat", pelanggan.Id);
_logger.LogError(exception, "Gagal memproses pesanan {PesananId}", pesanan.Id);

// BURUK - String interpolation
_logger.LogInformation($"Pelanggan {pelanggan.Id} berhasil dibuat");
```

### Gunakan Log Level yang Tepat

```csharp
// Trace - Detail debugging sangat rinci
_logger.LogTrace("Memasuki metode DapatkanPelanggan dengan id={Id}", id);

// Debug - Informasi debugging
_logger.LogDebug("Query pelanggan menghasilkan {Count} record", count);

// Information - Alur aplikasi normal
_logger.LogInformation("Pelanggan {PelangganId} berhasil didaftarkan", pelanggan.Id);

// Warning - Kejadian tidak normal tapi tidak error
_logger.LogWarning("Percobaan login gagal untuk email {Email}", email);

// Error - Error yang dapat ditangani
_logger.LogError(exception, "Gagal mengirim email ke {Email}", email);

// Critical - Error yang menyebabkan aplikasi gagal
_logger.LogCritical(exception, "Koneksi database terputus");
```

## Referensi

- [ASP.NET Core Web API Best Practices](https://learn.microsoft.com/en-us/aspnet/core/web-api/)
- [RESTful API Design Guidelines](https://restfulapi.net/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)

---

*Terakhir diperbarui: Desember 2025*
