# base_project_ladanglima

Aplikasi Laravel dengan Inertia + Svelte 5 (starter kit `laravel/svelte-starter-kit`), autentikasi Fortify, dan tooling AI agent (Laravel Boost + MCP Svelte).

## Prasyarat

| Tool     | Versi minimum | Catatan                                                |
| -------- | ------------- | ------------------------------------------------------ |
| PHP      | 8.3           | Repo dan CI memakai 8.5                                |
| Composer | 2             |                                                        |
| Node.js  | 22            | CI memakai 22, lokal 24 juga jalan; hanya jalur native |
| npm      | 10            | Repo memakai `package-lock.json`; hanya jalur native   |
| Git      | -             |                                                        |
| Docker   | -             | Opsional; Docker Desktop (WSL2 untuk jalur `sail`)     |

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

## Menjalankan dengan Docker (Sail)

Stack Docker memakai `compose.yaml` di root: satu service `laravel.test` (image Sail PHP 8.5, Node 24), SQLite dari bind mount, tanpa container database. File ini sudah disesuaikan untuk repo ini; jangan jalankan `php artisan sail:install` lagi karena perintah itu menimpa `compose.yaml` dan mengubah `phpunit.xml`.

| Bagian              | Nilai                                                                               |
| ------------------- | ----------------------------------------------------------------------------------- |
| Port aplikasi       | `APP_PORT` (compose default 80)                                                     |
| Port Vite           | `VITE_PORT` (compose default 5173)                                                  |
| `APP_URL` container | `http://localhost:${APP_PORT}` lewat `environment`, tanpa mengubah `APP_URL` native |
| `node_modules`      | Named volume `node_modules`, terpisah dari `node_modules` host                      |
| Web server          | `artisan serve --no-reload` agar `APP_URL` dari compose dipakai web SAPI            |
| Xdebug              | Off secara default (`SAIL_XDEBUG_MODE=off`)                                         |

Prasyarat: Docker Desktop. Jalur `sail` memerlukan WSL2 (Ubuntu) atau Linux/macOS; tanpa WSL gunakan `docker compose` seperti di bawah. PHP dan Composer host diperlukan untuk `composer install`, karena build context Sail berada di `./vendor/laravel/sail/runtimes/8.5`.

### Setup pertama

Jalur utama memakai Sail (WSL2 Ubuntu, Linux, atau macOS). Di Windows, jalankan baris `cp`, `composer install`, `key:generate`, dan `boost:install` di PowerShell; perintah `sail` di Ubuntu:

```bash
cp .env.example .env          # jika belum ada
composer install              # menghasilkan vendor/ untuk build context
php artisan key:generate      # jika APP_KEY masih kosong
php artisan boost:install --no-interaction --guidelines --skills   # artefak agent (shell host)
./vendor/bin/sail up -d

# sekali per volume baru; dijalankan sebagai root
./vendor/bin/sail root-shell -c "chown sail:sail /var/www/html/node_modules"

./vendor/bin/sail npm ci
./vendor/bin/sail npm run build
./vendor/bin/sail artisan migrate --force
```

Buka `http://localhost` (default `APP_PORT=80`; di mesin dev ini `APP_PORT=8080`, jadi `http://localhost:8080`).

`key:generate` harus dijalankan sebelum `up` (atau diikuti `./vendor/bin/sail restart`), karena web server memakai `--no-reload` dan mempertahankan nilai env saat boot.

`boost:install` dijalankan di shell host tempat MCP client berjalan, bukan di container: di container `LARAVEL_SAIL=1` mengaktifkan integrasi Sail sehingga `boost.json` dan config MCP yang di-commit ditulis ulang memakai launcher `vendor/bin/sail`. Perintah di atas sengaja tanpa `--mcp` karena config MCP yang di-commit sudah benar; tambahkan `--mcp` hanya dari shell Windows bila ingin meregenerasinya, karena dari WSL flag itu menulis launcher `wsl.exe` ke config tersebut.

