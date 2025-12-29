---
title: Panduan Testing
description: Panduan standar dan praktik terbaik untuk testing di proyek .NET, mencakup unit testing dengan xUnit, integration testing, FluentAssertions, Moq, dan Testcontainers.
published: true
date: 2025-12-29T00:00:00.000Z
tags: testing, xunit, integration-test
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Panduan Testing

Dokumen ini menjelaskan standar dan praktik terbaik untuk testing di proyek .NET. Testing yang baik adalah investasi untuk kualitas dan maintainability kode.

## Filosofi Testing

### Testing Mindset

```
"Tes bukan untuk membuktikan kode benar,
 tapi untuk menemukan bug sebelum user."
```

**Prinsip Utama:**
1. **Test apa yang penting** - Fokus pada business logic, bukan implementation detail
2. **Test harus cepat** - Tes lambat = tes yang tidak dijalankan
3. **Test harus reliable** - Tidak ada flaky tests
4. **Test sebagai dokumentasi** - Tes menjelaskan perilaku yang diharapkan

### Piramida Testing

```
         /\
        /  \        E2E Tests (sedikit, lambat)
       /----\
      /      \      Integration Tests (sedang)
     /--------\
    /          \    Unit Tests (banyak, cepat)
   /______________\
```

| Tipe | Proporsi | Kecepatan | Cakupan |
|------|----------|-----------|---------|
| Unit | 70% | Milidetik | Kelas/Metode |
| Integration | 20% | Detik | Modul/API |
| E2E | 10% | Menit | Sistem lengkap |

## Framework dan Library

### Stack Testing yang Direkomendasikan

```xml
<!-- File .csproj untuk proyek test -->
<ItemGroup>
  <!-- Framework -->
  <PackageReference Include="xunit" Version="2.9.*" />
  <PackageReference Include="xunit.runner.visualstudio" Version="2.8.*" />
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />

  <!-- Assertions -->
  <PackageReference Include="FluentAssertions" Version="6.*" />

  <!-- Mocking -->
  <PackageReference Include="Moq" Version="4.*" />
  <PackageReference Include="NSubstitute" Version="5.*" /> <!-- Alternatif -->

  <!-- Test Data -->
  <PackageReference Include="Bogus" Version="35.*" />
  <PackageReference Include="AutoFixture" Version="4.*" />

  <!-- Integration Testing -->
  <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.*" />
  <PackageReference Include="Testcontainers" Version="3.*" />
  <PackageReference Include="Testcontainers.MsSql" Version="3.*" />
  <PackageReference Include="Respawn" Version="6.*" />
</ItemGroup>
```

## Unit Testing

### Struktur Proyek Test

```
tests/
├── NamaProyek.UnitTests/
│   ├── Application/
│   │   └── Services/
│   │       ├── PelangganServiceTests.cs
│   │       └── PesananServiceTests.cs
│   ├── Domain/
│   │   └── Entities/
│   │       └── PelangganTests.cs
│   └── Common/
│       ├── Fixtures/
│       └── Builders/
└── NamaProyek.IntegrationTests/
    ├── API/
    │   └── Controllers/
    │       └── PelangganControllerTests.cs
    └── Infrastructure/
        └── Repositories/
            └── PelangganRepositoryTests.cs
```

### Pola Arrange-Act-Assert (AAA)

```csharp
public class PelangganServiceTests
{
    [Fact]
    public async Task BuatAsync_DataValid_ReturnPelangganBaru()
    {
        // Arrange - Siapkan data dan dependencies
        var unitOfWorkMock = new Mock<IUnitOfWork>();
        var request = new BuatPelangganRequest("John Doe", "john@example.com", null);

        unitOfWorkMock
            .Setup(x => x.Pelanggan.DapatkanByEmailAsync(It.IsAny<string>(), default))
            .ReturnsAsync((Pelanggan?)null);

        var sut = new PelangganService(unitOfWorkMock.Object, _mapper, _validator, _logger);

        // Act - Eksekusi aksi yang diuji
        var result = await sut.BuatAsync(request);

        // Assert - Verifikasi hasil
        result.Should().NotBeNull();
        result.Nama.Should().Be("John Doe");
        result.Email.Should().Be("john@example.com");

        unitOfWorkMock.Verify(x => x.SimpanPerubahanAsync(default), Times.Once);
    }
}
```

### Naming Convention untuk Test

