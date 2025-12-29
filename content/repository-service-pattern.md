---
title: Pola Repository dan Service
description: Panduan implementasi pola Repository dan Service dalam aplikasi .NET untuk memisahkan logika bisnis dari akses data dan meningkatkan testabilitas kode.
published: true
date: 2025-12-29T00:00:00.000Z
tags: repository, service, pattern
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

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

## Referensi

- [Repository Pattern in ASP.NET Core](https://code-maze.com/net-core-web-development-part4/)
- [Unit of Work Pattern](https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

*Terakhir diperbarui: Desember 2025*
