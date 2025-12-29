---
title: Kesadaran Keamanan
description: Panduan praktik keamanan untuk pengembang .NET berdasarkan OWASP Top 10, mencakup autentikasi, validasi input, manajemen secrets, dan keamanan upload file.
published: true
date: 2025-12-29T00:00:00.000Z
tags: security, owasp, authentication
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Kesadaran Keamanan untuk Pengembang

Dokumen ini menjelaskan praktik keamanan yang harus dipahami dan diterapkan oleh setiap pengembang .NET.

## OWASP Top 10

### 1. Broken Access Control

**Masalah:** User dapat mengakses resource yang bukan miliknya.

```csharp
// BURUK - Tidak ada pengecekan ownership
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _context.Orders.FindAsync(id);
    return Ok(order);  // Siapapun bisa akses order apapun!
}

// BAIK - Verifikasi ownership
[HttpGet("{id}")]
[Authorize]
public async Task<IActionResult> GetOrder(int id)
{
    var userId = User.GetUserId();
    var order = await _context.Orders
        .FirstOrDefaultAsync(o => o.Id == id && o.UserId == userId);

    if (order is null)
        return NotFound();

    return Ok(order);
}
```

### 2. Cryptographic Failures

**Masalah:** Data sensitif tidak dienkripsi dengan benar.

```csharp
// BURUK - Menyimpan password dalam plaintext
public void SaveUser(User user)
{
    user.Password = password;  // JANGAN!
    _context.Users.Add(user);
}

// BAIK - Hash password dengan salt
public void SaveUser(User user, string password)
{
    user.PasswordHash = BCrypt.Net.BCrypt.HashPassword(password, workFactor: 12);
    _context.Users.Add(user);
}

// Verifikasi password
public bool VerifyPassword(User user, string password)
{
    return BCrypt.Net.BCrypt.Verify(password, user.PasswordHash);
}
```

### 3. Injection

**Masalah:** Input user dieksekusi sebagai code/query.

```csharp
// BURUK - SQL Injection
public async Task<User?> GetUser(string email)
{
    var query = $"SELECT * FROM Users WHERE Email = '{email}'";
    return await _context.Users.FromSqlRaw(query).FirstOrDefaultAsync();
    // Input: ' OR '1'='1 → SELECT * FROM Users WHERE Email = '' OR '1'='1'
}

// BAIK - Parameterized query (EF Core otomatis)
public async Task<User?> GetUser(string email)
{
    return await _context.Users.FirstOrDefaultAsync(u => u.Email == email);
}

// BAIK - Jika harus raw SQL, gunakan parameter
public async Task<User?> GetUser(string email)
{
    return await _context.Users
        .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}")
        .FirstOrDefaultAsync();
}
```

### 4. Insecure Design

**Masalah:** Desain aplikasi tidak memperhitungkan keamanan.

```csharp
// BURUK - Business logic di client-side
// Frontend: if (user.role === 'admin') showAdminButton();
// Backend tidak memvalidasi!

// BAIK - Validasi di server
[HttpPost("admin/delete-user")]
[Authorize(Roles = "Admin")]  // Validasi di server
public async Task<IActionResult> DeleteUser(int userId)
{
    // Double-check role
    if (!User.IsInRole("Admin"))
        return Forbid();

    await _userService.DeleteAsync(userId);
    return NoContent();
}
```

### 5. Security Misconfiguration

**Masalah:** Konfigurasi keamanan tidak optimal.

```csharp
// Program.cs - Konfigurasi keamanan yang benar

// HTTPS redirect
app.UseHttpsRedirection();

// Security headers
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
    await next();
});

// Disable detailed errors di production
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// Jangan expose server info
app.Use(async (context, next) =>
{
    context.Response.Headers.Remove("Server");
    context.Response.Headers.Remove("X-Powered-By");
    await next();
});
```

### 6. Vulnerable Components

**Masalah:** Menggunakan library dengan kerentanan yang diketahui.

```bash
# Scan dependencies untuk vulnerabilities
dotnet list package --vulnerable

# Gunakan tools seperti:
# - Snyk
# - OWASP Dependency-Check
# - GitHub Dependabot

# Update packages secara regular
dotnet outdated
dotnet add package PackageName --version <latest>
```

### 7. Authentication Failures

**Masalah:** Implementasi autentikasi yang lemah.

