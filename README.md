# base_project_ladanglima

Aplikasi Laravel dengan Inertia + Svelte 5 (starter kit `laravel/svelte-starter-kit`), autentikasi Fortify, dan tooling AI agent (Laravel Boost + MCP Svelte).

## Prasyarat

| Tool     | Versi minimum | Catatan                            |
| -------- | ------------- | ---------------------------------- |
| PHP      | 8.3           | Repo dan CI memakai 8.5            |
| Composer | 2             |                                    |
| Node.js  | 22            | CI memakai 22, lokal 24 juga jalan |
| npm      | 10            | Repo memakai `package-lock.json`   |
| Git      | -             |                                    |

Database default SQLite, tidak perlu server database. Ekstensi PHP `pdo_sqlite` harus aktif.

## Menjalankan pertama kali

```bash
git clone <url-repo> base_project_ladanglima
cd base_project_ladanglima
composer setup
composer run dev
```

Buka `http://localhost:8000`. Pengguna Claude Code perlu menyetujui server MCP sekali per workspace, lihat [Persetujuan MCP di Claude Code](#persetujuan-mcp-di-claude-code).

### Apa yang dilakukan `composer setup`

| #   | Perintah                                                                 | Hasil                                                                           |
| --- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| 1   | `composer install`                                                       | `vendor/`, `package:discover`                                                   |
| 2   | copy `.env.example` ke `.env` (hanya jika `.env` belum ada)              | `APP_ENV=local`, `APP_URL=http://localhost:8000`                                |
| 3   | `php artisan key:generate`                                               | `APP_KEY`                                                                       |
| 4   | `php artisan boost:install --no-interaction --guidelines --skills --mcp` | `AGENTS.md`, `.claude/skills/`, `.agents/skills/`, config MCP setiap agent      |
| 5   | `php artisan migrate --force`                                            | tabel `users`, `cache`, `jobs`; file `database/database.sqlite` dibuat otomatis |
| 6   | `npm install`                                                            | `node_modules/`                                                                 |
| 7   | `npm run build`                                                          | `public/build/`                                                                 |

Langkah 2 harus mendahului langkah 4: `boost:install` hanya tersedia ketika `APP_ENV=local` atau `APP_DEBUG=true`. Tanpa `.env`, perintah itu gagal dengan `Command "boost:install" is not defined`.

Database bersifat lokal: `database/database.sqlite` di-ignore (`database/.gitignore`). File dibuat oleh langkah 5 karena `migrate --force` memanggil `touch()` untuk SQLite yang belum ada.

### Apa yang dijalankan `composer run dev`

`php artisan dev` menyalakan tiga proses sekaligus:

| Proses | Perintah                                         |
| ------ | ------------------------------------------------ |
| server | `php artisan serve`                              |
| queue  | `php artisan queue:listen --tries=1 --timeout=0` |
| vite   | `npm run dev`                                    |

Output inline otomatis saat bukan TTY. Paksa inline dengan `php artisan dev -i`, atau lihat daftar proses dengan `php artisan dev:list`.

## Halaman

| Path                 | Keterangan                                                   |
| -------------------- | ------------------------------------------------------------ |
| `/`                  | Halaman selamat datang (`resources/js/pages/Welcome.svelte`) |
| `/dashboard`         | Butuh login dan email terverifikasi                          |
| `/settings/profile`  | Ubah profil                                                  |
| `/settings/security` | Ubah password                                                |

Autentikasi memakai Fortify: registrasi, reset password, dan verifikasi email aktif (`config/fortify.php`).

## Setelah `git pull`

Jalankan perintah sesuai file yang berubah.

| Perubahan                                      | Perintah                                                                            |
| ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `composer.json` / `composer.lock`              | `composer install`                                                                  |
| `package.json` / `package-lock.json`           | `npm install`                                                                       |
| Migration baru                                 | `php artisan migrate`                                                               |
| `boost.json`, `.ai/skills/`, `.ai/guidelines/` | `php artisan boost:install --no-interaction --guidelines --skills --mcp`            |
| Config MCP (`.mcp.json` dan sejenisnya)        | langsung terpakai; server baru di `.mcp.json` perlu disetujui sekali di Claude Code |

`composer install` tidak memperbarui artefak agent, karena `AGENTS.md` dan folder skill di-ignore dan hanya dihasilkan oleh `boost:install`. Jangan gunakan `composer update` hanya untuk keperluan itu: perintah tersebut menaikkan versi dependency dan menulis ulang `composer.lock`.

## Konfigurasi AI agent dan MCP

### File yang di-commit

| File                      | Isi                                                                    | Client                   |
| ------------------------- | ---------------------------------------------------------------------- | ------------------------ |
| `boost.json`              | sumber konfigurasi Boost: daftar agent, skill, flag `mcp`/`guidelines` | pengatur `boost:install` |
| `.mcp.json`               | MCP `laravel-boost` dan `svelte`                                       | Claude Code              |
| `opencode.json`           | MCP `laravel-boost` dan `svelte`                                       | OpenCode                 |
| `.zed/settings.json`      | MCP `laravel-boost` dan `svelte`                                       | Zed                      |
| `.agents/mcp_config.json` | MCP `laravel-boost` dan `svelte`                                       | Antigravity              |

### File yang dihasilkan (gitignored)

`AGENTS.md`, `.claude/skills/`, dan `.agents/skills/` dibuat ulang oleh `boost:install` dari `boost.json` dan `.ai/`. `CLAUDE.md` hanya ditulis jika file itu sudah ada; tanpa `CLAUDE.md`, guideline untuk Claude Code masuk ke `AGENTS.md`.

### MCP Svelte

Server lokal resmi, dijalankan lewat `npx -y @sveltejs/mcp` (paket `@sveltejs/mcp`). Tool yang tersedia: `list-sections`, `get-documentation`, `svelte-autofixer`. Referensi: <https://svelte.dev/docs/ai/mcp>.

`boost:install` melakukan merge ke config MCP yang sudah ada, bukan menimpa, sehingga entri `svelte` tetap bertahan setiap kali setup diulang.

### Persetujuan MCP di Claude Code

Server dari `.mcp.json` berstatus `Pending approval` sampai folder ini dipercaya. Jalankan `claude` di root repo, terima dialog trust workspace, lalu setujui `laravel-boost` dan `svelte`. Repo tidak bisa menyetujui servernya sendiri, jadi langkah ini berulang setiap kali `git pull` menambah server MCP baru.

### Menambah skill khusus proyek

Letakkan skill di `.ai/skills/<nama>/SKILL.md` dan commit. `boost:install --skills` menyalin atau membuat symlink ke direktori setiap agent. Jangan menaruhnya langsung di `.agents/skills/` atau `.claude/skills/`, karena keduanya di-ignore dan hilang saat clone.

Guideline khusus proyek memakai jalur yang sama: `.ai/guidelines/`.

## Perintah lain

```bash
composer run dev        # server + queue + vite
composer test           # config:clear + pint + phpstan + pest
composer lint           # perbaiki format PHP (Pint)
composer lint:check     # cek format PHP
composer types:check    # phpstan
composer ci:check       # npm run check + npm run types:check + pest

npm run build           # build frontend produksi
npm run check           # lint + format check (vp check)
npm run check:fix       # perbaiki otomatis
npm run types:check     # svelte-check

php artisan route:list --except-vendor
php artisan db:seed
```

## Troubleshooting

| Gejala                                   | Penyebab                                      | Solusi                                                                            |
| ---------------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------- |
| `Command "boost:install" is not defined` | `.env` belum ada atau `APP_ENV` bukan `local` | Pastikan `.env` ada dan `APP_ENV=local`, lalu ulangi                              |
| `Unable to locate file in Vite manifest` | Aset frontend belum dibangun                  | `npm run build`, atau jalankan `composer run dev`                                 |
| `database/database.sqlite` tidak ada     | File di-ignore, tidak ikut clone              | `php artisan migrate --force`                                                     |
| `vp check` melaporkan format pada JSON   | File JSON ditulis oleh tool, bukan oxfmt      | Sudah dikecualikan di `vite.config.ts` `fmt.ignorePatterns`; jangan format manual |
| Port 8000 dipakai proses lain            |                                               | `php artisan serve --port=8001`, lalu sesuaikan `APP_URL`                         |

## Verifikasi

```bash
composer ci:check
```

CI menjalankan `composer setup` lalu `composer ci:check` pada PHP 8.5 dan Node 22 (`.github/workflows/tests.yml`).
