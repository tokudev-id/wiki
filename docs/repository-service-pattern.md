# Pola Repository dan Service

Dokumen ini menjelaskan implementasi pola Repository dan Service dalam aplikasi .NET. Pola-pola ini membantu memisahkan logika bisnis dari akses data dan meningkatkan testabilitas kode.

## Pengantar

### Apa itu Repository Pattern?

Repository Pattern adalah pola desain yang memisahkan logika akses data dari logika bisnis. Repository bertindak sebagai lapisan abstraksi antara domain aplikasi dan lapisan pemetaan data.

### Apa itu Service Pattern?

Service Pattern adalah pola yang mengenkapsulasi logika bisnis dalam kelas-kelas service yang terpisah. Service mengkoordinasikan operasi antara berbagai repository dan mengimplementasikan aturan bisnis.

### Arsitektur Berlapis

```
┌─────────────────────────────────────┐
│          Presentation Layer         │
│    (Controllers, Razor, Blazor)     │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│          Application Layer          │
│         (Services, DTOs)            │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│           Domain Layer              │
│     (Entities, Interfaces)          │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│        Infrastructure Layer         │
│   (Repositories, DbContext, API)    │
└─────────────────────────────────────┘
```

## Implementasi Repository Pattern

### 1. Generic Repository Interface

```csharp
// Domain/Interfaces/IRepository.cs
public interface IRepository<T> where T : class
{
    Task<T?> DapatkanByIdAsync(int id, CancellationToken ct = default);
    Task<IEnumerable<T>> DapatkanSemuaAsync(CancellationToken ct = default);
    Task<T> TambahAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task HapusAsync(T entity, CancellationToken ct = default);
    Task<bool> AdaAsync(int id, CancellationToken ct = default);
}
```

### 2. Generic Repository Implementation

```csharp
// Infrastructure/Repositories/Repository.cs
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<T?> DapatkanByIdAsync(int id, CancellationToken ct = default)
    {
        return await _dbSet.FindAsync(new object[] { id }, ct);
    }

    public virtual async Task<IEnumerable<T>> DapatkanSemuaAsync(CancellationToken ct = default)
    {
        return await _dbSet.AsNoTracking().ToListAsync(ct);
    }

    public virtual async Task<T> TambahAsync(T entity, CancellationToken ct = default)
    {
        await _dbSet.AddAsync(entity, ct);
        return entity;
    }

    public virtual Task UpdateAsync(T entity, CancellationToken ct = default)
    {
        _dbSet.Update(entity);
        return Task.CompletedTask;
    }

    public virtual Task HapusAsync(T entity, CancellationToken ct = default)
    {
        _dbSet.Remove(entity);
        return Task.CompletedTask;
    }

    public virtual async Task<bool> AdaAsync(int id, CancellationToken ct = default)
    {
        var entity = await DapatkanByIdAsync(id, ct);
        return entity != null;
    }
}
```

### 3. Specific Repository Interface

```csharp
// Domain/Interfaces/IPelangganRepository.cs
public interface IPelangganRepository : IRepository<Pelanggan>
{
    Task<Pelanggan?> DapatkanByEmailAsync(string email, CancellationToken ct = default);
    Task<IEnumerable<Pelanggan>> DapatkanPelangganAktifAsync(CancellationToken ct = default);
    Task<IEnumerable<Pelanggan>> CariByNamaAsync(string nama, CancellationToken ct = default);
    Task<PagedResult<Pelanggan>> DapatkanDenganPaginasiAsync(
        int pageNumber,
        int pageSize,
        CancellationToken ct = default);
}
```

### 4. Specific Repository Implementation

