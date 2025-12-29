# Standar Kode .NET

Dokumen ini menjelaskan standar penulisan kode untuk proyek .NET di Unictive. Standar ini mengacu pada panduan resmi Microsoft dan praktik terbaik industri.

## Prinsip Umum

### 1. Keterbacaan di Atas Segalanya

Kode ditulis sekali tetapi dibaca berkali-kali. Prioritaskan keterbacaan dalam setiap keputusan penulisan kode.

```csharp
// BAIK - Jelas dan mudah dibaca
var pelangganAktif = daftarPelanggan.Where(p => p.StatusAktif).ToList();

// BURUK - Singkatan tidak jelas
var pa = dp.Where(p => p.SA).ToList();
```

### 2. Kesederhanaan

Pilih solusi yang paling sederhana yang dapat menyelesaikan masalah. Hindari rekayasa berlebihan (over-engineering).

```csharp
// BAIK - Sederhana dan langsung
public string DapatkanNamaLengkap()
{
    return $"{NamaDepan} {NamaBelakang}";
}

// BURUK - Terlalu rumit untuk tugas sederhana
public string DapatkanNamaLengkap()
{
    var pembangunString = new StringBuilder();
    if (!string.IsNullOrEmpty(NamaDepan))
    {
        pembangunString.Append(NamaDepan);
    }
    if (!string.IsNullOrEmpty(NamaBelakang))
    {
        if (pembangunString.Length > 0)
        {
            pembangunString.Append(" ");
        }
        pembangunString.Append(NamaBelakang);
    }
    return pembangunString.ToString();
}
```

## Penggunaan Fitur Bahasa Modern

### Gunakan Fitur C# Terbaru

Manfaatkan fitur bahasa C# modern untuk kode yang lebih ringkas dan aman.

```csharp
// Gunakan nullable reference types
public class Pelanggan
{
    public string Nama { get; set; } = string.Empty;
    public string? NomorTelepon { get; set; }  // Boleh null
}

// Gunakan pattern matching
public string DapatkanDeskripsiStatus(StatusPesanan status) => status switch
{
    StatusPesanan.Baru => "Pesanan baru diterima",
    StatusPesanan.Diproses => "Pesanan sedang diproses",
    StatusPesanan.Dikirim => "Pesanan dalam pengiriman",
    StatusPesanan.Selesai => "Pesanan telah selesai",
    _ => "Status tidak diketahui"
};

// Gunakan record untuk DTO
public record PelangganDto(int Id, string Nama, string Email);

// Gunakan init-only properties
public class Konfigurasi
{
    public string ConnectionString { get; init; } = string.Empty;
    public int BatasWaktu { get; init; } = 30;
}
```

### Gunakan Kata Kunci Bahasa, Bukan Tipe Runtime

```csharp
// BAIK
string nama = "Unictive";
int jumlah = 100;
object data = new();

// BURUK
String nama = "Unictive";
Int32 jumlah = 100;
Object data = new();
```

## Penanganan Pengecualian

### Tangkap Pengecualian Spesifik

Hindari menangkap pengecualian umum. Tangkap hanya pengecualian yang dapat ditangani dengan benar.

```csharp
// BAIK - Menangkap pengecualian spesifik
try
{
    var data = await File.ReadAllTextAsync(jalurFile);
}
catch (FileNotFoundException ex)
{
    _logger.LogWarning("File tidak ditemukan: {Path}", jalurFile);
    throw new DataTidakDitemukanException($"File konfigurasi tidak ada: {jalurFile}", ex);
}
catch (UnauthorizedAccessException ex)
{
    _logger.LogError(ex, "Tidak memiliki izin untuk membaca file: {Path}", jalurFile);
    throw;
}

// BURUK - Menangkap semua pengecualian
try
{
    var data = await File.ReadAllTextAsync(jalurFile);
}
catch (Exception ex)
{
    // Ini menyembunyikan masalah sebenarnya
    return null;
}
```

### Jangan Gunakan Pengecualian untuk Alur Kontrol

```csharp
// BAIK - Gunakan pengecekan kondisi
public Pelanggan? DapatkanPelanggan(int id)
{
    return _pelangganRepository.CariById(id);
}

// Pemanggil
var pelanggan = DapatkanPelanggan(id);
if (pelanggan is null)
{
    return NotFound();
}

// BURUK - Menggunakan pengecualian untuk alur normal
public Pelanggan DapatkanPelanggan(int id)
{
    var pelanggan = _pelangganRepository.CariById(id);
    if (pelanggan is null)
    {
        throw new PelangganTidakDitemukanException();
    }
    return pelanggan;
}
```

## Pemrograman Asinkron

### Gunakan async/await untuk Operasi I/O

```csharp
// BAIK - Menggunakan async/await dengan benar
public async Task<Pelanggan?> DapatkanPelangganAsync(int id, CancellationToken ct = default)
{
    return await _dbContext.Pelanggan
        .AsNoTracking()
        .FirstOrDefaultAsync(p => p.Id == id, ct);
}

// BAIK - Menghindari async jika tidak diperlukan
public Task<Pelanggan?> DapatkanPelangganAsync(int id)
{
    // Jika hanya meneruskan Task, tidak perlu async/await
    return _dbContext.Pelanggan.FindAsync(id).AsTask();
}
```

