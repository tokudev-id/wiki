---
title: Panduan Onboarding
description: Panduan onboarding untuk pengembang .NET baru di tim Unictive, mencakup setup environment, akses sistem, struktur proyek, dan workflow pengembangan.
published: true
date: 2025-12-29T00:00:00.000Z
tags: onboarding, team, getting-started
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Panduan Onboarding Pengembang .NET

Selamat datang di tim pengembangan Unictive! Dokumen ini akan memandu Anda untuk memulai sebagai pengembang .NET di tim kami.

## Hari Pertama

### 1. Akses dan Akun

Pastikan Anda memiliki akses ke:

- [ ] Email perusahaan
- [ ] Bitbucket/GitLab repository
- [ ] Jira/Linear untuk manajemen proyek
- [ ] Slack/Teams untuk komunikasi
- [ ] VPN perusahaan (jika diperlukan)
- [ ] Azure Portal / Cloud Console
- [ ] Wiki dokumentasi (ini!)

### 2. Setup Development Environment

#### Perangkat Lunak Wajib

```powershell
# Windows - menggunakan winget
winget install Microsoft.VisualStudio.2022.Professional
winget install Microsoft.DotNet.SDK.8
winget install Git.Git
winget install Docker.DockerDesktop
winget install Microsoft.SQLServerManagementStudio
winget install Microsoft.AzureDataStudio
winget install JetBrains.Rider  # Alternatif IDE
```

#### Ekstensi Visual Studio yang Direkomendasikan

- ReSharper (opsional, berbayar)
- CodeMaid
- GitHub Copilot
- SonarLint
- REST Client

#### Ekstensi VS Code (untuk scripting/frontend)

```bash
code --install-extension ms-dotnettools.csharp
code --install-extension ms-dotnettools.csdevkit
code --install-extension ms-azuretools.vscode-docker
code --install-extension eamodio.gitlens
code --install-extension esbenp.prettier-vscode
```

### 3. Clone Repository Utama

```bash
# Clone repository
git clone https://bitbucket.org/unictive/[nama-proyek].git
cd [nama-proyek]

# Restore dependencies
dotnet restore

# Build untuk memastikan semuanya berfungsi
dotnet build

# Jalankan tests
dotnet test
```

### 4. Konfigurasi Lokal

```bash
# Salin file konfigurasi
cp appsettings.Development.example.json appsettings.Development.json

# Edit sesuai kebutuhan lokal
# - Connection string database
# - API keys
# - Service URLs
```

### 5. Setup Database Lokal

```bash
# Menggunakan Docker untuk SQL Server
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=YourStrong@Passw0rd" \
  -p 1433:1433 --name sqlserver \
  -d mcr.microsoft.com/mssql/server:2022-latest

# Jalankan migrasi
dotnet ef database update --project src/Infrastructure
```

## Minggu Pertama

### Hari 1-2: Pahami Arsitektur

1. **Baca dokumentasi ini** secara lengkap
2. **Pelajari struktur proyek**:
   ```
   src/
   ├── API/              # Controllers, Middleware
   ├── Application/      # Use Cases, Services, DTOs
   ├── Domain/           # Entities, Value Objects
   └── Infrastructure/   # Repositories, External Services
   ```
3. **Pahami alur data** dari request hingga response

### Hari 3-4: Setup dan Eksplorasi

1. **Jalankan aplikasi secara lokal**
2. **Coba endpoint API** menggunakan Swagger/Postman
3. **Baca kode** - mulai dari controller, ikuti alurnya
4. **Debug** beberapa flow untuk memahami

### Hari 5: Tugas Pertama

1. **Ambil tiket sederhana** (bug fix atau enhancement kecil)
2. **Buat branch** sesuai konvensi
3. **Implementasi** dengan mengikuti standar
4. **Buat Pull Request**

## Struktur Tim Engineering

```
Engineering Lead
├── Backend Team
│   ├── Senior Developer (.NET)
│   ├── Developer (.NET)
│   └── Junior Developer (.NET)
├── Frontend Team
│   └── Developer (React/Angular/Blazor)
├── DevOps/SRE
│   └── DevOps Engineer
└── QA Team
    └── QA Engineer
```

### Tanggung Jawab per Level

| Level | Tanggung Jawab |
|-------|----------------|
| **Junior** | Implementasi fitur sederhana, bug fix, code review dari senior |
| **Mid** | Implementasi fitur kompleks, code review, mentoring junior |
| **Senior** | Arsitektur, keputusan teknis, mentoring, code review |
| **Lead** | Roadmap teknis, koordinasi tim, stakeholder management |

## Standar Tim

### Git Workflow

```
main (production)
├── develop (staging)
│   ├── feature/TICKET-123-deskripsi-singkat
│   ├── bugfix/TICKET-456-deskripsi-singkat
│   └── hotfix/TICKET-789-deskripsi-singkat
```

### Branch Naming