```csharp
// Format: [MethodName]_[Scenario]_[ExpectedBehavior]

[Fact]
public async Task DapatkanByIdAsync_PelangganAda_ReturnPelangganDto()

[Fact]
public async Task DapatkanByIdAsync_PelangganTidakAda_ReturnNull()

[Fact]
public async Task BuatAsync_EmailSudahTerdaftar_ThrowConflictException()

[Fact]
public async Task UpdateAsync_DataValid_UpdateBerhasil()

[Fact]
public async Task HapusAsync_PelangganPunyaPesanan_ThrowConflictException()
```

### Menggunakan FluentAssertions

```csharp
using FluentAssertions;

[Fact]
public async Task DapatkanSemuaAsync_AdaData_ReturnListPelanggan()
{
    // Arrange
    var pelanggan = await _sut.DapatkanSemuaAsync();

    // Assert dengan FluentAssertions
    pelanggan.Should().NotBeNull();
    pelanggan.Should().HaveCount(3);
    pelanggan.Should().Contain(p => p.Nama == "John Doe");
    pelanggan.Should().OnlyContain(p => p.IsAktif);
    pelanggan.Should().BeInAscendingOrder(p => p.Nama);

    // Exception assertions
    Func<Task> action = async () => await _sut.DapatkanByIdAsync(-1);
    await action.Should().ThrowAsync<ArgumentException>()
        .WithMessage("*tidak valid*");
}
```

### Menggunakan Moq

```csharp
public class PelangganServiceTests
{
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly Mock<IPelangganRepository> _pelangganRepoMock;
    private readonly PelangganService _sut;

    public PelangganServiceTests()
    {
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _pelangganRepoMock = new Mock<IPelangganRepository>();

        _unitOfWorkMock.Setup(x => x.Pelanggan).Returns(_pelangganRepoMock.Object);

        _sut = new PelangganService(_unitOfWorkMock.Object, /* ... */);
    }

    [Fact]
    public async Task BuatAsync_EmailBaru_MenyimpanKePelangganRepository()
    {
        // Arrange
        var request = new BuatPelangganRequest("Test", "test@test.com", null);
        _pelangganRepoMock
            .Setup(x => x.DapatkanByEmailAsync(It.IsAny<string>(), default))
            .ReturnsAsync((Pelanggan?)null);

        // Act
        await _sut.BuatAsync(request);

        // Assert - Verify method dipanggil dengan parameter yang benar
        _pelangganRepoMock.Verify(
            x => x.TambahAsync(
                It.Is<Pelanggan>(p =>
                    p.Nama == "Test" &&
                    p.Email == "test@test.com"),
                default),
            Times.Once);

        _unitOfWorkMock.Verify(x => x.SimpanPerubahanAsync(default), Times.Once);
    }

    [Fact]
    public async Task BuatAsync_EmailSudahAda_TidakMenyimpan()
    {
        // Arrange
        var existingPelanggan = new Pelanggan { Email = "existing@test.com" };
        _pelangganRepoMock
            .Setup(x => x.DapatkanByEmailAsync("existing@test.com", default))
            .ReturnsAsync(existingPelanggan);

        var request = new BuatPelangganRequest("Test", "existing@test.com", null);

        // Act & Assert
        await Assert.ThrowsAsync<ConflictException>(() => _sut.BuatAsync(request));

        _pelangganRepoMock.Verify(
            x => x.TambahAsync(It.IsAny<Pelanggan>(), default),
            Times.Never);
    }
}
```

### Menggunakan Bogus untuk Test Data

```csharp
using Bogus;

public class PelangganFaker : Faker<Pelanggan>
{
    public PelangganFaker()
    {
        RuleFor(p => p.Id, f => f.IndexFaker + 1);
        RuleFor(p => p.Nama, f => f.Name.FullName());
        RuleFor(p => p.Email, f => f.Internet.Email());
        RuleFor(p => p.NomorTelepon, f => f.Phone.PhoneNumber("+62###########"));
        RuleFor(p => p.TanggalRegistrasi, f => f.Date.Past(2));
        RuleFor(p => p.IsAktif, f => f.Random.Bool(0.9f));
    }
}

// Penggunaan
[Fact]
public async Task DapatkanSemuaAsync_Return10Pelanggan()
{
    // Arrange
    var faker = new PelangganFaker();
    var pelangganList = faker.Generate(10);

    _pelangganRepoMock
        .Setup(x => x.DapatkanSemuaAsync(default))
        .ReturnsAsync(pelangganList);

    // Act
    var result = await _sut.DapatkanSemuaAsync();

    // Assert
    result.Should().HaveCount(10);
}
```

### Test Data Builder Pattern

