# Arsitektur dan Domain-Driven Design

Dokumen ini menjelaskan prinsip arsitektur dan Domain-Driven Design (DDD) untuk aplikasi .NET di Unictive.

## Arsitektur Dasar

### Clean Architecture

```
┌─────────────────────────────────────────────────┐
│                   Presentation                   │
│         (Controllers, Blazor, Razor)            │
└───────────────────────┬─────────────────────────┘
                        │ depends on
┌───────────────────────▼─────────────────────────┐
│                   Application                    │
│           (Use Cases, Services, DTOs)           │
└───────────────────────┬─────────────────────────┘
                        │ depends on
┌───────────────────────▼─────────────────────────┐
│                     Domain                       │
│    (Entities, Value Objects, Domain Events)     │
└───────────────────────┬─────────────────────────┘
                        │ implements
┌───────────────────────▼─────────────────────────┐
│                 Infrastructure                   │
│      (Repositories, External Services, DB)      │
└─────────────────────────────────────────────────┘
```

**Prinsip Dependency Rule:**
- Dependencies hanya boleh mengarah ke dalam (ke Domain)
- Domain tidak bergantung pada layer lain
- Infrastructure mengimplementasikan interface dari Domain

### Struktur Folder

```
src/
├── NamaProyek.API/
│   ├── Controllers/
│   ├── Filters/
│   ├── Middleware/
│   └── Program.cs
│
├── NamaProyek.Application/
│   ├── Common/
│   │   ├── Behaviors/
│   │   ├── Interfaces/
│   │   └── Mappings/
│   ├── Features/
│   │   └── Pelanggan/
│   │       ├── Commands/
│   │       │   ├── BuatPelanggan/
│   │       │   │   ├── BuatPelangganCommand.cs
│   │       │   │   ├── BuatPelangganHandler.cs
│   │       │   │   └── BuatPelangganValidator.cs
│   │       │   └── UpdatePelanggan/
│   │       └── Queries/
│   │           ├── DapatkanPelanggan/
│   │           └── DapatkanSemuaPelanggan/
│   └── DependencyInjection.cs
│
├── NamaProyek.Domain/
│   ├── Common/
│   │   ├── BaseEntity.cs
│   │   ├── ValueObject.cs
│   │   └── DomainEvent.cs
│   ├── Entities/
│   │   ├── Pelanggan.cs
│   │   └── Pesanan.cs
│   ├── Enums/
│   ├── Events/
│   ├── Exceptions/
│   └── Interfaces/
│       └── IPelangganRepository.cs
│
└── NamaProyek.Infrastructure/
    ├── Data/
    │   ├── ApplicationDbContext.cs
    │   └── Configurations/
    ├── Repositories/
    │   └── PelangganRepository.cs
    ├── Services/
    │   └── EmailService.cs
    └── DependencyInjection.cs
```

## Domain-Driven Design (DDD)

### Konsep Dasar DDD

| Konsep | Deskripsi | Contoh |
|--------|-----------|--------|
| **Entity** | Object dengan identitas unik | Pelanggan, Pesanan |
| **Value Object** | Object tanpa identitas, immutable | Email, Alamat, Uang |
| **Aggregate** | Cluster entities yang dikelola bersama | Pesanan + ItemPesanan |
| **Aggregate Root** | Entry point ke aggregate | Pesanan |
| **Domain Event** | Sesuatu yang terjadi di domain | PesananDibuat, PembayaranDiterima |
| **Repository** | Abstraksi untuk akses data | IPelangganRepository |
| **Domain Service** | Logic yang tidak cocok di Entity | TransferUangService |

### Entity

