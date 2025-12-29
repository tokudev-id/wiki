---
title: Blazor dan Razor Pages
description: Panduan praktik terbaik membangun aplikasi web dengan Blazor dan Razor Pages di ASP.NET Core, mencakup arsitektur komponen, state management, JavaScript interop, dan optimisasi kinerja.
published: true
date: 2025-12-29T00:00:00.000Z
tags: blazor, razor, frontend
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Blazor dan Razor Pages

Dokumen ini menjelaskan praktik terbaik dalam membangun aplikasi web menggunakan Blazor dan Razor Pages di ASP.NET Core.

## Kapan Menggunakan Blazor vs Razor Pages

### Gunakan Razor Pages untuk:

- Halaman statis dengan interaktivitas minimal
- Halaman yang membutuhkan SEO optimal
- Formulir sederhana
- Halaman konten seperti blog atau dokumentasi
- Aplikasi dengan kebutuhan JavaScript minimal

### Gunakan Blazor untuk:

- Aplikasi dengan interaktivitas tinggi (dashboard, SPA)
- Komponen UI yang kompleks
- Real-time updates
- Aplikasi yang membutuhkan state management

### Pendekatan Hybrid (Direkomendasikan .NET 8+)

```csharp
// Gunakan Razor Pages untuk konten statis, embed Blazor untuk interaktivitas
@page "/produk/{id}"
@model ProdukModel

<h1>@Model.Produk.Nama</h1>
<p>@Model.Produk.Deskripsi</p>

<!-- Embed komponen Blazor untuk keranjang belanja -->
<component type="typeof(KeranjangBelanja)"
           render-mode="ServerPrerendered"
           param-ProdukId="@Model.Produk.Id" />
```

## Arsitektur Blazor

### Struktur Proyek

```
src/NamaProyek.Web/
├── Components/
│   ├── Layout/
│   │   ├── MainLayout.razor
│   │   └── NavMenu.razor
│   ├── Pages/
│   │   ├── Beranda.razor
│   │   ├── Pelanggan/
│   │   │   ├── DaftarPelanggan.razor
│   │   │   ├── DetailPelanggan.razor
│   │   │   └── FormPelanggan.razor
│   │   └── _Imports.razor
│   └── Shared/
│       ├── LoadingSpinner.razor
│       ├── ConfirmDialog.razor
│       └── Pagination.razor
├── Services/
│   ├── IPelangganService.cs
│   └── PelangganService.cs
├── wwwroot/
│   ├── css/
│   └── js/
└── Program.cs
```

### Siklus Hidup Komponen

```csharp
@page "/pelanggan/{Id:int}"

@code {
    [Parameter]
    public int Id { get; set; }

    private Pelanggan? _pelanggan;
    private bool _isLoading = true;

    // 1. Dipanggil saat komponen diinisialisasi
    protected override void OnInitialized()
    {
        // Inisialisasi sinkron
    }

    // 2. Dipanggil saat komponen diinisialisasi (async)
    protected override async Task OnInitializedAsync()
    {
        await MuatDataAsync();
    }

    // 3. Dipanggil saat parameter berubah
    protected override async Task OnParametersSetAsync()
    {
        // Dipanggil setiap kali parameter berubah
        await MuatDataAsync();
    }

    // 4. Dipanggil setelah render
    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            // Hanya pada render pertama
            // Baik untuk inisialisasi JavaScript interop
        }
    }

    // 5. Dipanggil setelah render (async)
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Async operation setelah render pertama
        }
    }

    private async Task MuatDataAsync()
    {
        _isLoading = true;
        try
        {
            _pelanggan = await PelangganService.DapatkanByIdAsync(Id);
        }
        finally
        {
            _isLoading = false;
        }
    }
}
```

## Praktik Terbaik Komponen Blazor

### 1. Pisahkan Komponen Presentasi dan Container