```csharp
// Infrastructure/Repositories/PelangganRepository.cs
public class PelangganRepository : Repository<Pelanggan>, IPelangganRepository
{
    public PelangganRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<Pelanggan?> DapatkanByEmailAsync(string email, CancellationToken ct = default)
    {
        return await _dbSet
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Email == email, ct);
    }

    public async Task<IEnumerable<Pelanggan>> DapatkanPelangganAktifAsync(CancellationToken ct = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.IsAktif)
            .OrderBy(p => p.Nama)
            .ToListAsync(ct);
    }

    public async Task<IEnumerable<Pelanggan>> CariByNamaAsync(string nama, CancellationToken ct = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.Nama.Contains(nama))
            .OrderBy(p => p.Nama)
            .ToListAsync(ct);
    }

    public async Task<PagedResult<Pelanggan>> DapatkanDenganPaginasiAsync(
        int pageNumber,
        int pageSize,
        CancellationToken ct = default)
    {
        var totalCount = await _dbSet.CountAsync(ct);

        var items = await _dbSet
            .AsNoTracking()
            .OrderBy(p => p.Id)
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return new PagedResult<Pelanggan>
        {
            Items = items,
            TotalCount = totalCount,
            PageNumber = pageNumber,
            PageSize = pageSize
        };
    }
}
```

## Unit of Work Pattern

Unit of Work mengkoordinasikan beberapa repository untuk memastikan semua perubahan disimpan dalam satu transaksi.

### 1. Interface Unit of Work

```csharp
// Domain/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork : IDisposable
{
    IPelangganRepository Pelanggan { get; }
    IPesananRepository Pesanan { get; }
    IProdukRepository Produk { get; }

    Task<int> SimpanPerubahanAsync(CancellationToken ct = default);
    Task BeginTransactionAsync(CancellationToken ct = default);
    Task CommitTransactionAsync(CancellationToken ct = default);
    Task RollbackTransactionAsync(CancellationToken ct = default);
}
```

### 2. Implementasi Unit of Work

```csharp
// Infrastructure/Data/UnitOfWork.cs
public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _transaction;

    private IPelangganRepository? _pelangganRepository;
    private IPesananRepository? _pesananRepository;
    private IProdukRepository? _produkRepository;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }

    public IPelangganRepository Pelanggan =>
        _pelangganRepository ??= new PelangganRepository(_context);

    public IPesananRepository Pesanan =>
        _pesananRepository ??= new PesananRepository(_context);

    public IProdukRepository Produk =>
        _produkRepository ??= new ProdukRepository(_context);

    public async Task<int> SimpanPerubahanAsync(CancellationToken ct = default)
    {
        return await _context.SaveChangesAsync(ct);
    }

    public async Task BeginTransactionAsync(CancellationToken ct = default)
    {
        _transaction = await _context.Database.BeginTransactionAsync(ct);
    }

    public async Task CommitTransactionAsync(CancellationToken ct = default)
    {
        try
        {
            await _context.SaveChangesAsync(ct);
            await _transaction?.CommitAsync(ct)!;
        }
        catch
        {
            await RollbackTransactionAsync(ct);
            throw;
        }
        finally
        {
            _transaction?.Dispose();
            _transaction = null;
        }
    }

    public async Task RollbackTransactionAsync(CancellationToken ct = default)
    {
        if (_transaction != null)
        {
            await _transaction.RollbackAsync(ct);
            _transaction.Dispose();
            _transaction = null;
        }
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
        GC.SuppressFinalize(this);
    }
}
```

## Service Pattern

### 1. Service Interface

```csharp
// Application/Interfaces/IPelangganService.cs
public interface IPelangganService
{
    Task<PelangganDto?> DapatkanByIdAsync(int id, CancellationToken ct = default);
    Task<IEnumerable<PelangganDto>> DapatkanSemuaAsync(CancellationToken ct = default);
    Task<PagedResult<PelangganDto>> DapatkanDenganFilterAsync(
        PelangganFilter filter,
        CancellationToken ct = default);
    Task<PelangganDto> BuatAsync(BuatPelangganRequest request, CancellationToken ct = default);
    Task<PelangganDto> UpdateAsync(int id, UpdatePelangganRequest request, CancellationToken ct = default);
    Task HapusAsync(int id, CancellationToken ct = default);
    Task<bool> EmailTersediaAsync(string email, int? excludeId = null, CancellationToken ct = default);
}
```

### 2. Service Implementation