```csharp
// Konfigurasi JWT yang aman
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = config["Jwt:Issuer"],
            ValidAudience = config["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Key"]!)),
            ClockSkew = TimeSpan.Zero  // Tidak ada toleransi waktu
        };
    });

// Generate token dengan expiry yang wajar
public string GenerateToken(User user)
{
    var claims = new[]
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(ClaimTypes.Role, user.Role)
    };

    var token = new JwtSecurityToken(
        issuer: _config["Jwt:Issuer"],
        audience: _config["Jwt:Audience"],
        claims: claims,
        expires: DateTime.UtcNow.AddHours(1),  // Short-lived
        signingCredentials: new SigningCredentials(
            new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]!)),
            SecurityAlgorithms.HmacSha256));

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

### 8. Software and Data Integrity Failures

**Masalah:** Tidak memvalidasi integritas data/software.

```csharp
// Validasi integritas file upload
public async Task<bool> ValidateFileIntegrity(IFormFile file, string expectedHash)
{
    using var sha256 = SHA256.Create();
    using var stream = file.OpenReadStream();
    var hash = await sha256.ComputeHashAsync(stream);
    var hashString = Convert.ToHexString(hash);

    return hashString.Equals(expectedHash, StringComparison.OrdinalIgnoreCase);
}

// Signed cookies untuk mencegah tampering
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .ProtectKeysWithCertificate(certificate);
```

### 9. Logging and Monitoring Failures

**Masalah:** Tidak ada logging untuk security events.

```csharp
// Log security events
public class AuthService
{
    private readonly ILogger<AuthService> _logger;

    public async Task<LoginResult> LoginAsync(LoginRequest request)
    {
        var user = await _userRepository.GetByEmailAsync(request.Email);

        if (user is null)
        {
            _logger.LogWarning("Login gagal: email {Email} tidak ditemukan", request.Email);
            return LoginResult.Failed("Email atau password salah");
        }

        if (!VerifyPassword(user, request.Password))
        {
            _logger.LogWarning(
                "Login gagal: password salah untuk {Email}, IP: {IP}",
                request.Email,
                GetClientIp());

            // Track failed attempts
            await _lockoutService.RecordFailedAttemptAsync(request.Email);

            return LoginResult.Failed("Email atau password salah");
        }

        _logger.LogInformation(
            "Login berhasil: {UserId} ({Email}), IP: {IP}",
            user.Id,
            user.Email,
            GetClientIp());

        return LoginResult.Success(GenerateToken(user));
    }
}
```

### 10. Server-Side Request Forgery (SSRF)

**Masalah:** Aplikasi mengakses URL yang dikontrol oleh attacker.

```csharp
// BURUK - User bisa akses internal network
[HttpGet("fetch")]
public async Task<IActionResult> FetchUrl(string url)
{
    var content = await _httpClient.GetStringAsync(url);  // BERBAHAYA!
    return Ok(content);
}

// BAIK - Validasi dan whitelist URL
[HttpGet("fetch")]
public async Task<IActionResult> FetchUrl(string url)
{
    if (!IsAllowedUrl(url))
        return BadRequest("URL tidak diizinkan");

    var content = await _httpClient.GetStringAsync(url);
    return Ok(content);
}

private bool IsAllowedUrl(string url)
{
    var allowedDomains = new[] { "api.trusted.com", "cdn.trusted.com" };

    if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
        return false;

    // Hanya HTTPS
    if (uri.Scheme != "https")
        return false;

    // Hanya domain yang diizinkan
    return allowedDomains.Contains(uri.Host);
}
```

## Input Validation

### Validasi di Setiap Layer

```csharp
// 1. Model Binding Validation
public class CreateUserRequest
{
    [Required(ErrorMessage = "Nama wajib diisi")]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [MinLength(8)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&]).+$",
        ErrorMessage = "Password harus mengandung huruf besar, kecil, angka, dan karakter khusus")]
    public string Password { get; set; } = string.Empty;
}

// 2. FluentValidation (lebih fleksibel)
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator(IUserRepository userRepo)
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .Length(2, 100)
            .Matches(@"^[\p{L}\s'-]+$").WithMessage("Nama hanya boleh berisi huruf");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(async (email, ct) => !await userRepo.EmailExistsAsync(email, ct))
            .WithMessage("Email sudah terdaftar");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Must(HasUpperCase).WithMessage("Password harus mengandung huruf besar")
            .Must(HasLowerCase).WithMessage("Password harus mengandung huruf kecil")
            .Must(HasDigit).WithMessage("Password harus mengandung angka")
            .Must(HasSpecialChar).WithMessage("Password harus mengandung karakter khusus");
    }
}

// 3. Domain Validation
public class Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Email Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            throw new DomainException("Email tidak boleh kosong");

        email = email.Trim().ToLowerInvariant();

        if (!IsValidEmail(email))
            throw new DomainException("Format email tidak valid");

        return new Email(email);
    }
}
```

### Sanitasi Output

```csharp
// Untuk HTML output - mencegah XSS
public string SanitizeForHtml(string input)
{
    return HtmlEncoder.Default.Encode(input);
}

// Untuk JavaScript
public string SanitizeForJs(string input)
{
    return JavaScriptEncoder.Default.Encode(input);
}