```csharp
// Domain/Common/BaseEntity.cs
public abstract class BaseEntity
{
    public int Id { get; protected set; }

    private readonly List<DomainEvent> _domainEvents = new();
    public IReadOnlyCollection<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(DomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Domain/Entities/Pelanggan.cs
public class Pelanggan : BaseEntity
{
    public string Nama { get; private set; } = string.Empty;
    public Email Email { get; private set; } = null!;
    public string? NomorTelepon { get; private set; }
    public Alamat? Alamat { get; private set; }
    public DateTime TanggalRegistrasi { get; private set; }
    public bool IsAktif { get; private set; }

    private readonly List<Pesanan> _pesanan = new();
    public IReadOnlyCollection<Pesanan> Pesanan => _pesanan.AsReadOnly();

    // Private constructor untuk EF Core
    private Pelanggan() { }

    // Factory method untuk membuat pelanggan baru
    public static Pelanggan Buat(string nama, string email, string? nomorTelepon = null)
    {
        var pelanggan = new Pelanggan
        {
            Nama = nama,
            Email = Email.Buat(email),
            NomorTelepon = nomorTelepon,
            TanggalRegistrasi = DateTime.UtcNow,
            IsAktif = true
        };

        pelanggan.AddDomainEvent(new PelangganDibuatEvent(pelanggan));

        return pelanggan;
    }

    // Behavior methods
    public void UpdateProfil(string nama, string? nomorTelepon)
    {
        if (string.IsNullOrWhiteSpace(nama))
            throw new DomainException("Nama tidak boleh kosong");

        Nama = nama;
        NomorTelepon = nomorTelepon;

        AddDomainEvent(new PelangganDiupdateEvent(this));
    }

    public void UpdateAlamat(Alamat alamat)
    {
        Alamat = alamat ?? throw new ArgumentNullException(nameof(alamat));
    }

    public void Nonaktifkan()
    {
        if (!IsAktif)
            throw new DomainException("Pelanggan sudah tidak aktif");

        IsAktif = false;
        AddDomainEvent(new PelangganDinonaktifkanEvent(this));
    }

    public void Aktifkan()
    {
        if (IsAktif)
            throw new DomainException("Pelanggan sudah aktif");

        IsAktif = true;
    }
}
```

### Value Object

```csharp
// Domain/Common/ValueObject.cs
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType())
            return false;

        var other = (ValueObject)obj;
        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override int GetHashCode()
    {
        return GetEqualityComponents()
            .Select(x => x?.GetHashCode() ?? 0)
            .Aggregate((x, y) => x ^ y);
    }

    public static bool operator ==(ValueObject? left, ValueObject? right)
    {
        return Equals(left, right);
    }

    public static bool operator !=(ValueObject? left, ValueObject? right)
    {
        return !Equals(left, right);
    }
}

// Domain/ValueObjects/Email.cs
public class Email : ValueObject
{
    public string Value { get; }

    private Email(string value)
    {
        Value = value;
    }

    public static Email Buat(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            throw new DomainException("Email tidak boleh kosong");

        email = email.Trim().ToLowerInvariant();

        if (!IsValidEmail(email))
            throw new DomainException("Format email tidak valid");

        return new Email(email);
    }

    private static bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }

    public override string ToString() => Value;

    // Implicit conversion untuk kemudahan
    public static implicit operator string(Email email) => email.Value;
}

// Domain/ValueObjects/Alamat.cs
public class Alamat : ValueObject
{
    public string Jalan { get; }
    public string Kota { get; }
    public string Provinsi { get; }
    public string KodePos { get; }

    private Alamat(string jalan, string kota, string provinsi, string kodePos)
    {
        Jalan = jalan;
        Kota = kota;
        Provinsi = provinsi;
        KodePos = kodePos;
    }

    public static Alamat Buat(string jalan, string kota, string provinsi, string kodePos)
    {
        if (string.IsNullOrWhiteSpace(jalan))
            throw new DomainException("Jalan tidak boleh kosong");

        if (string.IsNullOrWhiteSpace(kota))
            throw new DomainException("Kota tidak boleh kosong");

        return new Alamat(jalan, kota, provinsi, kodePos);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Jalan;
        yield return Kota;
        yield return Provinsi;
        yield return KodePos;
    }

    public override string ToString() =>
        $"{Jalan}, {Kota}, {Provinsi} {KodePos}";
}

// Domain/ValueObjects/Money.cs
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Money Rupiah(decimal amount) => new(amount, "IDR");
    public static Money Dollar(decimal amount) => new(amount, "USD");

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Tidak dapat menambah mata uang berbeda");

        return new Money(Amount + other.Amount, Currency);
    }

    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Tidak dapat mengurangi mata uang berbeda");

        return new Money(Amount - other.Amount, Currency);
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public override string ToString() =>
        Currency == "IDR"
            ? $"Rp {Amount:N0}"
            : $"{Currency} {Amount:N2}";
}
```