```csharp
// Application/Services/PelangganService.cs
public class PelangganService : IPelangganService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMapper _mapper;
    private readonly IValidator<BuatPelangganRequest> _buatValidator;
    private readonly ILogger<PelangganService> _logger;

    public PelangganService(
        IUnitOfWork unitOfWork,
        IMapper mapper,
        IValidator<BuatPelangganRequest> buatValidator,
        ILogger<PelangganService> logger)
    {
        _unitOfWork = unitOfWork;
        _mapper = mapper;
        _buatValidator = buatValidator;
        _logger = logger;
    }

    public async Task<PelangganDto?> DapatkanByIdAsync(int id, CancellationToken ct = default)
    {
        var pelanggan = await _unitOfWork.Pelanggan.DapatkanByIdAsync(id, ct);
        return pelanggan is null ? null : _mapper.Map<PelangganDto>(pelanggan);
    }

    public async Task<IEnumerable<PelangganDto>> DapatkanSemuaAsync(CancellationToken ct = default)
    {
        var pelanggan = await _unitOfWork.Pelanggan.DapatkanSemuaAsync(ct);
        return _mapper.Map<IEnumerable<PelangganDto>>(pelanggan);
    }

    public async Task<PelangganDto> BuatAsync(
        BuatPelangganRequest request,
        CancellationToken ct = default)
    {
        // Validasi
        var validationResult = await _buatValidator.ValidateAsync(request, ct);
        if (!validationResult.IsValid)
        {
            throw new ValidationException(validationResult.Errors);
        }

        // Cek duplikat email
        if (!await EmailTersediaAsync(request.Email, null, ct))
        {
            throw new ConflictException($"Email '{request.Email}' sudah terdaftar");
        }

        // Buat entity
        var pelanggan = new Pelanggan
        {
            Nama = request.Nama,
            Email = request.Email.ToLowerInvariant(),
            NomorTelepon = request.NomorTelepon,
            TanggalRegistrasi = DateTime.UtcNow,
            IsAktif = true
        };

        await _unitOfWork.Pelanggan.TambahAsync(pelanggan, ct);
        await _unitOfWork.SimpanPerubahanAsync(ct);

        _logger.LogInformation("Pelanggan baru dibuat dengan ID {PelangganId}", pelanggan.Id);

        return _mapper.Map<PelangganDto>(pelanggan);
    }

    public async Task<PelangganDto> UpdateAsync(
        int id,
        UpdatePelangganRequest request,
        CancellationToken ct = default)
    {
        var pelanggan = await _unitOfWork.Pelanggan.DapatkanByIdAsync(id, ct);
        if (pelanggan is null)
        {
            throw NotFoundException.ForEntity<Pelanggan>(id);
        }

        // Cek duplikat email jika berubah
        if (pelanggan.Email != request.Email)
        {
            if (!await EmailTersediaAsync(request.Email, id, ct))
            {
                throw new ConflictException($"Email '{request.Email}' sudah digunakan");
            }
        }

        // Update properties
        pelanggan.Nama = request.Nama;
        pelanggan.Email = request.Email.ToLowerInvariant();
        pelanggan.NomorTelepon = request.NomorTelepon;
        pelanggan.TanggalDiperbarui = DateTime.UtcNow;

        await _unitOfWork.Pelanggan.UpdateAsync(pelanggan, ct);
        await _unitOfWork.SimpanPerubahanAsync(ct);

        _logger.LogInformation("Pelanggan {PelangganId} berhasil diperbarui", id);

        return _mapper.Map<PelangganDto>(pelanggan);
    }

    public async Task HapusAsync(int id, CancellationToken ct = default)
    {
        var pelanggan = await _unitOfWork.Pelanggan.DapatkanByIdAsync(id, ct);
        if (pelanggan is null)
        {
            throw NotFoundException.ForEntity<Pelanggan>(id);
        }

        // Cek apakah ada pesanan terkait
        var adaPesanan = await _unitOfWork.Pesanan.AdaPesananUntukPelangganAsync(id, ct);
        if (adaPesanan)
        {
            throw new ConflictException("Pelanggan tidak dapat dihapus karena memiliki pesanan");
        }

        await _unitOfWork.Pelanggan.HapusAsync(pelanggan, ct);
        await _unitOfWork.SimpanPerubahanAsync(ct);

        _logger.LogInformation("Pelanggan {PelangganId} berhasil dihapus", id);
    }

    public async Task<bool> EmailTersediaAsync(
        string email,
        int? excludeId = null,
        CancellationToken ct = default)
    {
        var pelanggan = await _unitOfWork.Pelanggan.DapatkanByEmailAsync(email, ct);

        if (pelanggan is null)
        {
            return true;
        }

        return excludeId.HasValue && pelanggan.Id == excludeId.Value;
    }
}
```

