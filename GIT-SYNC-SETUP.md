# Panduan Setup Git Sync untuk Wiki.js

Panduan ini menjelaskan cara mengkonfigurasi Wiki.js untuk sync otomatis dengan Git repository.

## Prasyarat

1. Docker Desktop sudah berjalan
2. Repository Git sudah dibuat (GitHub/GitLab/Bitbucket)
3. Wiki.js sudah running (`docker compose up -d`)

## Langkah 1: Push Repository ke Git Hosting

### GitHub

```bash
cd C:\Work\Unictive\Wiki

# Tambahkan remote
git remote add origin https://github.com/USERNAME/wiki-unictive.git

# Push
git branch -M main
git push -u origin main
```

### Bitbucket

```bash
cd C:\Work\Unictive\Wiki

# Tambahkan remote
git remote add origin https://bitbucket.org/WORKSPACE/wiki-unictive.git

# Push
git branch -M main
git push -u origin main
```

### GitLab

```bash
cd C:\Work\Unictive\Wiki

# Tambahkan remote
git remote add origin https://gitlab.com/USERNAME/wiki-unictive.git

# Push
git branch -M main
git push -u origin main
```

## Langkah 2: Buat Access Token

### GitHub Personal Access Token

1. Buka https://github.com/settings/tokens
2. Klik **"Generate new token (classic)"**
3. Beri nama: `wiki-js-sync`
4. Pilih scope: `repo` (Full control of private repositories)
5. Klik **Generate token**
6. **Salin token** (hanya ditampilkan sekali!)

### Bitbucket App Password

1. Buka https://bitbucket.org/account/settings/app-passwords/
2. Klik **"Create app password"**
3. Beri nama: `wiki-js-sync`
4. Pilih permissions: `Repositories: Read, Write`
5. Klik **Create**
6. **Salin password**

### GitLab Personal Access Token

1. Buka https://gitlab.com/-/profile/personal_access_tokens
2. Beri nama: `wiki-js-sync`
3. Pilih scope: `read_repository`, `write_repository`
4. Klik **Create personal access token**
5. **Salin token**

## Langkah 3: Konfigurasi Wiki.js

1. Buka Wiki.js di browser: http://localhost:3000

2. Login sebagai **Administrator**

3. Buka **Administration** (ikon gear di sidebar kiri)

4. Pilih **Storage** di menu kiri

5. Klik **Git** untuk mengaktifkan

6. Isi konfigurasi:

### Konfigurasi Git Storage

| Field | Nilai |
|-------|-------|
| **Authentication Type** | `basic` |
| **Repository URI** | (lihat di bawah) |
| **Branch** | `main` |
| **Username** | Username Git Anda |
| **Password / PAT** | Token yang sudah dibuat |
| **Default Author Email** | Email Anda |
| **Default Author Name** | Nama Anda |
| **Local Repository Path** | `./data/repo` |
| **Sync Direction** | `Bi-directional` |
| **Sync Schedule** | `5 minutes` (atau sesuai kebutuhan) |

### Repository URI Format

**GitHub:**
```
https://github.com/USERNAME/wiki-unictive.git
```

**Bitbucket:**
```
https://bitbucket.org/WORKSPACE/wiki-unictive.git
```

**GitLab:**
```
https://gitlab.com/USERNAME/wiki-unictive.git
```

7. Klik **Apply** di pojok kanan atas

8. Klik **Force Sync** untuk sync pertama kali

## Langkah 4: Verifikasi

1. Kembali ke halaman utama Wiki.js
2. Cek apakah halaman-halaman sudah muncul:
   - Home (Dokumentasi Standar .NET)
   - Onboarding Guide
   - Standar Kode
   - dll.

3. Jika belum muncul, tunggu beberapa menit atau klik **Force Sync** lagi

## Troubleshooting

### Error: Authentication Failed

- Pastikan username dan token benar
- Untuk GitHub, gunakan token sebagai password, bukan password akun
- Untuk Bitbucket, gunakan App Password

### Error: Repository Not Found

- Pastikan repository sudah ada dan Anda punya akses
- Cek URL repository sudah benar

### Halaman Tidak Muncul Setelah Sync

- Pastikan file ada di folder `content/` di repository
- Pastikan file memiliki frontmatter yang valid
- Cek log di Administration > System > Logs

### Sync Tidak Berjalan Otomatis

- Pastikan **Sync Schedule** sudah diset
- Restart container: `docker compose restart wiki`

## Struktur File untuk Git Sync

Wiki.js membaca file dari root repository. Struktur yang direkomendasikan:

```
wiki-unictive/
├── home.md                        # Halaman utama (/)
├── onboarding-guide.md            # /onboarding-guide
├── standar-kode-dotnet.md         # /standar-kode-dotnet
├── konvensi-penamaan.md           # /konvensi-penamaan
├── best-practice-api-service.md   # /best-practice-api-service
├── blazor-razor-pages.md          # /blazor-razor-pages
├── repository-service-pattern.md  # /repository-service-pattern
├── architecture-ddd.md            # /architecture-ddd
├── database-caching.md            # /database-caching
├── testing-guidelines.md          # /testing-guidelines
├── error-handling-logging.md      # /error-handling-logging
├── security-awareness.md          # /security-awareness
├── resilience-patterns.md         # /resilience-patterns
├── cicd-devops.md                 # /cicd-devops
└── observability.md               # /observability
```

## Tips

1. **Bi-directional sync**: Perubahan di Wiki.js akan di-push ke Git, dan sebaliknya
2. **Backup**: Dengan Git sync, Anda punya backup otomatis di Git
3. **Collaboration**: Tim bisa edit via Wiki.js UI atau langsung edit file di Git
4. **Version control**: Semua perubahan tercatat di Git history

## Referensi

- [Wiki.js Git Storage Documentation](https://docs.requarks.io/storage/git)
- [GitHub Personal Access Tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