### Aggregate Root

```csharp
// Domain/Entities/Pesanan.cs
public class Pesanan : BaseEntity
{
    public string NomorPesanan { get; private set; } = string.Empty;
    public int PelangganId { get; private set; }
    public Pelanggan Pelanggan { get; private set; } = null!;
    public DateTime TanggalPesanan { get; private set; }
    public StatusPesanan Status { get; private set; }
    public Money Total { get; private set; } = Money.Rupiah(0);
    public Alamat? AlamatPengiriman { get; private set; }

    private readonly List<ItemPesanan> _items = new();
    public IReadOnlyCollection<ItemPesanan> Items => _items.AsReadOnly();

    private Pesanan() { }

    public static Pesanan Buat(Pelanggan pelanggan, Alamat alamatPengiriman)
    {
        var pesanan = new Pesanan
        {
            NomorPesanan = GenerateNomorPesanan(),
            PelangganId = pelanggan.Id,
            Pelanggan = pelanggan,
            TanggalPesanan = DateTime.UtcNow,
            Status = StatusPesanan.Baru,
            AlamatPengiriman = alamatPengiriman
        };

        pesanan.AddDomainEvent(new PesananDibuatEvent(pesanan));

        return pesanan;
    }

    // Behavior: Tambah Item (hanya melalui Aggregate Root)
    public void TambahItem(Produk produk, int jumlah)
    {
        if (Status != StatusPesanan.Baru)
            throw new DomainException("Tidak dapat menambah item ke pesanan yang sudah diproses");

        if (jumlah <= 0)
            throw new DomainException("Jumlah harus lebih dari 0");

        var existingItem = _items.FirstOrDefault(i => i.ProdukId == produk.Id);

        if (existingItem != null)
        {
            existingItem.TambahJumlah(jumlah);
        }
        else
        {
            var item = ItemPesanan.Buat(this, produk, jumlah);
            _items.Add(item);
        }

        HitungTotal();
    }

    // Behavior: Hapus Item
    public void HapusItem(int produkId)
    {
        if (Status != StatusPesanan.Baru)
            throw new DomainException("Tidak dapat menghapus item dari pesanan yang sudah diproses");

        var item = _items.FirstOrDefault(i => i.ProdukId == produkId);
        if (item == null)
            throw new DomainException("Item tidak ditemukan");

        _items.Remove(item);
        HitungTotal();
    }

    // Behavior: Proses Pesanan
    public void Proses()
    {
        if (Status != StatusPesanan.Baru)
            throw new DomainException("Hanya pesanan baru yang dapat diproses");

        if (!_items.Any())
            throw new DomainException("Pesanan harus memiliki minimal 1 item");

        Status = StatusPesanan.Diproses;
        AddDomainEvent(new PesananDiprosesEvent(this));
    }

    // Behavior: Kirim Pesanan
    public void Kirim(string nomorResi)
    {
        if (Status != StatusPesanan.Diproses)
            throw new DomainException("Hanya pesanan yang diproses yang dapat dikirim");

        Status = StatusPesanan.Dikirim;
        AddDomainEvent(new PesananDikirimEvent(this, nomorResi));
    }

    // Behavior: Selesaikan Pesanan
    public void Selesai()
    {
        if (Status != StatusPesanan.Dikirim)
            throw new DomainException("Hanya pesanan yang dikirim yang dapat diselesaikan");

        Status = StatusPesanan.Selesai;
        AddDomainEvent(new PesananSelesaiEvent(this));
    }

    // Behavior: Batalkan Pesanan
    public void Batalkan(string alasan)
    {
        if (Status == StatusPesanan.Selesai || Status == StatusPesanan.Dibatalkan)
            throw new DomainException("Pesanan tidak dapat dibatalkan");

        Status = StatusPesanan.Dibatalkan;
        AddDomainEvent(new PesananDibatalkanEvent(this, alasan));
    }

    private void HitungTotal()
    {
        Total = _items.Aggregate(
            Money.Rupiah(0),
            (total, item) => total.Add(item.Subtotal));
    }

    private static string GenerateNomorPesanan()
    {
        return $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString()[..8].ToUpper()}";
    }
}

// Domain/Entities/ItemPesanan.cs (bukan Aggregate Root, bagian dari Pesanan)
public class ItemPesanan : BaseEntity
{
    public int PesananId { get; private set; }
    public int ProdukId { get; private set; }
    public Produk Produk { get; private set; } = null!;
    public int Jumlah { get; private set; }
    public Money HargaSatuan { get; private set; } = null!;
    public Money Subtotal => Money.Rupiah(HargaSatuan.Amount * Jumlah);

    private ItemPesanan() { }

    internal static ItemPesanan Buat(Pesanan pesanan, Produk produk, int jumlah)
    {
        return new ItemPesanan
        {
            PesananId = pesanan.Id,
            ProdukId = produk.Id,
            Produk = produk,
            Jumlah = jumlah,
            HargaSatuan = produk.Harga
        };
    }

    internal void TambahJumlah(int jumlah)
    {
        Jumlah += jumlah;
    }
}
```

