# Database dan Caching

Dokumen ini menjelaskan praktik terbaik untuk pengelolaan database dan strategi caching di aplikasi .NET.

## Database dengan Entity Framework Core

### Konfigurasi DbContext

```csharp
// Infrastructure/Data/ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Pelanggan> Pelanggan => Set<Pelanggan>();
    public DbSet<Pesanan> Pesanan => Set<Pesanan>();
    public DbSet<Produk> Produk => Set<Produk>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Terapkan semua konfigurasi dari assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

        base.OnModelCreating(modelBuilder);
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // Enable sensitive data logging hanya di development
        #if DEBUG
        optionsBuilder.EnableSensitiveDataLogging();
        optionsBuilder.EnableDetailedErrors();
        #endif
    }
}

// Program.cs
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);

            sqlOptions.CommandTimeout(30);
            sqlOptions.MigrationsAssembly("Infrastructure");
        });
});
```

### Entity Configuration

```csharp
// Infrastructure/Data/Configurations/PelangganConfiguration.cs
public class PelangganConfiguration : IEntityTypeConfiguration<Pelanggan>
{
    public void Configure(EntityTypeBuilder<Pelanggan> builder)
    {
        builder.ToTable("Pelanggan");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Nama)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(p => p.Email)
            .IsRequired()
            .HasMaxLength(255);

        builder.Property(p => p.NomorTelepon)
            .HasMaxLength(20);

        // Index
        builder.HasIndex(p => p.Email)
            .IsUnique()
            .HasDatabaseName("IX_Pelanggan_Email");

        builder.HasIndex(p => p.IsAktif)
            .HasDatabaseName("IX_Pelanggan_IsAktif");

        // Relasi
        builder.HasMany(p => p.Pesanan)
            .WithOne(o => o.Pelanggan)
            .HasForeignKey(o => o.PelangganId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

## Database Indexing

### Kapan Membuat Index

| Gunakan Index Untuk | Jangan Index Untuk |
|---------------------|-------------------|
| Kolom di WHERE clause | Tabel kecil (<1000 baris) |
| Kolom di JOIN | Kolom yang sering di-UPDATE |
| Kolom di ORDER BY | Kolom dengan kardinalitas rendah |
| Foreign Keys | Kolom BLOB/TEXT |

### Tipe-tipe Index

```csharp
public class IndexConfiguration : IEntityTypeConfiguration<Pesanan>
{
    public void Configure(EntityTypeBuilder<Pesanan> builder)
    {
        // Single column index
        builder.HasIndex(p => p.NomorPesanan)
            .IsUnique();

        // Composite index (urutan penting!)
        builder.HasIndex(p => new { p.PelangganId, p.TanggalPesanan })
            .HasDatabaseName("IX_Pesanan_Pelanggan_Tanggal");

        // Filtered index (SQL Server)
        builder.HasIndex(p => p.Status)
            .HasFilter("[Status] != 5")  // Exclude 'Selesai'
            .HasDatabaseName("IX_Pesanan_Status_Active");

        // Include columns (covering index)
        builder.HasIndex(p => p.PelangganId)
            .IncludeProperties(p => new { p.NomorPesanan, p.Total })
            .HasDatabaseName("IX_Pesanan_Pelanggan_Include");
    }
}
```

### Analisis Query Performance

```sql
-- Lihat execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM Pelanggan WHERE Email = 'test@test.com';

-- Cari missing indexes (SQL Server)
SELECT
    dm_mid.database_id,
    dm_migs.avg_total_user_cost * (dm_migs.avg_user_impact / 100.0) *
        (dm_migs.user_seeks + dm_migs.user_scans) AS improvement_measure,
    'CREATE INDEX [IX_' + OBJECT_NAME(dm_mid.object_id, dm_mid.database_id) + '_'
        + REPLACE(REPLACE(REPLACE(ISNULL(dm_mid.equality_columns, ''),
            ', ', '_'), '[', ''), ']', '') + ']'
        + ' ON ' + dm_mid.statement
        + ' (' + ISNULL(dm_mid.equality_columns, '')
        + CASE WHEN dm_mid.equality_columns IS NOT NULL
            AND dm_mid.inequality_columns IS NOT NULL THEN ',' ELSE '' END
        + ISNULL(dm_mid.inequality_columns, '')
        + ')' + ISNULL(' INCLUDE (' + dm_mid.included_columns + ')', '') AS create_index_statement