`chown` wajib setelah `up` pertama dan setiap `docker compose down -v`, karena volume baru dibuat milik root. Pakai `npm ci`, bukan `npm install`, agar `package-lock.json` tidak ditulis ulang di container.

Web server container memakai `artisan serve --no-reload`, jadi perubahan nilai di `.env` baru berlaku setelah `./vendor/bin/sail restart` (atau `docker compose restart`).

#### Tanpa WSL (Windows PowerShell)

Langkah host di atas (`key:generate`, `boost:install`) tetap sama. Setelah itu gunakan `docker compose` langsung; `sail` adalah skrip bash dan tidak jalan tanpa WSL. Perintahnya setara dengan `sail`: `docker compose exec -u sail laravel.test <cmd>` adalah padanan `sail <cmd>`.

```bash
docker compose up -d
docker compose exec -T laravel.test chown sail:sail /var/www/html/node_modules
docker compose exec -T -u sail laravel.test npm ci
docker compose exec -T -u sail laravel.test npm run build
docker compose exec -T -u sail laravel.test php artisan migrate --force
```

### Development

```bash
./vendor/bin/sail npm run dev                                   # Vite
./vendor/bin/sail artisan queue:listen --tries=1 --timeout=0     # queue, terminal lain
./vendor/bin/sail test                                          # test (php artisan test)
```

Tanpa opsi `-T` untuk perintah interaktif, agar Ctrl+C menghentikan Vite dan menghapus `public/hot`. Dari PowerShell, padanannya:

```bash
docker compose exec -u sail laravel.test npm run dev
```