### Domain Events

```csharp
// Domain/Common/DomainEvent.cs
public abstract class DomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
    public Guid EventId { get; } = Guid.NewGuid();
}

// Domain/Events/PesananEvents.cs
public class PesananDibuatEvent : DomainEvent
{
    public Pesanan Pesanan { get; }

    public PesananDibuatEvent(Pesanan pesanan)
    {
        Pesanan = pesanan;
    }
}

public class PesananDiprosesEvent : DomainEvent
{
    public Pesanan Pesanan { get; }

    public PesananDiprosesEvent(Pesanan pesanan)
    {
        Pesanan = pesanan;
    }
}

// Application/Common/Interfaces/IDomainEventDispatcher.cs
public interface IDomainEventDispatcher
{
    Task DispatchEventsAsync(IEnumerable<DomainEvent> events, CancellationToken ct = default);
}

// Infrastructure/Services/DomainEventDispatcher.cs
public class DomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IMediator _mediator;

    public DomainEventDispatcher(IMediator mediator)
    {
        _mediator = mediator;
    }

    public async Task DispatchEventsAsync(IEnumerable<DomainEvent> events, CancellationToken ct = default)
    {
        foreach (var domainEvent in events)
        {
            await _mediator.Publish(domainEvent, ct);
        }
    }
}

// Application/Features/Pesanan/EventHandlers/PesananDibuatEventHandler.cs
public class PesananDibuatEventHandler : INotificationHandler<PesananDibuatEvent>
{
    private readonly IEmailService _emailService;
    private readonly ILogger<PesananDibuatEventHandler> _logger;

    public async Task Handle(PesananDibuatEvent notification, CancellationToken ct)
    {
        _logger.LogInformation("Pesanan {NomorPesanan} dibuat", notification.Pesanan.NomorPesanan);

        // Kirim email konfirmasi
        await _emailService.KirimKonfirmasiPesananAsync(
            notification.Pesanan.Pelanggan.Email,
            notification.Pesanan.NomorPesanan,
            ct);
    }
}
```

### Repository Interface (di Domain Layer)

```csharp
// Domain/Interfaces/IPesananRepository.cs
public interface IPesananRepository
{
    Task<Pesanan?> DapatkanByIdAsync(int id, CancellationToken ct = default);
    Task<Pesanan?> DapatkanByNomorAsync(string nomorPesanan, CancellationToken ct = default);
    Task<IEnumerable<Pesanan>> DapatkanByPelangganAsync(int pelangganId, CancellationToken ct = default);
    Task TambahAsync(Pesanan pesanan, CancellationToken ct = default);
    Task UpdateAsync(Pesanan pesanan, CancellationToken ct = default);
}
```

## CQRS dengan MediatR

### Command