```csharp
public class PelangganBuilder
{
    private readonly Pelanggan _pelanggan;

    public PelangganBuilder()
    {
        _pelanggan = new Pelanggan
        {
            Id = 1,
            Nama = "Default User",
            Email = "default@test.com",
            IsAktif = true,
            TanggalRegistrasi = DateTime.UtcNow
        };
    }

    public PelangganBuilder WithId(int id)
    {
        _pelanggan.Id = id;
        return this;
    }

    public PelangganBuilder WithNama(string nama)
    {
        _pelanggan.Nama = nama;
        return this;
    }

    public PelangganBuilder WithEmail(string email)
    {
        _pelanggan.Email = email;
        return this;
    }

    public PelangganBuilder NonAktif()
    {
        _pelanggan.IsAktif = false;
        return this;
    }

    public Pelanggan Build() => _pelanggan;
}

// Penggunaan
[Fact]
public void Test_WithBuilder()
{
    var pelanggan = new PelangganBuilder()
        .WithNama("John Doe")
        .WithEmail("john@test.com")
        .NonAktif()
        .Build();

    pelanggan.Nama.Should().Be("John Doe");
    pelanggan.IsAktif.Should().BeFalse();
}
```

## Integration Testing

### Setup WebApplicationFactory

```csharp
// IntegrationTests/CustomWebApplicationFactory.cs
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Hapus registrasi DbContext asli
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            // Gunakan In-Memory Database untuk testing
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });

            // Seed data test
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.EnsureCreated();
            SeedTestData(db);
        });

        builder.UseEnvironment("Testing");
    }

    private void SeedTestData(ApplicationDbContext db)
    {
        db.Pelanggan.AddRange(
            new Pelanggan { Id = 1, Nama = "Test User 1", Email = "test1@test.com", IsAktif = true },
            new Pelanggan { Id = 2, Nama = "Test User 2", Email = "test2@test.com", IsAktif = true }
        );
        db.SaveChanges();
    }
}
```

### Integration Test dengan WebApplicationFactory

```csharp
public class PelangganApiTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public PelangganApiTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetPelanggan_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/pelanggan");

        // Assert
        response.EnsureSuccessStatusCode();
        response.Content.Headers.ContentType?.MediaType.Should().Be("application/json");
    }

    [Fact]
    public async Task GetPelangganById_PelangganAda_ReturnPelanggan()
    {
        // Act
        var response = await _client.GetAsync("/api/pelanggan/1");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var content = await response.Content.ReadFromJsonAsync<PelangganDto>();
        content.Should().NotBeNull();
        content!.Id.Should().Be(1);
    }

    [Fact]
    public async Task GetPelangganById_PelangganTidakAda_Return404()
    {
        // Act
        var response = await _client.GetAsync("/api/pelanggan/999");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task PostPelanggan_DataValid_Return201()
    {
        // Arrange
        var request = new BuatPelangganRequest("New User", "new@test.com", null);

        // Act
        var response = await _client.PostAsJsonAsync("/api/pelanggan", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var content = await response.Content.ReadFromJsonAsync<PelangganDto>();
        content!.Nama.Should().Be("New User");
    }

    [Fact]
    public async Task PostPelanggan_EmailInvalid_Return400()
    {
        // Arrange
        var request = new BuatPelangganRequest("Test", "invalid-email", null);

        // Act
        var response = await _client.PostAsJsonAsync("/api/pelanggan", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### Menggunakan Testcontainers

```csharp
public class DatabaseIntegrationTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer;
    private ApplicationDbContext _dbContext = null!;

    public DatabaseIntegrationTests()
    {
        _sqlContainer = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        _dbContext = new ApplicationDbContext(options);
        await _dbContext.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _dbContext.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task Repository_TambahPelanggan_BerhasilDisimpan()
    {
        // Arrange
        var repository = new PelangganRepository(_dbContext);
        var pelanggan = new Pelanggan
        {
            Nama = "Test User",
            Email = "test@test.com",
            IsAktif = true
        };

        // Act
        await repository.TambahAsync(pelanggan);
        await _dbContext.SaveChangesAsync();

        // Assert
        var saved = await _dbContext.Pelanggan.FindAsync(pelanggan.Id);
        saved.Should().NotBeNull();
        saved!.Nama.Should().Be("Test User");
    }
}
```

### Reset Database dengan Respawn

```csharp
public class IntegrationTestBase : IAsyncLifetime
{
    protected readonly CustomWebApplicationFactory Factory;
    protected readonly HttpClient Client;
    private Respawner _respawner = null!;

    public IntegrationTestBase(CustomWebApplicationFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient();
    }

    public async Task InitializeAsync()
    {
        using var scope = Factory.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();

        _respawner = await Respawner.CreateAsync(
            dbContext.Database.GetConnectionString()!,
            new RespawnerOptions
            {
                TablesToIgnore = new[] { "__EFMigrationsHistory" }
            });
    }