### 3. Operasi dengan Transaksi

```csharp
public class PesananService : IPesananService
{
    private readonly IUnitOfWork _unitOfWork;

    public async Task<PesananDto> BuatPesananAsync(
        BuatPesananRequest request,
        CancellationToken ct = default)
    {
        await _unitOfWork.BeginTransactionAsync(ct);

        try
        {
            // 1. Validasi pelanggan
            var pelanggan = await _unitOfWork.Pelanggan.DapatkanByIdAsync(request.PelangganId, ct);
            if (pelanggan is null)
            {
                throw NotFoundException.ForEntity<Pelanggan>(request.PelangganId);
            }

            // 2. Buat pesanan
            var pesanan = new Pesanan
            {
                PelangganId = request.PelangganId,
                TanggalPesanan = DateTime.UtcNow,
                Status = StatusPesanan.Baru
            };

            await _unitOfWork.Pesanan.TambahAsync(pesanan, ct);

            // 3. Proses item pesanan
            foreach (var item in request.Items)
            {
                var produk = await _unitOfWork.Produk.DapatkanByIdAsync(item.ProdukId, ct);
                if (produk is null)
                {
                    throw NotFoundException.ForEntity<Produk>(item.ProdukId);
                }

                // Cek stok
                if (produk.Stok < item.Jumlah)
                {
                    throw new ValidationException($"Stok produk '{produk.Nama}' tidak mencukupi");
                }

                // Kurangi stok
                produk.Stok -= item.Jumlah;
                await _unitOfWork.Produk.UpdateAsync(produk, ct);

                // Tambah item pesanan
                pesanan.Items.Add(new ItemPesanan
                {
                    ProdukId = item.ProdukId,
                    Jumlah = item.Jumlah,
                    HargaSatuan = produk.Harga
                });
            }

            // 4. Hitung total
            pesanan.Total = pesanan.Items.Sum(i => i.Jumlah * i.HargaSatuan);

            // 5. Commit transaksi
            await _unitOfWork.CommitTransactionAsync(ct);

            return _mapper.Map<PesananDto>(pesanan);
        }
        catch
        {
            await _unitOfWork.RollbackTransactionAsync(ct);
            throw;
        }
    }
}
```

## DTOs (Data Transfer Objects)

### 1. Definisi DTO

```csharp
// Application/DTOs/PelangganDto.cs
public record PelangganDto(
    int Id,
    string Nama,
    string Email,
    string? NomorTelepon,
    DateTime TanggalRegistrasi,
    bool IsAktif
);

// Request DTOs
public record BuatPelangganRequest(
    string Nama,
    string Email,
    string? NomorTelepon
);

public record UpdatePelangganRequest(
    string Nama,
    string Email,
    string? NomorTelepon
);

// Response dengan paginasi
public class PagedResult<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => PageNumber > 1;
    public bool HasNext => PageNumber < TotalPages;
}
```

### 2. AutoMapper Configuration