```csharp
// Application/Features/Pesanan/Commands/BuatPesanan/BuatPesananCommand.cs
public record BuatPesananCommand(
    int PelangganId,
    List<ItemPesananDto> Items,
    AlamatDto AlamatPengiriman
) : IRequest<PesananDto>;

public record ItemPesananDto(int ProdukId, int Jumlah);
public record AlamatDto(string Jalan, string Kota, string Provinsi, string KodePos);

// Application/Features/Pesanan/Commands/BuatPesanan/BuatPesananHandler.cs
public class BuatPesananHandler : IRequestHandler<BuatPesananCommand, PesananDto>
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMapper _mapper;
    private readonly IDomainEventDispatcher _eventDispatcher;

    public async Task<PesananDto> Handle(BuatPesananCommand request, CancellationToken ct)
    {
        // Ambil pelanggan
        var pelanggan = await _unitOfWork.Pelanggan.DapatkanByIdAsync(request.PelangganId, ct)
            ?? throw NotFoundException.ForEntity<Pelanggan>(request.PelangganId);

        // Buat alamat
        var alamat = Alamat.Buat(
            request.AlamatPengiriman.Jalan,
            request.AlamatPengiriman.Kota,
            request.AlamatPengiriman.Provinsi,
            request.AlamatPengiriman.KodePos);

        // Buat pesanan (menggunakan factory method)
        var pesanan = Pesanan.Buat(pelanggan, alamat);

        // Tambah items
        foreach (var item in request.Items)
        {
            var produk = await _unitOfWork.Produk.DapatkanByIdAsync(item.ProdukId, ct)
                ?? throw NotFoundException.ForEntity<Produk>(item.ProdukId);

            pesanan.TambahItem(produk, item.Jumlah);
        }

        // Simpan
        await _unitOfWork.Pesanan.TambahAsync(pesanan, ct);
        await _unitOfWork.SimpanPerubahanAsync(ct);

        // Dispatch domain events
        await _eventDispatcher.DispatchEventsAsync(pesanan.DomainEvents, ct);
        pesanan.ClearDomainEvents();

        return _mapper.Map<PesananDto>(pesanan);
    }
}
```

### Query

```csharp
// Application/Features/Pesanan/Queries/DapatkanPesanan/DapatkanPesananQuery.cs
public record DapatkanPesananQuery(string NomorPesanan) : IRequest<PesananDto?>;

// Application/Features/Pesanan/Queries/DapatkanPesanan/DapatkanPesananHandler.cs
public class DapatkanPesananHandler : IRequestHandler<DapatkanPesananQuery, PesananDto?>
{
    private readonly IApplicationDbContext _context;
    private readonly IMapper _mapper;

    public async Task<PesananDto?> Handle(DapatkanPesananQuery request, CancellationToken ct)
    {
        // Query langsung, tanpa melalui repository (untuk efisiensi read)
        var pesanan = await _context.Pesanan
            .AsNoTracking()
            .Include(p => p.Items)
                .ThenInclude(i => i.Produk)
            .Include(p => p.Pelanggan)
            .FirstOrDefaultAsync(p => p.NomorPesanan == request.NomorPesanan, ct);

        return pesanan is null ? null : _mapper.Map<PesananDto>(pesanan);
    }
}
```

## Architecture Decision Records (ADR)

### Template ADR

```markdown
# ADR-001: Menggunakan CQRS dengan MediatR

## Status
Accepted

## Konteks
Aplikasi kita perlu menangani operasi baca dan tulis dengan karakteristik yang berbeda.
Operasi tulis memerlukan validasi bisnis kompleks, sedangkan operasi baca perlu dioptimasi untuk performa.

## Keputusan
Kita akan menggunakan pola CQRS (Command Query Responsibility Segregation) dengan library MediatR untuk memisahkan
operasi command (write) dan query (read).

## Konsekuensi

### Positif
- Pemisahan yang jelas antara read dan write operations
- Query dapat dioptimasi tanpa mempengaruhi commands
- Lebih mudah di-test karena handlers terpisah
- Mendukung eventual consistency jika diperlukan

### Negatif
- Kompleksitas bertambah dibanding CRUD sederhana
- Learning curve untuk developer baru
- Overhead tambahan untuk use case sederhana

## Alternatif yang Dipertimbangkan
1. CRUD tradisional dengan service layer
2. Event Sourcing penuh

## Catatan
Keputusan ini berlaku untuk domain kompleks (Pesanan, Pembayaran).
Untuk domain sederhana, boleh menggunakan pendekatan CRUD tradisional.
```

### Lokasi ADR

```
docs/
└── adr/
    ├── ADR-001-cqrs-dengan-mediatr.md
    ├── ADR-002-postgresql-sebagai-database-utama.md
    ├── ADR-003-redis-untuk-caching.md
    └── ADR-004-clean-architecture.md
```

## Referensi

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [MediatR Documentation](https://github.com/jbogard/MediatR)
- [CQRS Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)

---

*Terakhir diperbarui: Desember 2025*
