# BAB 6 — Centralized Logging dengan Fluent Bit dan PostgreSQL

Laporan praktikum DevOps/DevSecOps: membangun pipeline log terpusat menggunakan **Docker Compose**, **Fluent Bit**, dan **PostgreSQL**, ditambah visualisasi dengan **Grafana** dan simulasi implementasi pada modul **PMB (Penerimaan Mahasiswa Baru)**.

**Anggota Kelompok 4**

| No | Nama | NIM |
|----|------|-----|
| 1 | Rifqi Fayyadh | 3126640030 |

## Daftar Isi

- [Gambaran Umum](#gambaran-umum)
- [Bagian 1 — Praktikum Bab 6](#bagian-1--praktikum-bab-6)
- [Bagian 2 — Tambahan: Grafana](#bagian-2--tambahan-grafana)
- [Bagian 3 — Modul PMB](#bagian-3--modul-pmb)
- [Kesimpulan](#kesimpulan)

## Gambaran Umum

Alur log yang dibangun pada praktikum ini:

```text
generator-auth ─┐
generator-api  ─┼─ Docker logging driver (fluentd, :24224) ─► Fluent Bit ─┬─► PostgreSQL (logs.fluentbit)
generator-worker┘                                                         └─► stdout (json_lines)
                                                                  │
                                                                  └─ HTTP monitoring :2020 (/api/v1/health, /api/v1/metrics)
```

| Komponen | Image | Peran |
|----------|-------|-------|
| `postgres-db` | `postgres:16-alpine` | Basis data penyimpan log |
| `fluent-bit` | `fluent/fluent-bit:4.2.8` | Pengumpul dan penerus log |
| `generator-auth` / `generator-api` / `generator-worker` | `alpine:3.20` | Generator log tiruan (auth, API, worker) |
| `grafana` | `grafana/grafana:latest` | Dashboard (bagian tambahan) |

---

# Bagian 1 — Praktikum Bab 6

## 1.1 Menyiapkan struktur folder

Buat folder `bab-6` beserta subfolder yang diperlukan (`init`, `fluent-bit`, `queries`, `scripts`).

![Membuat folder bab 6 dan subfolder](images/bab6/bab6-01.png)

## 1.2 Membuat `.env.example`

File `.env.example` berfungsi sebagai **template** kredensial (user PostgreSQL, user dan password Fluent Bit, dan sebagainya) yang aman untuk dimasukkan ke repository.

> **Fungsi command:** `nano` dipakai untuk membuat dan mengedit berkas secara manual, sedangkan `cat` dipakai untuk mengisi teks secara instan tanpa membuka editor.

![Isi file .env.example](images/bab6/bab6-02.png)

## 1.3 Membuat `.env` dari template

Template disalin menjadi `.env` (file kredensial yang benar-benar dipakai), lalu aksesnya dibatasi agar hanya owner yang dapat membaca dan menulis.

```bash
cp .env.example .env
chmod 600 .env
```

> **Fungsi command:** `cp` menyalin berkas sekaligus menentukan nama barunya, sehingga `.env.example` terduplikasi menjadi `.env`.

![Menyalin .env.example menjadi .env](images/bab6/bab6-03.png)

## 1.4 Mengamankan `.env` dengan `.gitignore`

`.env` dimasukkan ke `.gitignore` agar kredensial **tidak ikut terunggah** ke repository GitHub publik. Langkah ini memakai `nano` dan `cat` seperti sebelumnya.

![Isi file .gitignore](images/bab6/bab6-04.png)

## 1.5 Membuat `compose.yml`

`compose.yml` mendefinisikan seluruh service, volume, dan jaringan.

### a. `postgres-db`

- Basis data utama dengan image `postgres:16-alpine` (ringan dan efisien).
- Kredensial dibaca dari `.env`: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, serta `FLUENTBIT_DB_USER` dan `FLUENTBIT_DB_PASSWORD` untuk akses Fluent Bit.
- Data dipertahankan lewat volume `pg-data` di `/var/lib/postgresql/data`. Direktori lokal `./init` di-mount read-only ke `/docker-entrypoint-initdb.d` untuk inisialisasi skrip SQL.
- Terhubung ke jaringan `logging-net`.
- `healthcheck` memakai `pg_isready`: interval 5 detik, timeout 5 detik, 12 kali retry, dan `start_period` 10 detik.
- `restart: unless-stopped` agar service menyala kembali otomatis kecuali dihentikan manual.

![compose.yml — service postgres-db](images/bab6/bab6-05.png)

### b. `fluent-bit`

- Pengumpul dan penerus log (log aggregator) dengan image `fluent/fluent-bit:4.2.8`.
- `./fluent-bit/fluent-bit.conf` di-mount read-only ke `/fluent-bit/etc/fluent-bit.conf`; volume `flb-storage` di `/var/log/flb-storage` dipakai sebagai buffer sementara agar log tidak hilang.
- Port diteruskan ke localhost: `24224` (menerima log) dan `2020` (monitoring metrik internal).
- `depends_on` `postgres-db` dengan `condition: service_healthy`, sehingga Fluent Bit baru berjalan setelah PostgreSQL siap.
- Keamanan: `no-new-privileges:true` dan `restart: unless-stopped`.

![compose.yml — service fluent-bit](images/bab6/bab6-06.png)

### c. `generator-auth` dan `generator-api`

Keduanya memakai `alpine:3.20` dan menjalankan skrip shell inline yang mencetak 20 baris log JSON, satu baris per detik. Log ditangkap oleh Docker logging driver `fluentd` yang dikirim asinkron (`fluentd-async: "true"`) ke `127.0.0.1:24224` dengan tag `docker.{{.Name}}`. Keduanya menunggu `fluent-bit` berjalan terlebih dahulu.

| Service | `service` | `severity` | `event` |
|---------|-----------|------------|---------|
| `generator-auth` | `auth` | `INFO` | `login_test` |
| `generator-api` | `api` | `WARN` | `latency_test` (disertai durasi) |

![compose.yml — service generator-auth dan generator-api](images/bab6/bab6-07.png)

### d. `generator-worker`, `volumes`, dan `networks`

- `generator-worker` mensimulasikan background worker: 20 baris log (`service: worker`, `severity: ERROR`, `event: retry_test`) dengan konfigurasi logging yang sama dan hanya berjalan setelah `fluent-bit` aktif.
- `volumes`: `pg-data` (data PostgreSQL) dan `flb-storage` (buffer Fluent Bit).
- `networks`: satu jaringan kustom `logging-net` dengan driver `bridge`, sehingga semua kontainer dapat berkomunikasi lewat nama service.

![compose.yml — service generator-worker, volumes, dan networks](images/bab6/bab6-08.png)

## 1.6 Membuat `init/01-init-logging.sh`

Skrip ini membuat skema awal database saat PostgreSQL pertama kali dijalankan.

1. **Mode ketat.** Skrip diawali `#!/usr/bin/env bash` dan `set -Eeuo pipefail`: `e` berhenti saat ada perintah gagal, `u` menganggap variabel belum terdefinisi sebagai error, `o pipefail` menangkap kegagalan dalam pipeline, dan `E` menjaga trap error tetap berlaku di dalam fungsi/sub-shell.
2. **Validasi variabel.** Memeriksa `POSTGRES_USER`, `POSTGRES_DB`, `FLUENTBIT_DB_USER`, dan `FLUENTBIT_DB_PASSWORD` dengan sintaks `"${VAR:?pesan_error}"`. Jika ada yang kosong, skrip berhenti dan menampilkan pesan.
3. **Role Fluent Bit.** `psql` dijalankan dengan `ON_ERROR_STOP=1`, lalu `SELECT format(...)` dikombinasikan dengan `\gexec` untuk membuat role Fluent Bit **jika belum ada** di `pg_roles`.

![01-init-logging.sh — bagian awal skrip](images/bab6/bab6-09.png)

4. **Skema dan tabel.** Dibuat skema `logs` dan tabel `logs.fluentbit` dengan kolom `tag` (`TEXT`), `time` (`TIMESTAMP WITHOUT TIME ZONE`), dan `data` (`JSONB`), semuanya `NOT NULL`. Tiga indeks dibuat untuk mempercepat pencarian: `idx_fluentbit_time` (waktu, menurun), `idx_fluentbit_tag` (tag), dan `idx_fluentbit_data_gin` (GIN pada kolom JSONB).
5. **View `logs.recent_logs`.** Menyederhanakan pembacaan log terbaru: kolom `time`, `tag`, nama kontainer (tanda `/` dihapus), `source`, dan `log_preview` (200 karakter pertama), diurutkan `ORDER BY time DESC` dengan `LIMIT 100`.

![01-init-logging.sh — skema, tabel, indeks, dan view](images/bab6/bab6-10.png)

6. **Hak akses.** `search_path` role Fluent Bit diatur ke skema `logs` dan `public`. Role tersebut diberi `USAGE` dan `CREATE` pada skema `logs`, `SELECT` dan `INSERT` pada `logs.fluentbit`, serta `SELECT` pada view `logs.recent_logs`.

![01-init-logging.sh — pemberian hak akses](images/bab6/bab6-11.png)

## 1.7 Memberi izin eksekusi pada skrip init

```bash
chmod +x init/01-init-logging.sh
```

> **Fungsi command:** `chmod` (*change mode*) mengubah hak akses berkas, `+` menambahkan hak akses, dan `x` (*execute*) adalah hak akses agar berkas dapat dijalankan sebagai skrip.

![chmod +x pada 01-init-logging.sh](images/bab6/bab6-12.png)

## 1.8 Membuat `fluent-bit/fluent-bit.conf`

1. **`[SERVICE]`** — pengaturan global: `Flush 1`, `Daemon Off` (foreground, standar untuk kontainer), `Log_Level info`, HTTP server aktif (`HTTP_Listen 0.0.0.0`, `HTTP_Port 2020`), dan manajemen storage: `storage.path /var/log/flb-storage`, `storage.sync normal`, `storage.checksum on`, `storage.backlog.mem_limit 10M`.
2. **`[INPUT]`** — plugin `forward` pada `0.0.0.0:24224` (protokol bawaan Docker logging driver), `Mem_Buf_Limit 10M`, dan `storage.type filesystem` agar buffer disimpan di disk.
3. **`[OUTPUT]` PostgreSQL** — plugin `pgsql` dengan `Match *`, `Host postgres-db`, `Port 5432`, `User ${FLUENTBIT_DB_USER}`, `Password ${FLUENTBIT_DB_PASSWORD}`, `Database ${POSTGRES_DB}`, `Table fluentbit`, dan `storage.total_limit_size 256M`.
4. **`[OUTPUT]` stdout** — menduplikasi seluruh log ke konsol (`Match *`) dengan `Format json_lines`.

![Isi fluent-bit.conf](images/bab6/bab6-13.png)

## 1.9 Membuat `queries/analysis.sql`

File ini berisi kueri analisis log:

1. **Pengaturan sesi:** `\pset pager off` (menonaktifkan pager) dan `\timing on` (menampilkan durasi eksekusi). Kueri pertama menghitung statistik dasar: `count(*) AS total_event`, `min(time) AS event_pertama`, dan `max(time) AS event_terakhir`.
2. **Distribusi per tag:** `SELECT tag, count(*) AS total FROM logs.fluentbit GROUP BY tag ORDER BY total DESC, tag`.
3. **Metadata Docker logging driver:** mengambil `time`, `tag`, nama kontainer (`replace(data ->> 'container_name', '/', '')`), `data ->> 'source'`, dan `left(data ->> 'log', 160) AS log_preview`, diurutkan terbaru dengan `LIMIT 15`.

![analysis.sql — kueri 1 sampai 3](images/bab6/bab6-14.png)

4. **Pencarian severity `ERROR`:** menyaring `data ->> 'log' ILIKE '%"severity":"ERROR"%'` (case-insensitive), diurutkan dari yang terbaru.
5. **Ingest rate per menit:** `date_trunc('minute', time) AS minute_bucket` dengan `count(*)`, dikelompokkan, diurutkan menurun, dan dibatasi `LIMIT 10`.

![analysis.sql — kueri 4 dan 5](images/bab6/bab6-15.png)

## 1.10 Membuat `scripts/generate-logs.sh`

Skrip otomatisasi untuk menjalankan stack secara bertahap:

1. Memakai `set -Eeuo pipefail`, pindah ke folder utama proyek dengan `cd "$(dirname "$0")/.."`, lalu memeriksa keberadaan `.env`. Jika tidak ada, skrip mencetak `[FAIL] File .env tidak ditemukan.` ke stderr dan keluar dengan kode 1.
2. Menjalankan `docker compose up -d postgres-db fluent-bit`, lalu melakukan perulangan hingga 30 kali untuk memeriksa `http://127.0.0.1:2020/api/v1/health` dengan `curl`. Jika berhasil, perulangan berhenti (`break`); jika sampai percobaan ke-30 belum merespons, skrip mencetak pesan gagal dan berhenti. Jeda antarpercobaan 2 detik (`sleep 2`).
3. Setelah service utama sehat, menjalankan `docker compose up -d --force-recreate generator-auth generator-api generator-worker` agar ketiga generator dibuat ulang dari kondisi bersih, kemudian mencetak pesan hasil.

![Isi generate-logs.sh](images/bab6/bab6-16.png)

## 1.11 Membuat `scripts/test-buffering.sh`

Skrip untuk menguji buffering Fluent Bit saat database mati:

1. Memakai `set -Eeuo pipefail`, pindah ke folder utama proyek, memeriksa `.env`, lalu memuat variabel dengan `set -a`, `source .env`, `set +a`.
2. Mendefinisikan fungsi `count_rows()` yang menjalankan `docker compose exec -T postgres-db psql ... --tuples-only --no-align` untuk menjalankan `SELECT count(*) FROM logs.fluentbit;`.
3. Mengambil jumlah awal (`before`), menghentikan database dengan `docker compose stop postgres-db`, menjalankan ulang ketiga generator dengan `docker compose up -d --force-recreate`, dan menunggu 25 detik (`sleep 25`) untuk menguji apakah Fluent Bit menampung log di buffer.

![test-buffering.sh — bagian awal sampai simulasi database mati](images/bab6/bab6-17.png)

4. Menyalakan kembali database dengan `docker compose start postgres-db`, menunggu `pg_isready` sehat, lalu mengecek hingga 60 kali apakah jumlah baris (`after`) lebih besar dari `before`. Jika ya, skrip mencetak pesan sukses beserta kenaikan record lalu `exit 0`. Jika tidak, skrip mencetak pesan gagal, menyarankan `docker compose logs --tail 200 fluent-bit`, dan keluar dengan kode 1.

![test-buffering.sh — bagian pemulihan (1)](images/bab6/bab6-18.png)

![test-buffering.sh — bagian pemulihan (2)](images/bab6/bab6-19.png)

## 1.12 Membuat kedua skrip executable

```bash
chmod +x scripts/generate-logs.sh scripts/test-buffering.sh
```

![chmod +x pada skrip generate-logs dan test-buffering](images/bab6/bab6-20.png)

## 1.13 Validasi sintaks dan konfigurasi

```bash
bash -n init/01-init-logging.sh scripts/generate-logs.sh scripts/test-buffering.sh
docker compose config --quiet
```

`bash -n` (*noexec*) memeriksa kesalahan sintaks tanpa mengeksekusi skrip, sedangkan `docker compose config --quiet` memvalidasi struktur `compose.yml`.

![Validasi sintaks skrip dan compose.yml](images/bab6/bab6-21.png)

## 1.14 Menjalankan PostgreSQL dan Fluent Bit

```bash
docker compose up -d postgres-db fluent-bit
```

Perintah ini mengunduh image, membuat volume dan jaringan, lalu menjalankan `postgres-db` sampai **Healthy** dan `fluent-bit` di latar belakang.

![docker compose up -d postgres-db fluent-bit](images/bab6/bab6-22.png)

Status kontainer diperiksa dengan `docker compose ps`; kedua kontainer berstatus **Up**, artinya berhasil dijalankan.

![docker compose ps](images/bab6/bab6-23.png)

Log Fluent Bit diperiksa untuk memastikan tidak ada error:

```bash
docker compose logs --tail 100 fluent-bit
```

![Log kontainer fluent-bit](images/bab6/bab6-24.png)

## 1.15 Menjalankan generator log

```bash
./scripts/generate-logs.sh
docker compose ps
```

- Output skrip menunjukkan `postgres-db` **Healthy** dan `fluent-bit` **Running**, diikuti pengunduhan image `alpine:3.20`, pembuatan jaringan default, dan peluncuran tiga generator (`generator-worker`, `generator-auth`, `generator-api`). Skrip diakhiri pesan `[PASS] Tiga generator telah dijalankan. Tunggu sekitar 25 detik.` dan `[INFO] Pantau dengan: docker compose ps`.
- Pada `docker compose ps`, `bab-6-fluent-bit-1` dan `bab-6-postgres-db-1` berstatus **Up**, sedangkan ketiga generator berstatus **Exited (0)**. Ini normal: generator selesai mencetak 20 baris log lalu berhenti dengan kode sukses.

![docker compose ps setelah generator selesai](images/bab6/bab6-25.png)

## 1.16 Health check dan metrik Fluent Bit

```bash
curl -fsS http://127.0.0.1:2020/api/v1/health
curl -fsS http://127.0.0.1:2020/api/v1/metrics
```

Endpoint `health` mengembalikan JSON berisi versi dan fitur aktif Fluent Bit, sedangkan `metrics` menampilkan jumlah record yang masuk, diproses, dan dikirim ke PostgreSQL.

> **Fungsi command:** `curl -fsS` — `-s` menyembunyikan progress bar, `-f` menyembunyikan respons HTML error dari server, dan `-S` tetap menampilkan pesan kesalahan jika gagal.

![Output health check dan metrics Fluent Bit](images/bab6/bab6-26.png)

## 1.17 Menjalankan kueri analisis

```bash
set -a; source .env; set +a
docker compose exec -T postgres-db psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -f /dev/stdin < queries/analysis.sql
```

> **Fungsi command:** `set -a` mengekspor otomatis setiap variabel yang dimuat setelahnya, `source .env` memuat isi `.env` ke sesi terminal, dan `set +a` mematikan kembali ekspor otomatis (`-` menyalakan opsi, `+` mematikannya).
>
> `docker compose exec` menjalankan perintah di kontainer yang sedang berjalan; `-T` menonaktifkan pseudo-TTY agar aman untuk skrip; `-U` dan `-d` menentukan user dan database; `-f /dev/stdin` membuat `psql` membaca masukan dari operator `<` yang mengarah ke `queries/analysis.sql`.

![Memuat variabel dari .env](images/bab6/bab6-27.png)

![Menjalankan analysis.sql lewat psql](images/bab6/bab6-28.png)

**Hasil:** total **60 event log** dengan rentang waktu 05:47:27 sampai 05:47:46. Distribusi per tag menunjukkan `generator-api`, `generator-auth`, dan `generator-worker` masing-masing menghasilkan tepat **20 baris**. Output juga menampilkan 15 log terbaru beserta metadata, hasil pencarian severity `ERROR` dari service worker, dan ringkasan per menit yang mengonfirmasi seluruh 60 event masuk pada menit 05:47:00.

![Output analysis.sql](images/bab6/bab6-29.png)

## 1.18 Melihat view `logs.recent_logs`

Argumen `-c` menjalankan satu perintah SQL langsung tanpa file eksternal.

![Perintah SELECT dari logs.recent_logs](images/bab6/bab6-30.png)

Hasilnya 60 baris log terstruktur, berurutan dari terbaru ke terlama, dengan kolom `time`, `tag`, `container_name`, `source`, dan `log_preview` dari service worker, api, dan auth yang tersimpan di PostgreSQL.

![Isi view logs.recent_logs](images/bab6/bab6-31.png)

## 1.19 Menguji buffering (`test-buffering.sh`)

```bash
./scripts/test-buffering.sh
```

- `[INFO] Jumlah awal: 60` — baris yang sudah ada sebelum pengujian.
- Database dihentikan sementara selama 25 detik, sementara generator dan Fluent Bit tetap berjalan. Fluent Bit menampung log baru di buffer penyimpanannya tanpa kehilangan data.
- Setelah database kembali **Healthy**, Fluent Bit mengirim ulang (*flush*) data tertunda ke PostgreSQL.
- Hasil: `[PASS] Jumlah sesudah pemulihan: 120` dan `[INFO] Kenaikan record: 60`, membuktikan seluruh 60 log susulan berhasil masuk.

![Hasil test-buffering.sh](images/bab6/bab6-32.png)

---

# Bagian 2 — Tambahan: Grafana

Bagian ini adalah tambahan di luar cheatsheet dan Bab 6: menambahkan **Grafana** sebagai dashboard untuk membaca log dari PostgreSQL.

## 2.1 Menambahkan kredensial Grafana ke `.env`

Variabel baru ditambahkan: `GRAFANA_DB_USER`, `GRAFANA_DB_PASSWORD`, dan `GRAFANA_ADMIN_PASSWORD`.

![Isi .env setelah penambahan variabel Grafana](images/bab6/bab6-33.png)

## 2.2 Membuat role read-only `grafana_reader`

Role dibuat dengan hak minimum: `CONNECT` ke database, `USAGE` pada skema `logs`, serta `SELECT` pada seluruh tabel di skema tersebut (termasuk default privileges untuk tabel baru).

![Membuat role grafana_reader di PostgreSQL](images/bab6/bab6-34.png)

## 2.3 Menambahkan service Grafana di `compose.yml`

Service `grafana` memakai image `grafana/grafana:latest`, mengambil password admin dari `.env`, mengekspos port `127.0.0.1:3000`, tergabung di `logging-net`, bergantung pada `postgres-db` yang sehat, serta menambahkan volume `grafana-storage`.

![compose.yml setelah penambahan Grafana](images/bab6/bab6-35.png)

![Service grafana di compose.yml](images/bab6/bab6-36.png)

![Bagian akhir compose.yml (volumes dan networks)](images/bab6/bab6-37.png)

## 2.4 Validasi dan menjalankan Grafana

```bash
docker compose config --quiet
docker compose up -d grafana
docker compose ps
```

![Menjalankan service grafana](images/bab6/bab6-38.png)

Log Grafana diperiksa untuk memastikan migrasi database internal berjalan tanpa error:

![Log kontainer grafana](images/bab6/bab6-39.png)

## 2.5 Mengakses Grafana lewat SSH tunnel

Grafana hanya di-bind ke `127.0.0.1` di server, sehingga diakses dari komputer lokal melalui SSH port forwarding:

```bash
ssh -L 3000:127.0.0.1:3000 ubuntu-107@10.252.108.107
```

Lalu buka `http://localhost:3000` di browser.

![Halaman login Grafana](images/bab6/bab6-40.png)

![Halaman utama Grafana setelah login](images/bab6/bab6-41.png)

## 2.6 Menambahkan data source PostgreSQL

Data source PostgreSQL ditambahkan dengan menu **Connections → Data sources**, memakai kredensial `grafana_reader`. Tombol **Save & test** menampilkan *Database Connection OK*.

![Membuat data source PostgreSQL](images/bab6/bab6-42.png)

![Konfigurasi autentikasi data source](images/bab6/bab6-43.png)

![Database Connection OK](images/bab6/bab6-44.png)

## 2.7 Membuat dashboard dan panel

Panel dibuat dengan kueri SQL terhadap `logs.fluentbit` sehingga log dari Fluent Bit tampil dalam bentuk tabel.

![Editor panel dengan kueri SQL](images/bab6/bab6-45.png)

![Dashboard Grafana menampilkan log](images/bab6/bab6-46.png)

---

# Bagian 3 — Modul PMB

Simulasi implementasi Bab 6 pada aplikasi **PMB (Penerimaan Mahasiswa Baru)**, proyek `pmb-devsecops-lab`, dengan enam service wajib dan pipeline logging Fluent Bit → PostgreSQL (`log-db`).

## 3.1 Menyiapkan direktori dan mengekstrak project

Buat direktori project terlebih dahulu (`mkdir` membuat direktori), lalu ekstrak file zip PMB. Setelah diekstrak akan muncul folder `pmb-devsecops-lab`.

![Membuat direktori project](images/bab6/bab6-47.png)

![Ekstraksi file zip PMB](images/bab6/bab6-48.png)

Validasi isi folder untuk memastikan semua berkas berhasil diekstrak.

![Isi folder pmb-devsecops-lab](images/bab6/bab6-49.png)

## 3.2 Mereviu `README.md`

`README.md` berisi instruksi jelas, termasuk bagian **Quick Start** yang memuat skrip bash siap pakai dan perintah build image Docker.

![README.md](images/bab6/bab6-50.png)

![Bagian Quick Start pada README](images/bab6/bab6-51.png)

## 3.3 Menjalankan Quick Start

**Langkah 1 — `scripts/setup.sh`.** Langkah pindah direktori sudah dilakukan sebelumnya; selanjutnya jalankan `setup.sh` untuk setup awal aplikasi.

> **Catatan:** `setup.sh` pada praktik ini hanya boleh dijalankan **satu kali** agar kredensial di `.env` tidak berubah secara acak.

![Menjalankan setup.sh](images/bab6/bab6-52.png)

**Langkah 2 — `scripts/validate.sh`.** Skrip ini adalah alat pengujian otomatis (linting dan testing) lokal. Ia memeriksa sintaks semua skrip Bash dan kode Python, memastikan `compose.yaml` valid dan mendefinisikan enam service wajib, menjalankan seluruh unit test di `app/tests`, serta memverifikasi konfigurasi Docker Compose beserta `.env`.

![Menjalankan validate.sh](images/bab6/bab6-53.png)

Agar pengujian berjalan lancar, periksa bagian `networks` pada `compose.yaml`: nilai `internal` pada dua service terkait harus `false` agar tidak terisolasi total dari luar.

![Pemeriksaan network pada compose.yaml](images/bab6/bab6-54.png)

**Langkah 3 — Build dan jalankan semua service.**

```bash
docker compose up -d --build
```

`--build` membangun ulang image dari kode terkini, dan `-d` (*detached*) menjalankan seluruh service di latar belakang tanpa memblokir terminal.

![docker compose up -d --build](images/bab6/bab6-55.png)

Periksa bahwa keenam service wajib yang divalidasi `validate.sh` sudah hidup dan berjalan normal:

![Status enam service](images/bab6/bab6-56.png)

## 3.4 Health check

Perintah `curl -fsS` dikirim ke endpoint internal untuk memastikan aplikasi dan komponen logging aktif dan tidak crash:

```bash
curl -fsS http://127.0.0.1:8080/healthz
curl -fsS http://127.0.0.1:2020/api/v1/health
```

![Health check aplikasi dan Fluent Bit](images/bab6/bab6-57.png)

Lalu buka `http://127.0.0.1:8080` di browser.

![Portal PMB di browser](images/bab6/bab6-58.png)

## 3.5 Skenario pengujian

### Skenario 1 — Alur normal pendaftaran

Dijalankan otomatis dengan `./scripts/demo-normal.sh`. Alur yang diharapkan: `login_success` → `registration_submitted` → `verification_started` → `verification_completed`, sehingga status akhir pengguna menjadi **VERIFIED**.

![Hasil demo-normal.sh](images/bab6/bab6-59.png)

### Skenario 2 — Timeout dependency

Dijalankan otomatis dengan `./scripts/demo-incident.sh`. Pengguna berhasil masuk (`login_success`), mengajukan registrasi (`registration_submitted`), dan memulai verifikasi (`verification_started`). Sistem lalu mendeteksi gangguan koneksi dan melakukan percobaan ulang 3 kali (`verification_retry` ×3). Karena layanan luar tetap tidak merespons tepat waktu, terjadi `dependency_timeout` dan status akhir menjadi **FAILED / VERIFY-TIMEOUT**.

![Hasil demo-incident.sh](images/bab6/bab6-60.png)

## 3.6 Log tracing

Ambil daftar registrasi beserta `request_id` terbaru dari database aplikasi (`app-db`):

![Daftar registrasi dan request_id dari app-db](images/bab6/bab6-61.png)

Berdasarkan output tersebut dilakukan pelacakan jejak masalah (*log tracing*). Ubah nilai `investigated_request_id` di file SQL sesuai `request_id` yang diselidiki.

![Mengganti request_id pada incident-analysis.sql](images/bab6/bab6-62.png)

Perintah berikut mengeksekusi analisis dan menampilkan timeline kejadian secara terperinci untuk mengetahui penyebab kegagalan transaksi.

![Menjalankan incident-analysis.sql](images/bab6/bab6-63.png)

## 3.7 Verifikasi umum dan unit test

Verifikasi umum yang mencakup keenam service (`./scripts/verify.sh`):

![Hasil verify.sh](images/bab6/bab6-64.png)

Setup unit test khusus komponen `registration-api`:

```bash
docker compose run --rm --no-deps registration-api python -m unittest discover -s tests -v
```

![Hasil unit test registration-api (3 test, OK)](images/bab6/bab6-65.png)

## 3.8 Menguji buffering saat `log-db` mati

Tujuannya menguji ketahanan log (*buffering/backlog*) Fluent Bit ketika database log tiba-tiba mati.

![Langkah pengujian buffering](images/bab6/bab6-66.png)

1. **Catat jumlah record awal.** Pada `logs.fluentbit` tersedia **7831** baris.

   ![Jumlah record awal pada logs.fluentbit](images/bab6/bab6-67.png)

2. **Matikan database log.**

   ```bash
   docker compose stop log-db
   ```

   ![Menghentikan log-db](images/bab6/bab6-68.png)

3. **Buat pendaftaran akun baru** melalui halaman portal aplikasi.

   ![Pendaftaran baru di portal](images/bab6/bab6-69.png)

4. **Status data setelah submit** saat database log mati.

   ![Status data saat log-db mati](images/bab6/bab6-70.png)

5. **Periksa retry Fluent Bit.** Karena database log mati, Fluent Bit terus mencoba mengirim log berulang kali.

   ```bash
   docker compose logs --tail 100 fluent-bit
   ```

   ![Log retry Fluent Bit](images/bab6/bab6-71.png)

   Interpretasi log:
   - `could not translate host name "log-db" to address` adalah bukti `log-db` mati (dihentikan lewat `docker compose stop log-db`), sehingga Fluent Bit kehilangan jalur komunikasi ke database.
   - `failed to flush chunk ..., retry in X seconds` menunjukkan Fluent Bit **tidak membuang** log; data ditahan di memori/antrean (buffer) dan dijadwalkan kirim ulang secara berkala (misalnya 8, 9, atau 10 detik kemudian).
   - Potongan seperti `"request_id":"REQ-da295..."` terlihat masuk saat database mati. Data ini tersimpan aman di antrean Fluent Bit dan tidak hilang.

6. **Nyalakan kembali database log.**

   ```bash
   docker compose start log-db
   ```

   ![Menyalakan kembali log-db](images/bab6/bab6-72.png)

7. **Hitung ulang record.** Total pada `logs.fluentbit` bertambah dari **7831** menjadi **8058**. Data yang semula berada di antrean buffer kini berhasil dikirim.

   ![Jumlah record setelah log-db hidup kembali](images/bab6/bab6-73.png)

8. **Status akhir.** Data yang diinput sebelumnya kini berstatus **FAILED** (timeout), dan catatan kegagalannya tetap berhasil direkam di database log sehingga dapat dianalisis di kemudian hari.

   ![Status akhir data pada aplikasi](images/bab6/bab6-74.png)

## 3.9 Membuat laporan bukti (`reports/`)

Seluruh metrik kesehatan kontainer, riwayat transaksi anonim, statistik Fluent Bit, dokumen analisis insiden, dan nilai checksum dikonsolidasikan ke direktori `reports/` sebagai bukti pemenuhan tata kelola DevSecOps, **tanpa menyertakan `.env`**.

> **Catatan:** `incident-analysis` mengalami kegagalan pada tahap investigasi sebelumnya karena error pada identifikasi `request_id`.

![Proses pembuatan laporan reports/](images/bab6/bab6-75.png)

![Isi direktori reports/](images/bab6/bab6-76.png)

### Isi file laporan

**`registration-status.txt`**

![registration-status.txt](images/bab6/bab6-77.png)

**`compose-ps.txt`**

![compose-ps.txt](images/bab6/bab6-78.png)

**`config-sha256.txt`**

![config-sha256.txt](images/bab6/bab6-79.png)

**`fluent-bit-metrics.json`**

![fluent-bit-metrics.json](images/bab6/bab6-80.png)

**`incident-analysis.txt`**

![incident-analysis.txt](images/bab6/bab6-81.png)

Ringkasan hasil analisis:

**Total ringkasan per-kontainer**

![Ringkasan per kontainer](images/bab6/bab6-82.png)

**Tabel pencarian riwayat Request ID**

![Riwayat Request ID](images/bab6/bab6-83.png)

**Statistik tingkat keparahan error (severity log)**

![Statistik severity log](images/bab6/bab6-84.png)

**Tren distribusi log per menit (minute bucket)**

![Distribusi log per menit](images/bab6/bab6-85.png)

---

## Kesimpulan

1. Pipeline logging terpusat berhasil dibangun: log dari tiga generator dikirim lewat Docker logging driver `fluentd` ke Fluent Bit, lalu disimpan di PostgreSQL (`logs.fluentbit`) dan juga dicetak ke stdout.
2. Dari 3 generator × 20 baris diperoleh 60 event log, dan kueri analisis (total, distribusi per tag, metadata, severity `ERROR`, ingest rate per menit) berjalan sesuai harapan.
3. Mekanisme buffering Fluent Bit terbukti: saat database mati, log ditahan di buffer dan dikirim ulang setelah database pulih (60 → 120 record pada `test-buffering.sh`; 7831 → 8058 record pada modul PMB).
4. Grafana dengan role read-only `grafana_reader` berhasil terhubung ke PostgreSQL untuk menampilkan log dalam dashboard.
5. Pada modul PMB, log tracing berbasis `request_id` memungkinkan rekonstruksi alur transaksi, termasuk insiden `VERIFY-TIMEOUT`, dan laporan bukti dibuat dalam `reports/` tanpa membocorkan `.env`.