```csharp
// Application/Mappings/MappingProfile.cs
public class MappingProfile : Profile
{
    public MappingProfile()
    {
        CreateMap<Pelanggan, PelangganDto>();
        CreateMap<BuatPelangganRequest, Pelanggan>()
            .ForMember(dest => dest.TanggalRegistrasi, opt => opt.MapFrom(_ => DateTime.UtcNow))
            .ForMember(dest => dest.IsAktif, opt => opt.MapFrom(_ => true));

        CreateMap<Pesanan, PesananDto>()
            .ForMember(dest => dest.NamaPelanggan, opt => opt.MapFrom(src => src.Pelanggan.Nama));
    }
}
```

## Dependency Injection

### Registrasi di Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Unit of Work
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Repositories
builder.Services.AddScoped<IPelangganRepository, PelangganRepository>();
builder.Services.AddScoped<IPesananRepository, PesananRepository>();
builder.Services.AddScoped<IProdukRepository, ProdukRepository>();

// Services
builder.Services.AddScoped<IPelangganService, PelangganService>();
builder.Services.AddScoped<IPesananService, PesananService>();
builder.Services.AddScoped<IProdukService, ProdukService>();

// AutoMapper
builder.Services.AddAutoMapper(typeof(MappingProfile));

// Validators
builder.Services.AddValidatorsFromAssemblyContaining<BuatPelangganRequestValidator>();
```

### Extension Method untuk Registrasi

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IPelangganRepository, PelangganRepository>();
        services.AddScoped<IPesananRepository, PesananRepository>();
        services.AddScoped<IProdukRepository, ProdukRepository>();

        return services;
    }
}

// Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<IPelangganService, PelangganService>();
        services.AddScoped<IPesananService, PesananService>();
        services.AddScoped<IProdukService, ProdukService>();

        services.AddAutoMapper(typeof(MappingProfile));
        services.AddValidatorsFromAssemblyContaining<BuatPelangganRequestValidator>();

        return services;
    }
}

// Program.cs
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddApplication();
```

## Testing

### 1. Unit Test untuk Service

```csharp
public class PelangganServiceTests
{
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly Mock<IMapper> _mapperMock;
    private readonly Mock<IValidator<BuatPelangganRequest>> _validatorMock;
    private readonly Mock<ILogger<PelangganService>> _loggerMock;
    private readonly PelangganService _sut;

    public PelangganServiceTests()
    {
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _mapperMock = new Mock<IMapper>();
        _validatorMock = new Mock<IValidator<BuatPelangganRequest>>();
        _loggerMock = new Mock<ILogger<PelangganService>>();

        _sut = new PelangganService(
            _unitOfWorkMock.Object,
            _mapperMock.Object,
            _validatorMock.Object,
            _loggerMock.Object);
    }

    [Fact]
    public async Task DapatkanByIdAsync_PelangganAda_ReturnDto()
    {
        // Arrange
        var pelanggan = new Pelanggan { Id = 1, Nama = "Test", Email = "test@test.com" };
        var dto = new PelangganDto(1, "Test", "test@test.com", null, DateTime.UtcNow, true);

        _unitOfWorkMock
            .Setup(x => x.Pelanggan.DapatkanByIdAsync(1, default))
            .ReturnsAsync(pelanggan);

        _mapperMock
            .Setup(x => x.Map<PelangganDto>(pelanggan))
            .Returns(dto);

        // Act
        var result = await _sut.DapatkanByIdAsync(1);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(1, result.Id);
        Assert.Equal("Test", result.Nama);
    }

    [Fact]
    public async Task DapatkanByIdAsync_PelangganTidakAda_ReturnNull()
    {
        // Arrange
        _unitOfWorkMock
            .Setup(x => x.Pelanggan.DapatkanByIdAsync(999, default))
            .ReturnsAsync((Pelanggan?)null);

        // Act
        var result = await _sut.DapatkanByIdAsync(999);

        // Assert
        Assert.Null(result);
    }

    [Fact]
    public async Task BuatAsync_EmailSudahAda_ThrowConflictException()
    {
        // Arrange
        var request = new BuatPelangganRequest("Test", "existing@test.com", null);
        var existingPelanggan = new Pelanggan { Id = 1, Email = "existing@test.com" };

        _validatorMock
            .Setup(x => x.ValidateAsync(request, default))
            .ReturnsAsync(new FluentValidation.Results.ValidationResult());

        _unitOfWorkMock
            .Setup(x => x.Pelanggan.DapatkanByEmailAsync("existing@test.com", default))
            .ReturnsAsync(existingPelanggan);

        // Act & Assert
        await Assert.ThrowsAsync<ConflictException>(() => _sut.BuatAsync(request));
    }
}
```