```
feature/JIRA-123-tambah-fitur-login
bugfix/JIRA-456-perbaiki-validasi-email
hotfix/JIRA-789-fix-critical-bug
refactor/JIRA-012-optimasi-query
```

### Commit Message

```
feat(auth): tambah fitur login dengan OAuth2
fix(pelanggan): perbaiki validasi email duplikat
refactor(pesanan): optimasi query untuk performa
docs(readme): update instruksi instalasi
test(unit): tambah test untuk PelangganService
```

Format: `<type>(<scope>): <description>`

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

### Pull Request

1. **Judul**: `[JIRA-123] Deskripsi singkat perubahan`
2. **Deskripsi**:
   - Apa yang berubah?
   - Mengapa berubah?
   - Bagaimana cara menguji?
3. **Checklist**:
   - [ ] Unit tests ditambahkan/diperbarui
   - [ ] Tidak ada breaking changes
   - [ ] Dokumentasi diperbarui (jika perlu)
   - [ ] Self-review dilakukan

### Code Review Guidelines

**Sebagai Reviewer:**
- Review dalam 24 jam kerja
- Berikan feedback konstruktif
- Fokus pada: kebenaran, keterbacaan, performa, keamanan
- Approve jika sudah sesuai standar

**Sebagai Author:**
- Respond setiap comment
- Jangan defensive, terima masukan
- Request re-review setelah perubahan

## Pemecahan Masalah

### Langkah-langkah Problem Solving

```
1. PAHAMI masalahnya dengan jelas
   ├── Baca error message dengan teliti
   ├── Reproduksi masalah secara konsisten
   └── Identifikasi kondisi yang menyebabkan masalah

2. KUMPULKAN informasi
   ├── Cek logs (Seq, Application Insights, dll)
   ├── Debug kode
   └── Cari di Stack Overflow, GitHub Issues

3. HIPOTESIS & UJI
   ├── Buat hipotesis penyebab
   ├── Uji hipotesis dengan perubahan kecil
   └── Validasi solusi

4. IMPLEMENTASI & DOKUMENTASI
   ├── Implementasi solusi yang tepat
   ├── Tambahkan test untuk mencegah regresi
   └── Dokumentasikan solusi jika kompleks
```

### Kapan Minta Bantuan?

- Setelah **30 menit** stuck tanpa progres
- Setelah mencoba **3 pendekatan** berbeda
- Jika masalah terkait **domain bisnis** yang tidak dipahami
- Jika masalah terkait **infrastruktur/DevOps**

### Cara Minta Bantuan yang Baik

```markdown
**Masalah:**
Endpoint POST /api/pelanggan mengembalikan 500 Internal Server Error

**Yang sudah dicoba:**
1. Cek request body - sudah valid
2. Debug di PelangganController - masuk dengan benar
3. Error terjadi di PelangganService.BuatAsync()

**Error Message:**
SqlException: Cannot insert duplicate key in object 'dbo.Pelanggan'

**Hipotesis:**
Mungkin ada race condition saat validasi email?

**Pertanyaan:**
Bagaimana cara terbaik menangani concurrent insert dengan unique constraint?
```

## Komunikasi

### Channel yang Tepat

| Situasi | Channel |
|---------|---------|
| Pertanyaan teknis cepat | Slack #tech-discussion |
| Diskusi arsitektur | Meeting + dokumen |
| Bug production | Slack #alerts + Jira ticket |
| Code review | Bitbucket PR comments |
| 1-on-1 dengan lead | Meeting terjadwal |

### Meeting Rutin

- **Daily Standup**: 15 menit, setiap hari
- **Sprint Planning**: Awal sprint
- **Sprint Review**: Akhir sprint
- **Retrospective**: Akhir sprint
- **Tech Discussion**: Weekly (opsional)

## Checklist Onboarding

### Minggu 1
- [ ] Semua akses diterima
- [ ] Environment development siap
- [ ] Aplikasi berjalan lokal
- [ ] Memahami struktur proyek
- [ ] Selesaikan 1 tiket kecil

### Minggu 2
- [ ] Memahami domain bisnis
- [ ] Memahami arsitektur sistem
- [ ] Selesaikan 2-3 tiket
- [ ] Melakukan code review pertama

### Bulan 1
- [ ] Produktif dengan tiket reguler
- [ ] Memahami seluruh alur CI/CD
- [ ] Kontribusi dalam diskusi teknis
- [ ] Memiliki buddy/mentor yang jelas

## Resources

### Dokumentasi Internal
- [Standar Kode .NET](./standar-kode-dotnet.md)
- [Konvensi Penamaan](./konvensi-penamaan.md)
- [Best Practice API](./best-practice-api-service.md)
- [Testing Guidelines](./testing-guidelines.md)

### Referensi Eksternal
- [Microsoft .NET Documentation](https://docs.microsoft.com/en-us/dotnet/)
- [ASP.NET Core Documentation](https://docs.microsoft.com/en-us/aspnet/core/)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

*Terakhir diperbarui: Desember 2025*