### Konvensi Penamaan Metode Asinkron

Semua metode asinkron harus memiliki akhiran `Async`.

```csharp
// BAIK
public async Task<List<Produk>> DapatkanSemuaProdukAsync()
public async Task SimpanPesananAsync(Pesanan pesanan)
public async Task<bool> ValidasiEmailAsync(string email)

// BURUK
public async Task<List<Produk>> DapatkanSemuaProduk()
public async Task Simpan(Pesanan pesanan)
```

### Selalu Teruskan CancellationToken

```csharp
public async Task<IActionResult> DapatkanData(CancellationToken cancellationToken)
{
    var data = await _layanan.ProsesDataAsync(cancellationToken);
    return Ok(data);
}
```

## Penggunaan LINQ

### Gunakan LINQ untuk Manipulasi Koleksi

```csharp
// BAIK - Menggunakan LINQ
var pelangganPremium = pelanggan
    .Where(p => p.TotalPembelian > 1000000)
    .OrderByDescending(p => p.TotalPembelian)
    .Select(p => new PelangganDto(p.Id, p.Nama, p.Email))
    .ToList();

// BURUK - Loop manual
var pelangganPremium = new List<PelangganDto>();
foreach (var p in pelanggan)
{
    if (p.TotalPembelian > 1000000)
    {
        pelangganPremium.Add(new PelangganDto(p.Id, p.Nama, p.Email));
    }
}
pelangganPremium.Sort((a, b) => b.TotalPembelian.CompareTo(a.TotalPembelian));
```

### Hindari Multiple Enumeration

```csharp
// BAIK - Materialisasi sekali
var daftarPelanggan = pelanggan.Where(p => p.Aktif).ToList();
var jumlah = daftarPelanggan.Count;
var pertama = daftarPelanggan.FirstOrDefault();

// BURUK - Multiple enumeration
var pelangganAktif = pelanggan.Where(p => p.Aktif);
var jumlah = pelangganAktif.Count();      // Enumerasi pertama
var pertama = pelangganAktif.FirstOrDefault();  // Enumerasi kedua
```

## Penggunaan Kurung Kurawal

### Selalu Gunakan Kurung Kurawal untuk Blok If

```csharp
// BAIK - Selalu gunakan kurung kurawal
if (kondisi)
{
    LakukanSesuatu();
}

// BAIK - Pernyataan sederhana satu baris diperbolehkan
if (kondisi)
    return nilai;

// BURUK - Membingungkan dan rawan kesalahan
if (kondisi)
    LakukanSesuatu();
    LakukanHalLain();  // Ini TIDAK dalam blok if!
```

## Penggunaan var

### Gunakan var Ketika Tipe Sudah Jelas

```csharp
// BAIK - Tipe jelas dari sisi kanan
var pelanggan = new Pelanggan();
var nama = pelanggan.Nama;  // Jelas bahwa ini string
var daftar = new List<Produk>();

// BAIK - Gunakan tipe eksplisit jika tidak jelas
Dictionary<string, List<Pelanggan>> pelangganPerWilayah = DapatkanPelangganPerWilayah();

// BURUK - Tidak jelas apa tipenya
var hasil = Proses();  // Apa tipe hasil?
```

## Komentar dan Dokumentasi

### Tulis Kode yang Menjelaskan Dirinya Sendiri

```csharp
// BURUK - Komentar menjelaskan yang sudah jelas
// Menambahkan pelanggan ke daftar
daftarPelanggan.Add(pelanggan);

// BAIK - Komentar menjelaskan "mengapa", bukan "apa"
// Menggunakan cache karena query ini sangat mahal dan data jarang berubah
var produk = await _cache.GetOrCreateAsync(kunciCache, async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);
    return await _repository.DapatkanSemuaProdukAsync();
});
```

### Gunakan XML Documentation untuk API Publik

```csharp
/// <summary>
/// Mendaftarkan pelanggan baru ke dalam sistem.
/// </summary>
/// <param name="permintaan">Data pendaftaran pelanggan.</param>
/// <param name="cancellationToken">Token untuk membatalkan operasi.</param>
/// <returns>Pelanggan yang berhasil didaftarkan.</returns>
/// <exception cref="EmailSudahTerdaftarException">
/// Dilempar jika alamat email sudah terdaftar.
/// </exception>
public async Task<Pelanggan> DaftarPelangganAsync(
    DaftarPelangganRequest permintaan,
    CancellationToken cancellationToken = default)
{
    // Implementasi
}
```

## Struktur File

### Satu Tipe per File

Setiap kelas, antarmuka, enum, atau record harus berada di file tersendiri dengan nama yang sama.

```
// Struktur folder yang benar
src/
├── Domain/
│   ├── Entities/
│   │   ├── Pelanggan.cs
│   │   ├── Pesanan.cs
│   │   └── Produk.cs
│   └── Enums/
│       ├── StatusPesanan.cs
│       └── TipePelanggan.cs
└── Application/
    └── Services/
        ├── IPelangganService.cs
        └── PelangganService.cs
```

## Referensi

- [Microsoft C# Coding Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [.NET Runtime Coding Style](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md)

---

*Terakhir diperbarui: Desember 2025*