```csharp
// Komponen Presentasi - hanya menampilkan data
// Components/Shared/PelangganCard.razor
<div class="card">
    <div class="card-header">
        <h5>@Pelanggan.Nama</h5>
    </div>
    <div class="card-body">
        <p>@Pelanggan.Email</p>
        <button class="btn btn-primary" @onclick="OnLihatDetail">
            Lihat Detail
        </button>
    </div>
</div>

@code {
    [Parameter, EditorRequired]
    public PelangganDto Pelanggan { get; set; } = default!;

    [Parameter]
    public EventCallback OnLihatDetail { get; set; }
}

// Komponen Container - mengelola data dan logika
// Components/Pages/DaftarPelanggan.razor
@page "/pelanggan"
@inject IPelangganService PelangganService

<h1>Daftar Pelanggan</h1>

@if (_isLoading)
{
    <LoadingSpinner />
}
else
{
    <div class="row">
        @foreach (var pelanggan in _daftarPelanggan)
        {
            <div class="col-md-4">
                <PelangganCard
                    Pelanggan="pelanggan"
                    OnLihatDetail="() => NavigasiKeDetail(pelanggan.Id)" />
            </div>
        }
    </div>
}

@code {
    private List<PelangganDto> _daftarPelanggan = new();
    private bool _isLoading = true;

    protected override async Task OnInitializedAsync()
    {
        _daftarPelanggan = await PelangganService.DapatkanSemuaAsync();
        _isLoading = false;
    }
}
```

### 2. Gunakan Parameter dengan EditorRequired

```csharp
@code {
    // Wajib diisi saat menggunakan komponen
    [Parameter, EditorRequired]
    public string Judul { get; set; } = default!;

    // Opsional dengan nilai default
    [Parameter]
    public string Subjudul { get; set; } = string.Empty;

    // EventCallback untuk komunikasi ke parent
    [Parameter]
    public EventCallback<int> OnItemDipilih { get; set; }
}
```

### 3. Hindari Logika Kompleks di Komponen

```csharp
// BURUK - Logika bisnis di komponen
@code {
    private async Task SimpanPelanggan()
    {
        // Validasi
        if (string.IsNullOrEmpty(_pelanggan.Email))
        {
            _error = "Email wajib diisi";
            return;
        }

        // Cek duplikat
        var existing = await DbContext.Pelanggan
            .FirstOrDefaultAsync(p => p.Email == _pelanggan.Email);
        if (existing != null)
        {
            _error = "Email sudah terdaftar";
            return;
        }

        // Simpan
        DbContext.Pelanggan.Add(_pelanggan);
        await DbContext.SaveChangesAsync();
    }
}

// BAIK - Logika dipindahkan ke service
@inject IPelangganService PelangganService

@code {
    private async Task SimpanPelanggan()
    {
        try
        {
            await PelangganService.SimpanAsync(_pelanggan);
            NavigationManager.NavigateTo("/pelanggan");
        }
        catch (ValidationException ex)
        {
            _error = ex.Message;
        }
    }
}
```

### 4. Render Mode Agnostic (.NET 8+)

```csharp
// Komponen yang dapat berjalan di Server maupun WebAssembly
// Hindari penggunaan langsung HttpContext atau service yang server-only

// BAIK - Menggunakan abstraksi
@inject IUserService UserService

@code {
    private UserInfo? _user;

    protected override async Task OnInitializedAsync()
    {
        _user = await UserService.DapatkanUserSaatIniAsync();
    }
}

// Service yang berbeda untuk Server dan WASM
// Server:
public class ServerUserService : IUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public Task<UserInfo?> DapatkanUserSaatIniAsync()
    {
        var user = _httpContextAccessor.HttpContext?.User;
        // ...
    }
}

// WASM:
public class WasmUserService : IUserService
{
    private readonly AuthenticationStateProvider _authStateProvider;

    public async Task<UserInfo?> DapatkanUserSaatIniAsync()
    {
        var authState = await _authStateProvider.GetAuthenticationStateAsync();
        // ...
    }
}
```

## State Management

### 1. Component State (Paling Sederhana)

```csharp
@code {
    private int _counter = 0;
    private string _message = string.Empty;

    private void Increment()
    {
        _counter++;
    }
}
```

### 2. Cascading Values (Untuk State Bersama di Hierarki)

```csharp
// App.razor atau Layout
<CascadingValue Value="@_tema">
    <Router AppAssembly="typeof(App).Assembly">
        <!-- ... -->
    </Router>
</CascadingValue>

@code {
    private Tema _tema = new Tema { Mode = "dark" };
}

// Komponen child
@code {
    [CascadingParameter]
    public Tema Tema { get; set; } = default!;
}
```

### 3. State Container (Untuk State Global)