    public Task DisposeAsync() => Task.CompletedTask;

    protected async Task ResetDatabase()
    {
        using var scope = Factory.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        await _respawner.ResetAsync(dbContext.Database.GetConnectionString()!);
    }
}
```

## Test Coverage

### Apa yang Harus Ditest?

**Wajib Ditest:**
- Business logic di Service layer
- Validasi input
- Edge cases dan error handling
- Domain entity behavior

**Tidak Perlu Ditest:**
- Framework code (ASP.NET Core, EF Core)
- Simple CRUD tanpa logic
- Private methods (test melalui public method)
- Third-party libraries

### Minimum Coverage Target

| Layer | Target | Prioritas |
|-------|--------|-----------|
| Domain | 90% | Tinggi |
| Application/Service | 80% | Tinggi |
| Infrastructure | 60% | Sedang |
| API/Controllers | 50% | Rendah |

### Jalankan Coverage Report

```bash
# Install tool
dotnet tool install -g dotnet-reportgenerator-globaltool

# Jalankan test dengan coverage
dotnet test --collect:"XPlat Code Coverage"

# Generate report HTML
reportgenerator \
  -reports:"**/coverage.cobertura.xml" \
  -targetdir:"coveragereport" \
  -reporttypes:Html
```

## Menjalankan Tests

### Command Line

```bash
# Semua tests
dotnet test

# Test tertentu
dotnet test --filter "FullyQualifiedName~PelangganServiceTests"

# Dengan output detail
dotnet test --logger "console;verbosity=detailed"

# Parallel execution
dotnet test --parallel

# Dengan coverage
dotnet test --collect:"XPlat Code Coverage"
```

### Kategori Test

```csharp
// Menandai test dengan Trait
[Fact]
[Trait("Category", "Unit")]
public void UnitTest() { }

[Fact]
[Trait("Category", "Integration")]
public void IntegrationTest() { }

[Fact]
[Trait("Category", "Slow")]
public void SlowTest() { }

// Menjalankan kategori tertentu
// dotnet test --filter "Category=Unit"
// dotnet test --filter "Category!=Slow"
```

## Anti-Patterns yang Harus Dihindari

### 1. Flaky Tests

```csharp
// BURUK - Bergantung pada waktu nyata
[Fact]
public void Test_TanggalHariIni()
{
    var result = service.GetTodayDate();
    Assert.Equal(DateTime.Today, result);  // Bisa gagal di tengah malam
}

// BAIK - Inject time provider
[Fact]
public void Test_TanggalHariIni()
{
    var timeProvider = new FakeTimeProvider(new DateTime(2024, 1, 15));
    var service = new Service(timeProvider);

    var result = service.GetTodayDate();

    Assert.Equal(new DateTime(2024, 1, 15), result);
}
```

### 2. Test Implementation, bukan Behavior

```csharp
// BURUK - Test implementation detail
[Fact]
public void Test_MemanggilRepository()
{
    // Ini test bahwa repository dipanggil, bukan behavior
    _repoMock.Verify(x => x.DapatkanByIdAsync(1), Times.Once);
}

// BAIK - Test behavior
[Fact]
public void Test_MengembalikanPelanggan()
{
    // Test apa yang dikembalikan, bukan bagaimana
    var result = await _sut.DapatkanByIdAsync(1);
    result.Should().NotBeNull();
    result.Nama.Should().Be("Expected Name");
}
```

### 3. Shared State

```csharp
// BURUK - Shared state antar test
public class BadTests
{
    private static int _counter = 0;

    [Fact]
    public void Test1()
    {
        _counter++;
        Assert.Equal(1, _counter);  // Bisa gagal jika Test2 jalan duluan
    }

    [Fact]
    public void Test2()
    {
        _counter++;
        Assert.Equal(1, _counter);  // Bisa gagal jika Test1 jalan duluan
    }
}

// BAIK - Isolated state
public class GoodTests
{
    [Fact]
    public void Test1()
    {
        var counter = 0;
        counter++;
        Assert.Equal(1, counter);
    }
}
```

## Referensi

- [xUnit.net Documentation](https://xunit.net/)
- [FluentAssertions](https://fluentassertions.com/)
- [Moq Documentation](https://github.com/moq/moq4)
- [Microsoft Unit Testing Best Practices](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
- [Integration Testing in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [Testcontainers for .NET](https://testcontainers.com/guides/getting-started-with-testcontainers-for-dotnet/)

---

*Terakhir diperbarui: Desember 2025*