FROM sys.dm_db_missing_index_groups dm_mig
INNER JOIN sys.dm_db_missing_index_group_stats dm_migs
    ON dm_migs.group_handle = dm_mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details dm_mid
    ON dm_mig.index_handle = dm_mid.index_handle
WHERE dm_mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;
```

### EF Core Query Optimization

```csharp
// BURUK - N+1 Query Problem
var pelanggan = await _context.Pelanggan.ToListAsync();
foreach (var p in pelanggan)
{
    var pesanan = p.Pesanan.Count; // Setiap iterasi = 1 query
}

// BAIK - Eager Loading
var pelanggan = await _context.Pelanggan
    .Include(p => p.Pesanan)
    .ToListAsync();

// BAIK - Projection (lebih efisien)
var pelangganDto = await _context.Pelanggan
    .Select(p => new PelangganDto
    {
        Id = p.Id,
        Nama = p.Nama,
        JumlahPesanan = p.Pesanan.Count
    })
    .ToListAsync();

// BAIK - Split Query untuk relasi banyak
var pelanggan = await _context.Pelanggan
    .Include(p => p.Pesanan)
        .ThenInclude(o => o.Items)
    .AsSplitQuery()  // Mencegah cartesian explosion
    .ToListAsync();

// Gunakan AsNoTracking untuk read-only
var pelanggan = await _context.Pelanggan
    .AsNoTracking()
    .Where(p => p.IsAktif)
    .ToListAsync();
```

## Strategi Caching

### Cache Levels

```
┌─────────────────────────────────────────────────┐
│              Application Request                │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│  L1: In-Memory Cache (IMemoryCache)             │
│  • Tercepat                                     │
│  • Per-instance                                 │
│  • Hilang saat restart                          │
└───────────────────────┬─────────────────────────┘
                        │ Miss
┌───────────────────────▼─────────────────────────┐
│  L2: Distributed Cache (Redis)                  │
│  • Cepat                                        │
│  • Shared antar instance                        │
│  • Persisten                                    │
└───────────────────────┬─────────────────────────┘
                        │ Miss
┌───────────────────────▼─────────────────────────┐
│  L3: Database                                   │
│  • Lambat                                       │
│  • Source of truth                              │
└─────────────────────────────────────────────────┘
```

### In-Memory Caching

```csharp
// Program.cs
builder.Services.AddMemoryCache();

// Service dengan caching
public class ProdukService : IProdukService
{
    private readonly IMemoryCache _cache;
    private readonly ApplicationDbContext _context;
    private const string CacheKeySemuaProduk = "produk:semua";

    public ProdukService(IMemoryCache cache, ApplicationDbContext context)
    {
        _cache = cache;
        _context = context;
    }

    public async Task<List<ProdukDto>> DapatkanSemuaAsync()
    {
        // Coba ambil dari cache
        if (_cache.TryGetValue(CacheKeySemuaProduk, out List<ProdukDto>? cachedProduk))
        {
            return cachedProduk!;
        }

        // Tidak ada di cache, ambil dari database
        var produk = await _context.Produk
            .AsNoTracking()
            .Select(p => new ProdukDto(p.Id, p.Nama, p.Harga))
            .ToListAsync();

        // Simpan ke cache
        var cacheOptions = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(5))
            .SetAbsoluteExpiration(TimeSpan.FromHours(1))
            .SetPriority(CacheItemPriority.Normal);

        _cache.Set(CacheKeySemuaProduk, produk, cacheOptions);