// Untuk URL
public string SanitizeForUrl(string input)
{
    return UrlEncoder.Default.Encode(input);
}

// Razor otomatis encode HTML
<p>@Model.UserInput</p>  <!-- Aman, auto-encoded -->
<p>@Html.Raw(Model.UserInput)</p>  <!-- BAHAYA! Tidak di-encode -->
```

## Secrets Management

### Jangan Hardcode Secrets

```csharp
// BURUK
var connectionString = "Server=prod;Password=super_secret_123";
var apiKey = "sk-abc123xyz";

// BAIK - Gunakan configuration
var connectionString = builder.Configuration.GetConnectionString("Default");
var apiKey = builder.Configuration["ExternalApi:Key"];

// BAIK - Gunakan User Secrets (development)
// dotnet user-secrets set "ExternalApi:Key" "sk-abc123xyz"

// BAIK - Gunakan environment variables (production)
// export ExternalApi__Key="sk-abc123xyz"

// BAIK - Gunakan Azure Key Vault (production)
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

### appsettings.json

```json
{
  "ConnectionStrings": {
    "Default": ""  // Jangan isi di file ini!
  },
  "Jwt": {
    "Issuer": "https://myapp.com",
    "Audience": "https://myapp.com",
    "Key": ""  // Jangan isi di file ini!
  }
}
```

### .gitignore

```gitignore
# Secrets
appsettings.Development.json
appsettings.Local.json
*.pfx
*.key
*.pem
secrets/
.env
```

## File Upload Security

```csharp
public class FileUploadService
{
    private readonly string[] _allowedExtensions = { ".jpg", ".jpeg", ".png", ".pdf" };
    private readonly string[] _allowedMimeTypes = { "image/jpeg", "image/png", "application/pdf" };
    private const long MaxFileSize = 10 * 1024 * 1024; // 10 MB

    public async Task<string> UploadAsync(IFormFile file)
    {
        // 1. Validasi ukuran
        if (file.Length == 0 || file.Length > MaxFileSize)
            throw new ValidationException($"Ukuran file harus antara 1 byte dan {MaxFileSize} bytes");

        // 2. Validasi extension
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!_allowedExtensions.Contains(extension))
            throw new ValidationException($"Extension {extension} tidak diizinkan");

        // 3. Validasi MIME type
        if (!_allowedMimeTypes.Contains(file.ContentType))
            throw new ValidationException($"Content type {file.ContentType} tidak diizinkan");

        // 4. Validasi content (magic bytes)
        if (!await IsValidFileContentAsync(file))
            throw new ValidationException("Konten file tidak valid");

        // 5. Generate nama file yang aman
        var safeFileName = $"{Guid.NewGuid()}{extension}";

        // 6. Simpan di luar web root
        var uploadPath = Path.Combine(_config["UploadPath"]!, safeFileName);
        await using var stream = new FileStream(uploadPath, FileMode.Create);
        await file.CopyToAsync(stream);

        return safeFileName;
    }

    private async Task<bool> IsValidFileContentAsync(IFormFile file)
    {
        // Check magic bytes
        var magicBytes = new Dictionary<string, byte[]>
        {
            { ".jpg", new byte[] { 0xFF, 0xD8, 0xFF } },
            { ".png", new byte[] { 0x89, 0x50, 0x4E, 0x47 } },
            { ".pdf", new byte[] { 0x25, 0x50, 0x44, 0x46 } }
        };

        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!magicBytes.TryGetValue(extension, out var expected))
            return false;

        using var reader = new BinaryReader(file.OpenReadStream());
        var header = reader.ReadBytes(expected.Length);

        return header.SequenceEqual(expected);
    }
}
```

## Security Checklist

### Development

- [ ] Tidak ada secrets di source code
- [ ] Input validation di setiap layer
- [ ] Parameterized queries untuk database
- [ ] Output encoding untuk prevent XSS
- [ ] HTTPS enforced
- [ ] Secure cookies (HttpOnly, Secure, SameSite)

### Authentication & Authorization

- [ ] Password di-hash dengan bcrypt/argon2
- [ ] Rate limiting untuk login
- [ ] Account lockout setelah failed attempts
- [ ] JWT dengan expiry yang wajar
- [ ] Authorization check di setiap endpoint

### Dependencies

- [ ] Regular vulnerability scanning
- [ ] Dependencies selalu up-to-date
- [ ] Lockfile (packages.lock.json) digunakan

### Logging & Monitoring

- [ ] Security events di-log
- [ ] Tidak ada sensitive data di logs
- [ ] Alerts untuk suspicious activities

## Referensi

- [OWASP Top 10](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [ASP.NET Core Security](https://learn.microsoft.com/en-us/aspnet/core/security/)
- [Microsoft Security Best Practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/best-practices-and-patterns)

---

*Terakhir diperbarui: Desember 2025*