### 2. Integration Test

```csharp
public class PelangganRepositoryIntegrationTests : IDisposable
{
    private readonly ApplicationDbContext _context;
    private readonly PelangganRepository _repository;

    public PelangganRepositoryIntegrationTests()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        _context = new ApplicationDbContext(options);
        _repository = new PelangganRepository(_context);
    }

    [Fact]
    public async Task DapatkanByEmailAsync_EmailAda_ReturnPelanggan()
    {
        // Arrange
        var pelanggan = new Pelanggan
        {
            Nama = "Test",
            Email = "test@test.com",
            TanggalRegistrasi = DateTime.UtcNow
        };
        await _context.Pelanggan.AddAsync(pelanggan);
        await _context.SaveChangesAsync();

        // Act
        var result = await _repository.DapatkanByEmailAsync("test@test.com");

        // Assert
        Assert.NotNull(result);
        Assert.Equal("test@test.com", result.Email);
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

## Praktik Terbaik

### 1. Jangan Mengekspos Entity ke Lapisan Presentasi

```csharp
// BURUK - Mengekspos entity langsung
public async Task<Pelanggan> DapatkanByIdAsync(int id)
{
    return await _repository.DapatkanByIdAsync(id);
}

// BAIK - Menggunakan DTO
public async Task<PelangganDto?> DapatkanByIdAsync(int id)
{
    var pelanggan = await _repository.DapatkanByIdAsync(id);
    return pelanggan is null ? null : _mapper.Map<PelangganDto>(pelanggan);
}
```

### 2. Gunakan AsNoTracking untuk Query Read-Only

```csharp
public async Task<IEnumerable<Pelanggan>> DapatkanSemuaAsync()
{
    return await _dbSet
        .AsNoTracking()  // Lebih cepat untuk read-only
        .ToListAsync();
}
```

### 3. Hindari N+1 Query Problem

```csharp
// BURUK - N+1 queries
public async Task<IEnumerable<PesananDto>> DapatkanPesananAsync()
{
    var pesanan = await _dbSet.ToListAsync();
    foreach (var p in pesanan)
    {
        p.Pelanggan = await _context.Pelanggan.FindAsync(p.PelangganId);  // N queries
    }
    return _mapper.Map<IEnumerable<PesananDto>>(pesanan);
}

// BAIK - Eager loading
public async Task<IEnumerable<PesananDto>> DapatkanPesananAsync()
{
    var pesanan = await _dbSet
        .Include(p => p.Pelanggan)  // 1 query dengan JOIN
        .Include(p => p.Items)
            .ThenInclude(i => i.Produk)
        .AsNoTracking()
        .ToListAsync();

    return _mapper.Map<IEnumerable<PesananDto>>(pesanan);
}
```

### 4. Pertimbangkan Specification Pattern untuk Query Kompleks

```csharp
// Specification
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
}

public class PelangganAktifSpecification : ISpecification<Pelanggan>
{
    public Expression<Func<Pelanggan, bool>> Criteria =>
        p => p.IsAktif;

    public List<Expression<Func<Pelanggan, object>>> Includes => new()
    {
        p => p.Pesanan
    };
}

// Repository dengan Specification
public async Task<IEnumerable<T>> DapatkanDenganSpecAsync(ISpecification<T> spec)
{
    var query = _dbSet.Where(spec.Criteria);

    foreach (var include in spec.Includes)
    {
        query = query.Include(include);
    }

    return await query.AsNoTracking().ToListAsync();
}
```

## Referensi

- [Repository Pattern in ASP.NET Core](https://code-maze.com/net-core-web-development-part4/)
- [Unit of Work Pattern](https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

*Terakhir diperbarui: Desember 2025*