        return produk;
    }

    public async Task<ProdukDto> UpdateAsync(int id, UpdateProdukRequest request)
    {
        // Update database
        var produk = await _context.Produk.FindAsync(id);
        // ... update logic

        // Invalidate cache
        _cache.Remove(CacheKeySemuaProduk);
        _cache.Remove($"produk:{id}");

        return new ProdukDto(produk.Id, produk.Nama, produk.Harga);
    }
}
```

### Distributed Caching dengan Redis

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "Unictive:";
});

// Service dengan Redis
public class ProdukCacheService : IProdukCacheService
{
    private readonly IDistributedCache _cache;
    private readonly IProdukRepository _repository;
    private readonly JsonSerializerOptions _jsonOptions;

    public ProdukCacheService(IDistributedCache cache, IProdukRepository repository)
    {
        _cache = cache;
        _repository = repository;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }

    public async Task<ProdukDto?> DapatkanByIdAsync(int id, CancellationToken ct = default)
    {
        var cacheKey = $"produk:{id}";

        // Coba ambil dari cache
        var cachedData = await _cache.GetStringAsync(cacheKey, ct);
        if (cachedData is not null)
        {
            return JsonSerializer.Deserialize<ProdukDto>(cachedData, _jsonOptions);
        }

        // Tidak ada di cache
        var produk = await _repository.DapatkanByIdAsync(id, ct);
        if (produk is null)
        {
            return null;
        }

        var dto = new ProdukDto(produk.Id, produk.Nama, produk.Harga);

        // Simpan ke cache
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration = TimeSpan.FromMinutes(10)
        };

        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(dto, _jsonOptions),
            options,
            ct);

        return dto;
    }

    public async Task InvalidateAsync(int id, CancellationToken ct = default)
    {
        await _cache.RemoveAsync($"produk:{id}", ct);
    }
}
```

### Cache-Aside Pattern dengan Extension

```csharp
public static class DistributedCacheExtensions
{
    public static async Task<T?> GetOrSetAsync<T>(
        this IDistributedCache cache,
        string key,
        Func<Task<T?>> factory,
        DistributedCacheEntryOptions? options = null,
        CancellationToken ct = default) where T : class
    {
        // Coba ambil dari cache
        var cachedData = await cache.GetStringAsync(key, ct);
        if (cachedData is not null)
        {
            return JsonSerializer.Deserialize<T>(cachedData);
        }

        // Tidak ada di cache, panggil factory
        var data = await factory();
        if (data is null)
        {
            return null;
        }

        // Simpan ke cache
        options ??= new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
        };

        await cache.SetStringAsync(
            key,
            JsonSerializer.Serialize(data),
            options,
            ct);

        return data;
    }
}

// Penggunaan
public async Task<ProdukDto?> DapatkanByIdAsync(int id)
{
    return await _cache.GetOrSetAsync(
        $"produk:{id}",
        async () => await _repository.DapatkanByIdAsync(id),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
        });
}
```

### Response Caching

```csharp
// Program.cs
builder.Services.AddResponseCaching();

var app = builder.Build();
app.UseResponseCaching();

// Controller
[ApiController]
[Route("api/[controller]")]
public class ProdukController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "kategori", "page" })]
    public async Task<IActionResult> DapatkanSemua([FromQuery] string? kategori, [FromQuery] int page = 1)
    {
        // Response akan di-cache selama 60 detik
        var produk = await _service.DapatkanSemuaAsync(kategori, page);
        return Ok(produk);
    }

    [HttpGet("{id}")]
    [ResponseCache(Duration = 300, VaryByHeader = "Accept-Language")]
    public async Task<IActionResult> DapatkanById(int id)
    {
        var produk = await _service.DapatkanByIdAsync(id);
        return produk is null ? NotFound() : Ok(produk);
    }
}
```

### Output Caching (.NET 7+)