```csharp
// Services/AppState.cs
public class AppState
{
    private Pelanggan? _pelangganTerpilih;

    public Pelanggan? PelangganTerpilih
    {
        get => _pelangganTerpilih;
        set
        {
            _pelangganTerpilih = value;
            NotifyStateChanged();
        }
    }

    public event Action? OnChange;

    private void NotifyStateChanged() => OnChange?.Invoke();
}

// Program.cs
builder.Services.AddScoped<AppState>();

// Komponen yang menggunakan state
@inject AppState AppState
@implements IDisposable

<p>Pelanggan terpilih: @AppState.PelangganTerpilih?.Nama</p>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }

    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
}
```

## Razor Pages

### Struktur Halaman

```csharp
// Pages/Pelanggan/Index.cshtml
@page
@model IndexModel

<h1>Daftar Pelanggan</h1>

<table class="table">
    <thead>
        <tr>
            <th>Nama</th>
            <th>Email</th>
            <th>Aksi</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var pelanggan in Model.DaftarPelanggan)
        {
            <tr>
                <td>@pelanggan.Nama</td>
                <td>@pelanggan.Email</td>
                <td>
                    <a asp-page="./Detail" asp-route-id="@pelanggan.Id">Detail</a>
                </td>
            </tr>
        }
    </tbody>
</table>

// Pages/Pelanggan/Index.cshtml.cs
public class IndexModel : PageModel
{
    private readonly IPelangganService _pelangganService;

    public IndexModel(IPelangganService pelangganService)
    {
        _pelangganService = pelangganService;
    }

    public List<PelangganDto> DaftarPelanggan { get; set; } = new();

    public async Task OnGetAsync()
    {
        DaftarPelanggan = await _pelangganService.DapatkanSemuaAsync();
    }
}
```

### Handler Methods

```csharp
public class FormPelangganModel : PageModel
{
    [BindProperty]
    public PelangganInput Input { get; set; } = new();

    // GET request
    public async Task<IActionResult> OnGetAsync(int? id)
    {
        if (id.HasValue)
        {
            var pelanggan = await _service.DapatkanByIdAsync(id.Value);
            if (pelanggan == null)
            {
                return NotFound();
            }
            Input = MapToInput(pelanggan);
        }
        return Page();
    }

    // POST request
    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        await _service.SimpanAsync(Input);
        return RedirectToPage("./Index");
    }

    // Named handler - form dengan asp-page-handler="Hapus"
    public async Task<IActionResult> OnPostHapusAsync(int id)
    {
        await _service.HapusAsync(id);
        return RedirectToPage("./Index");
    }
}
```

### Validasi di Razor Pages

```csharp
// Model
public class PelangganInput
{
    [Required(ErrorMessage = "Nama wajib diisi")]
    [StringLength(100, ErrorMessage = "Nama maksimal 100 karakter")]
    [Display(Name = "Nama Lengkap")]
    public string Nama { get; set; } = string.Empty;

    [Required(ErrorMessage = "Email wajib diisi")]
    [EmailAddress(ErrorMessage = "Format email tidak valid")]
    public string Email { get; set; } = string.Empty;
}

// View
<form method="post">
    <div class="form-group">
        <label asp-for="Input.Nama"></label>
        <input asp-for="Input.Nama" class="form-control" />
        <span asp-validation-for="Input.Nama" class="text-danger"></span>
    </div>

    <div class="form-group">
        <label asp-for="Input.Email"></label>
        <input asp-for="Input.Email" class="form-control" />
        <span asp-validation-for="Input.Email" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-primary">Simpan</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

## Akses Database di Blazor

### Gunakan DbContext Factory

```csharp
// Program.cs
builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));

// Komponen
@inject IDbContextFactory<ApplicationDbContext> DbContextFactory

@code {
    private List<Pelanggan> _pelanggan = new();

    protected override async Task OnInitializedAsync()
    {
        // Buat instance baru untuk setiap operasi
        await using var context = await DbContextFactory.CreateDbContextAsync();
        _pelanggan = await context.Pelanggan.ToListAsync();
    }
}
```

### Lebih Baik: Gunakan Service Layer

```csharp
// Service
public class PelangganService : IPelangganService
{
    private readonly IDbContextFactory<ApplicationDbContext> _dbContextFactory;

    public PelangganService(IDbContextFactory<ApplicationDbContext> dbContextFactory)
    {
        _dbContextFactory = dbContextFactory;
    }

