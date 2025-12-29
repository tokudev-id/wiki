---
title: Konvensi Penamaan
description: Panduan konvensi penamaan untuk proyek .NET di Unictive, mencakup penamaan kelas, metode, variabel, file, dan folder untuk meningkatkan keterbacaan dan pemeliharaan kode.
published: true
date: 2025-12-29T00:00:00.000Z
tags: naming, convention, coding-style
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Konvensi Penamaan

Dokumen ini menjelaskan konvensi penamaan yang harus diikuti dalam seluruh proyek .NET di Unictive. Penamaan yang konsisten meningkatkan keterbacaan dan pemeliharaan kode.

## Prinsip Umum Penamaan

### 1. Gunakan Nama yang Deskriptif

Nama harus menjelaskan tujuan atau fungsi dari elemen tersebut.

```csharp
// BAIK - Deskriptif dan jelas
public class PemrosesanPembayaran { }
public void HitungTotalPesanan() { }
private int _jumlahPercobaan;

// BURUK - Tidak deskriptif
public class PP { }
public void Hitung() { }
private int _n;
```

### 2. Hindari Singkatan

Gunakan kata lengkap kecuali untuk singkatan yang sudah umum dikenal.

```csharp
// BAIK
public string NomorIdentifikasiPegawai { get; set; }
public void SinkronisasiData() { }

// BAIK - Singkatan umum diperbolehkan
public int Id { get; set; }
public string Url { get; set; }
public void KirimHttp() { }

// BURUK
public string NoIdPeg { get; set; }
public void SyncDt() { }
```

## Gaya Penulisan (Casing)

### PascalCase

Digunakan untuk: **Kelas, Antarmuka, Metode, Properti, Event, Enum, Namespace, Konstanta Publik**

```csharp
// Kelas
public class PelangganService { }

// Antarmuka (awali dengan 'I')
public interface IPelangganRepository { }

// Metode
public void ProsesPermintaan() { }
public async Task<Pelanggan> DapatkanPelangganAsync() { }

// Properti
public string NamaLengkap { get; set; }
public DateTime TanggalDibuat { get; init; }

// Enum
public enum StatusPesanan
{
    Baru,
    Diproses,
    Dikirim,
    Selesai,
    Dibatalkan
}

// Konstanta publik
public const int BatasanMaksimumItem = 100;
public static readonly TimeSpan WaktuTungguDefault = TimeSpan.FromSeconds(30);
```

### camelCase

Digunakan untuk: **Parameter metode, Variabel lokal**

```csharp
public void ProsesData(string dataMasukan, int jumlahItem)
{
    var hasilPemrosesan = new List<string>();
    int penghitung = 0;

    foreach (var item in koleksi)
    {
        var nilaiSementara = TransformasiItem(item);
        hasilPemrosesan.Add(nilaiSementara);
        penghitung++;
    }
}
```

### _camelCase (dengan awalan garis bawah)

Digunakan untuk: **Field privat, Field internal, Field protected**

```csharp
public class PelangganService
{
    private readonly IPelangganRepository _pelangganRepository;
    private readonly ILogger<PelangganService> _logger;
    private int _jumlahOperasi;

    protected string _konfigurasiDefault;
    internal bool _sudahDiinisialisasi;

    public PelangganService(
        IPelangganRepository pelangganRepository,
        ILogger<PelangganService> logger)
    {
        _pelangganRepository = pelangganRepository;
        _logger = logger;
    }
}
```

## Penamaan Berdasarkan Tipe

### Kelas dan Struct

```csharp
// Gunakan kata benda atau frasa kata benda
public class Pelanggan { }
public class PemrosesanPembayaran { }
public class PengaturanAplikasi { }

// Struct untuk tipe nilai kecil
public struct Koordinat { }
public struct RentangTanggal { }
```

### Antarmuka

```csharp
// Awali dengan huruf 'I'
public interface IPelangganRepository { }
public interface IEmailService { }
public interface INotifikasiHandler { }

// Gunakan kata sifat untuk kemampuan
public interface IDisposable { }
public interface ISerializable { }
public interface ICacheable { }
```

### Metode

```csharp
// Gunakan kata kerja atau frasa kata kerja
public void Simpan() { }
public void HapusPelanggan() { }
public bool ValidasiInput() { }
public Pelanggan DapatkanPelangganById() { }
public async Task KirimEmailAsync() { }

// Metode boolean - gunakan awalan Is, Has, Can, Should
public bool IsValid() { }
public bool HasPermission() { }
public bool CanExecute() { }
public bool ShouldProcess() { }
```

### Properti

```csharp
public class Pelanggan
{
    // Gunakan kata benda atau frasa kata benda
    public string Nama { get; set; }
    public string AlamatEmail { get; set; }
    public DateTime TanggalRegistrasi { get; init; }

    // Properti boolean - gunakan awalan Is, Has, Can
    public bool IsAktif { get; set; }
    public bool HasVerifikasiEmail { get; set; }
    public bool CanOrder { get; set; }
}
```

### Variabel Boolean

```csharp
// BAIK - Jelas menunjukkan kondisi
bool isValid = true;
bool hasItems = daftar.Count > 0;
bool canProceed = ValidasiSemua();
bool shouldRetry = percobaanSekarang < batasMaksimum;

// BURUK - Tidak jelas
bool valid = true;
bool flag = daftar.Count > 0;
bool status = ValidasiSemua();
```

### Koleksi