```csharp
// Program.cs
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder =>
        builder.Expire(TimeSpan.FromMinutes(5)));

    options.AddPolicy("ProdukCache", builder =>
        builder.Expire(TimeSpan.FromMinutes(30))
               .Tag("produk"));

    options.AddPolicy("ShortCache", builder =>
        builder.Expire(TimeSpan.FromSeconds(30)));
});

var app = builder.Build();
app.UseOutputCache();

// Controller
[HttpGet]
[OutputCache(PolicyName = "ProdukCache")]
public async Task<IActionResult> DapatkanSemua()
{
    var produk = await _service.DapatkanSemuaAsync();
    return Ok(produk);
}

// Invalidate cache by tag
[HttpPost]
public async Task<IActionResult> Buat(BuatProdukRequest request)
{
    var produk = await _service.BuatAsync(request);

    // Invalidate semua cache dengan tag "produk"
    await _outputCacheStore.EvictByTagAsync("produk", default);

    return CreatedAtAction(nameof(DapatkanById), new { id = produk.Id }, produk);
}
```

## Kapan Menggunakan Cache

### Kandidat yang Baik untuk Caching

| Data | Alasan | TTL Recommended |
|------|--------|-----------------|
| Konfigurasi/Settings | Jarang berubah | 1 jam - 24 jam |
| Data master (kategori, provinsi) | Jarang berubah | 1 jam - 24 jam |
| Hasil perhitungan mahal | Mengurangi beban CPU | 5 - 30 menit |
| Response API eksternal | Mengurangi latency | Sesuai SLA |
| Session data | Akses frequent | Sesuai session timeout |

### Data yang TIDAK Boleh di-Cache

- Data sensitif (PII, credentials)
- Data yang sering berubah
- Data per-user yang unik
- Transaksi real-time

### Cache Invalidation Strategies

```csharp
public class CacheInvalidationService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<CacheInvalidationService> _logger;

    // 1. Time-based expiration (sudah built-in)

    // 2. Event-based invalidation
    public async Task OnProdukUpdated(int produkId)
    {
        await _cache.RemoveAsync($"produk:{produkId}");
        await _cache.RemoveAsync("produk:semua");
        _logger.LogInformation("Cache produk {ProdukId} di-invalidate", produkId);
    }

    // 3. Version-based (cache key includes version)
    public string GetVersionedCacheKey(string baseKey, int version)
    {
        return $"{baseKey}:v{version}";
    }

    // 4. Pattern-based invalidation (memerlukan Redis)
    public async Task InvalidateByPatternAsync(string pattern)
    {
        // Implementasi dengan Redis SCAN + DEL
    }
}
```

## Database Scaling

### Read Replicas

```csharp
// Konfigurasi multiple connection strings
public class DatabaseSettings
{
    public string WriteConnection { get; set; } = string.Empty;
    public string ReadConnection { get; set; } = string.Empty;
}

// DbContext dengan read/write separation
public class ApplicationDbContext : DbContext
{
    private readonly DatabaseSettings _settings;
    private readonly bool _useReadReplica;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        DatabaseSettings settings,
        bool useReadReplica = false)
        : base(options)
    {
        _settings = settings;
        _useReadReplica = useReadReplica;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        var connectionString = _useReadReplica
            ? _settings.ReadConnection
            : _settings.WriteConnection;

        optionsBuilder.UseSqlServer(connectionString);
    }
}

// Factory untuk read replica
public interface IDbContextFactory
{
    ApplicationDbContext CreateReadContext();
    ApplicationDbContext CreateWriteContext();
}
```

### Connection Pooling

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=...;Min Pool Size=5;Max Pool Size=100;Connection Timeout=30;"
  }
}

// Best practices
// - Min Pool Size: 5-10 untuk kebanyakan aplikasi
// - Max Pool Size: 100 default, sesuaikan dengan beban
// - Selalu Dispose DbContext setelah selesai
// - Gunakan async methods untuk menghindari blocking pool
```

## Referensi

- [EF Core Performance](https://learn.microsoft.com/en-us/ef/core/performance/)
- [SQL Server Index Design Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- [Caching in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/)
- [Redis Caching Best Practices](https://redis.io/docs/management/optimization/benchmarks/)

---

*Terakhir diperbarui: Desember 2025*