    public async Task<List<PelangganDto>> DapatkanSemuaAsync()
    {
        await using var context = await _dbContextFactory.CreateDbContextAsync();
        return await context.Pelanggan
            .AsNoTracking()
            .Select(p => new PelangganDto(p.Id, p.Nama, p.Email))
            .ToListAsync();
    }
}

// Komponen
@inject IPelangganService PelangganService

@code {
    private List<PelangganDto> _pelanggan = new();

    protected override async Task OnInitializedAsync()
    {
        _pelanggan = await PelangganService.DapatkanSemuaAsync();
    }
}
```

## JavaScript Interop

### Memanggil JavaScript dari C#

```csharp
// wwwroot/js/interop.js
window.showAlert = (message) => {
    alert(message);
};

window.getWindowWidth = () => {
    return window.innerWidth;
};

// Komponen
@inject IJSRuntime JSRuntime

@code {
    private async Task TampilkanAlert()
    {
        await JSRuntime.InvokeVoidAsync("showAlert", "Halo dari Blazor!");
    }

    private async Task<int> DapatkanLebarWindow()
    {
        return await JSRuntime.InvokeAsync<int>("getWindowWidth");
    }
}
```

### Memanggil C# dari JavaScript

```csharp
// Komponen
@inject IJSRuntime JSRuntime
@implements IAsyncDisposable

<button id="myButton">Klik Saya</button>

@code {
    private DotNetObjectReference<MyComponent>? _objRef;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            _objRef = DotNetObjectReference.Create(this);
            await JSRuntime.InvokeVoidAsync("setupButton", _objRef);
        }
    }

    [JSInvokable]
    public void HandleButtonClick()
    {
        // Dipanggil dari JavaScript
    }

    public async ValueTask DisposeAsync()
    {
        _objRef?.Dispose();
    }
}

// JavaScript
window.setupButton = (dotNetRef) => {
    document.getElementById('myButton').addEventListener('click', () => {
        dotNetRef.invokeMethodAsync('HandleButtonClick');
    });
};
```

## Keamanan

### Autentikasi dan Otorisasi

```csharp
// Blazor Server - menggunakan AuthorizeView
<AuthorizeView>
    <Authorized>
        <p>Selamat datang, @context.User.Identity?.Name!</p>
    </Authorized>
    <NotAuthorized>
        <p>Anda belum login.</p>
        <a href="Identity/Account/Login">Login</a>
    </NotAuthorized>
</AuthorizeView>

// Otorisasi berbasis role
<AuthorizeView Roles="Admin">
    <Authorized>
        <AdminPanel />
    </Authorized>
</AuthorizeView>

// Otorisasi di komponen
@attribute [Authorize(Roles = "Admin")]
@page "/admin"

// Otorisasi programatik
@inject AuthenticationStateProvider AuthenticationStateProvider

@code {
    private bool _isAdmin;

    protected override async Task OnInitializedAsync()
    {
        var authState = await AuthenticationStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;
        _isAdmin = user.IsInRole("Admin");
    }
}
```

## Optimisasi Kinerja

### 1. Virtualisasi untuk Daftar Besar

```csharp
@page "/pelanggan"

<h1>Daftar Pelanggan</h1>

<div style="height: 500px; overflow-y: auto;">
    <Virtualize Items="_pelanggan" Context="pelanggan">
        <div class="pelanggan-item">
            <span>@pelanggan.Nama</span>
            <span>@pelanggan.Email</span>
        </div>
    </Virtualize>
</div>

@code {
    private List<Pelanggan> _pelanggan = new();
}
```

### 2. Hindari Re-render yang Tidak Perlu

```csharp
// Implementasikan ShouldRender
@code {
    private bool _shouldRender = true;

    protected override bool ShouldRender() => _shouldRender;

    private void UpdateData()
    {
        // Hanya render jika ada perubahan
        var newData = FetchData();
        if (!DataEquals(_currentData, newData))
        {
            _currentData = newData;
            _shouldRender = true;
        }
    }
}
```

### 3. Gunakan @key untuk Koleksi

```csharp
@foreach (var item in items)
{
    <ItemComponent @key="item.Id" Item="item" />
}
```

## Referensi

- [ASP.NET Core Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/)
- [Blazor Performance Best Practices](https://learn.microsoft.com/en-us/aspnet/core/blazor/performance)
- [Razor Pages in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/razor-pages/)

---

*Terakhir diperbarui: Desember 2025*