Secara default container memakai inotify. Jika file repo berada di filesystem Windows (`C:\` atau `/mnt/c`), set `CHOKIDAR_USEPOLLING=true` di `.env`, lalu jalankan `./vendor/bin/sail up -d` (tanpa WSL: `docker compose up -d`) agar container dibuat ulang (variabel ini diteruskan lewat `environment`). Interval polling diatur `CHOKIDAR_INTERVAL` (default 500 ms). Untuk performa terbaik, simpan repo di filesystem WSL agar inotify bekerja tanpa polling.

### Kinerja

| Kondisi                                  | CPU                    | RAM container |
| ---------------------------------------- | ---------------------- | ------------- |
| Idle (`up -d`, tanpa Vite)               | ~0%                    | ~75–80 MiB    |
| `npm run dev` steady state               | ~10%                   | ~0,95 GiB     |
| Start pertama Vite (optimasi dependency) | spike ~100%, < 1 menit | -             |

Yang menjaga tetap ringan: tanpa container database, Xdebug off, `node_modules` di named volume, dan polling hanya aktif bila `CHOKIDAR_USEPOLLING=true`. Folder `vendor/`, `storage/`, `bootstrap/cache/`, dan `public/build/` di-ignore dari watcher. Setelah selesai bekerja, `./vendor/bin/sail stop` (tanpa WSL: `docker compose stop`) menghentikan container dan membebaskan RAM; service tidak punya restart policy sehingga tetap berhenti. `PHP_CLI_SERVER_WORKERS` biarkan default (satu proses); menaikkannya hanya berguna jika banyak request paralel, dan hasil ukur 1/2/4 worker hanya berbeda ~1–3 MiB.

### Filesystem WSL

Menyimpan repo di filesystem WSL (misalnya `~/projects/base_project_ladanglima`) memberi I/O lebih cepat dan tidak memerlukan polling. Saat memindahkan:

- `.env` dan `database/database.sqlite` tidak ikut `git clone` (gitignored); salin atau jalankan `php artisan migrate --force`.
- `compose.yaml` harus ada di salinan (file ini bagian dari repo).
- `vendor/` harus dibuat di sana lewat `composer install` (butuh PHP + Composer di Ubuntu) atau disalin dari checkout lama.
- `npm ci` dijalankan lewat Sail, bukan di host.
- Jika checkout lama di `C:\` masih dipakai, hentikan dulu atau set `COMPOSE_PROJECT_NAME` berbeda agar container dan volume tidak saling menimpa.

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

| Perubahan                                      | Perintah                                                                                                           |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `composer.json` / `composer.lock`              | `composer install`                                                                                                 |
| `package.json` / `package-lock.json`           | `npm install`                                                                                                      |
| Migration baru                                 | `php artisan migrate`                                                                                              |
| Migration baru (Docker)                        | `./vendor/bin/sail artisan migrate` (tanpa WSL: `docker compose exec -T -u sail laravel.test php artisan migrate`) |
| `boost.json`, `.ai/skills/`, `.ai/guidelines/` | `php artisan boost:install --no-interaction --guidelines --skills --mcp`                                           |
| Config MCP (`.mcp.json` dan sejenisnya)        | langsung terpakai; server baru di `.mcp.json` perlu disetujui sekali di Claude Code                                |
| `compose.yaml`                                 | `./vendor/bin/sail up -d` (tanpa WSL: `docker compose up -d`)                                                      |
| `package.json` / `package-lock.json` (Docker)  | `./vendor/bin/sail npm ci` (tanpa WSL: `docker compose exec -T -u sail laravel.test npm ci`)                       |

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

Untuk Docker, jalankan perintah `php`, `composer`, `npm`, dan `artisan` di atas dengan prefiks `./vendor/bin/sail` (misalnya `./vendor/bin/sail artisan db:seed`). Pengecualian: `composer run dev` tidak dipakai di container karena menyalakan `artisan serve` di port 8000 yang tidak di-publish; gunakan `sail npm run dev` + `sail artisan queue:listen` seperti di bagian Development.

## Troubleshooting

| Gejala                                   | Penyebab                                       | Solusi                                                                                                                                                                       |
| ---------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Command "boost:install" is not defined` | `.env` belum ada atau `APP_ENV` bukan `local`  | Pastikan `.env` ada dan `APP_ENV=local`, lalu ulangi                                                                                                                         |
| `Unable to locate file in Vite manifest` | Aset frontend belum dibangun                   | `npm run build`, atau jalankan `composer run dev`                                                                                                                            |
| `database/database.sqlite` tidak ada     | File di-ignore, tidak ikut clone               | `php artisan migrate --force`                                                                                                                                                |
| `vp check` melaporkan format pada JSON   | File JSON ditulis oleh tool, bukan oxfmt       | Sudah dikecualikan di `vite.config.ts` `fmt.ignorePatterns`; jangan format manual                                                                                            |
| Port 8000 dipakai proses lain            |                                                | `php artisan serve --port=8001`, lalu sesuaikan `APP_URL`                                                                                                                    |
| Docker: `npm ci` gagal `EACCES`          | Volume `node_modules` baru milik root          | `./vendor/bin/sail root-shell -c "chown sail:sail /var/www/html/node_modules"` (tanpa WSL: `docker compose exec -T laravel.test chown sail:sail /var/www/html/node_modules`) |
| Docker: port 80 atau 5173 dipakai        | Proyek lain berjalan di port tersebut          | Set `APP_PORT` dan `VITE_PORT` di `.env`; `APP_URL` container mengikuti `APP_PORT`                                                                                           |
| Docker: HMR tidak bereaksi saat edit     | File di `C:\`/`/mnt/c` tanpa polling           | Set `CHOKIDAR_USEPOLLING=true` di `.env`, lalu `./vendor/bin/sail up -d` (tanpa WSL: `docker compose up -d`)                                                                 |
| Docker: halaman memakai dev server mati  | `public/hot` tertinggal setelah proses di-kill | Hapus `public/hot`, lalu `npm run build` atau jalankan `npm run dev` lagi                                                                                                    |

## Verifikasi

```bash
composer ci:check
```

CI menjalankan `composer setup` lalu `composer ci:check` pada PHP 8.5 dan Node 22 (`.github/workflows/tests.yml`).