```csharp
// Gunakan bentuk jamak atau sufiks yang menunjukkan koleksi
public List<Pelanggan> DaftarPelanggan { get; set; }
public IEnumerable<Pesanan> Pesanan { get; set; }
public Dictionary<string, Produk> ProdukPerKode { get; set; }

// Untuk parameter dan variabel lokal
public void ProsesPelanggan(IEnumerable<Pelanggan> pelangganList)
{
    var pesananAktif = new List<Pesanan>();
}
```

### Konstanta dan Field Statis Readonly

```csharp
public class Konstanta
{
    // Konstanta - PascalCase
    public const int BatasMaksimumUkuranFile = 10485760; // 10 MB
    public const string FormatTanggalDefault = "yyyy-MM-dd";

    // Static readonly - PascalCase
    public static readonly TimeSpan WaktuTungguDefault = TimeSpan.FromSeconds(30);
    public static readonly string[] EkstensiGambarValid = { ".jpg", ".png", ".gif" };
}
```

### Enum

```csharp
// Nama enum - singular, PascalCase
public enum StatusPesanan
{
    Baru,
    MenungguPembayaran,
    Dibayar,
    Diproses,
    Dikirim,
    Selesai,
    Dibatalkan
}

// Flags enum - plural
[Flags]
public enum IzinPengguna
{
    Tidak = 0,
    Baca = 1,
    Tulis = 2,
    Hapus = 4,
    Admin = Baca | Tulis | Hapus
}
```

### Parameter Generik

```csharp
// Gunakan T untuk parameter tunggal
public class Repository<T> where T : class { }

// Gunakan nama deskriptif dengan awalan T untuk multiple parameter
public class Dictionary<TKey, TValue> { }
public interface IHandler<TRequest, TResponse> { }
public class Mapper<TSource, TDestination> { }
```

### Event dan Delegate

```csharp
// Event - gunakan kata kerja bentuk lampau atau sekarang
public event EventHandler<PelangganDibuatEventArgs> PelangganDibuat;
public event EventHandler<PesananEventArgs> PesananDiproses;
public event EventHandler DataBerubah;

// EventArgs
public class PelangganDibuatEventArgs : EventArgs
{
    public Pelanggan Pelanggan { get; init; }
    public DateTime WaktuDibuat { get; init; }
}

// Delegate
public delegate void ProsesHandler(object sender, ProsesEventArgs e);
public delegate Task<TResult> QueryAsync<TResult>(CancellationToken ct);
```

## Penamaan File dan Folder

### Struktur Proyek

```
src/
├── NamaProyek.Domain/
│   ├── Entities/
│   │   ├── Pelanggan.cs
│   │   ├── Pesanan.cs
│   │   └── ItemPesanan.cs
│   ├── Enums/
│   │   ├── StatusPesanan.cs
│   │   └── TipePelanggan.cs
│   ├── ValueObjects/
│   │   └── Alamat.cs
│   └── Interfaces/
│       └── IPelangganRepository.cs
├── NamaProyek.Application/
│   ├── Services/
│   │   ├── IPelangganService.cs
│   │   └── PelangganService.cs
│   ├── DTOs/
│   │   ├── PelangganDto.cs
│   │   └── PesananDto.cs
│   └── Validators/
│       └── PelangganValidator.cs
├── NamaProyek.Infrastructure/
│   ├── Repositories/
│   │   └── PelangganRepository.cs
│   ├── Data/
│   │   └── ApplicationDbContext.cs
│   └── Configurations/
│       └── PelangganConfiguration.cs
└── NamaProyek.API/
    ├── Controllers/
    │   └── PelangganController.cs
    └── Program.cs
```

### Penamaan File

```
// Nama file harus sama dengan nama tipe utama di dalamnya
Pelanggan.cs           -> public class Pelanggan
IPelangganService.cs   -> public interface IPelangganService
StatusPesanan.cs       -> public enum StatusPesanan

// Untuk partial class
Pelanggan.cs
Pelanggan.Validasi.cs
Pelanggan.Kalkulasi.cs
```

## Penamaan Khusus ASP.NET Core

### Controller

```csharp
// Akhiri dengan 'Controller'
public class PelangganController : ControllerBase { }
public class PesananController : ControllerBase { }
public class AuthController : ControllerBase { }
```

### Middleware

```csharp
// Akhiri dengan 'Middleware'
public class PenangananKesalahanMiddleware { }
public class LoggingMiddleware { }
public class AuthentikasiMiddleware { }
```

### Extension Methods

```csharp
// Akhiri dengan 'Extensions'
public static class StringExtensions { }
public static class ServiceCollectionExtensions { }
public static class QueryableExtensions { }
```

### Options/Settings

```csharp
// Akhiri dengan 'Options' atau 'Settings'
public class JwtOptions { }
public class EmailSettings { }
public class CacheOptions { }
```

## Singkatan yang Diperbolehkan

| Singkatan | Kepanjangan |
|-----------|-------------|
| Id | Identifier |
| Db | Database |
| Dto | Data Transfer Object |
| Api | Application Programming Interface |
| Url | Uniform Resource Locator |
| Uri | Uniform Resource Identifier |
| Http | Hypertext Transfer Protocol |
| Json | JavaScript Object Notation |
| Xml | Extensible Markup Language |
| Sql | Structured Query Language |
| Io | Input/Output |
| Ui | User Interface |

### Penulisan Singkatan

```csharp
// Singkatan 2 huruf - semua kapital
public string IO { get; set; }
public void KirimUI() { }

// Singkatan 3+ huruf - PascalCase
public int HttpStatusCode { get; set; }
public string JsonData { get; set; }
public void ProsesXmlFile() { }
```

## Referensi

- [Microsoft Naming Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/naming-guidelines)
- [C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)

---

*Terakhir diperbarui: Desember 2025*
