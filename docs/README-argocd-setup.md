# ArgoCD App — Setup CloudNativePG & Immich (K3s)

Dokumentasi setup ArgoCD CLI, deployment CloudNativePG + Immich di cluster k3s single-node, beserta kendala yang dihadapi dan kebutuhan lingkungan.

## Arsitektur

```
Internet / Caddy (reverse proxy)
  └── https://photos.sawangan.web.id ──► NodePort 8283 ──► immich-server:2283 (ClusterIP)
                                                               │
                                                               ├── PostgreSQL (CNPG) immich-1 ──► StorageClass local-storage
                                                               └── Redis
```

Semua manifest dikelola GitOps via **ArgoCD** dari repo `github.com/mamatnurahmat/gitops` (branch `main`).

| Komponen | ArgoCD App | Namespace | Source |
|---|---|---|---|
| CNPG Operator | `cloudnative-pg` | `cnpg-system` | Helm chart `cloudnative-pg` 0.29.0 |
| Database Immich | `immich-db` | `immich` | repo path `apps/immich-db` |
| Aplikasi Immich | `immich` | `immich` | repo path `apps/immich` |

## Struktur Repo

```
argocd-app/
├── .env                  # kredensial (TIDAK di-commit)
├── .env.sops             # versi ter-enkripsi SOPS (bisa di-commit)
├── .sops.yaml            # config SOPS (age key)
├── gitops/               # clone repo mamatnurahmat/gitops
│   └── apps/
│       ├── cloudnative-pg/app.yaml     # Application: operator CNPG (Helm)
│       ├── immich-db/app.yaml          # Application: path apps/immich-db
│       ├── immich-db/cluster.yaml      # CNPG Cluster "immich" (db + user)
│       └── immich/                     # Application + namespace, configmap,
│                                       # deployment, service (NodePort), redis, pvc
└── README.md
```

## 1. Setup ArgoCD CLI

### Prasyarat
- `argocd` CLI v3.4.5+ (Homebrew: `brew install argocd`)
- `kubectl` terhubung ke cluster k3s (`kubectl config get-contexts` → context `local-lab`)
- `helm` (untuk verifikasi chart)

### Login & Token

```bash
set -a; source .env; set +a

# Penting: server TANPA scheme https:// (bug argocd CLI v3.4.5)
argocd login "argocd.sawangan.web.id" \
  --username "$ARGOCD_USER" --password "$ARGOCD_PASSWORD" --insecure

# Token tersimpan otomatis di ~/.config/argocd/config
# Ekstrak untuk dipakai sebagai ARGOCD_AUTH_TOKEN:
TOKEN=$(python3 -c "import yaml,os; d=yaml.safe_load(open(os.path.expanduser('~/.config/argocd/config'))); print([u['auth-token'] for u in d['users'] if u['name']=='argocd.sawangan.web.id'][0])")
```

### Daftarkan Repo GitOps

```bash
argocd repo add "$GITOPS_REPO" \
  --username "$GITOPS_USER" --password "$GITOPS_PAT" \
  --insecure-skip-server-verification \
  --server "argocd.sawangan.web.id" --auth-token "$ARGOCD_AUTH_TOKEN"
```

## 2. StorageClass

k3s bawaan hanya punya `local-path` (default). Karena permintaan memakai `local-storage`, dibuat StorageClass baru dengan provisioner yang sama:

```bash
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
EOF
```

**Catatan k3s**: range NodePort default k3s diubah menjadi `8000-9000` (bukan 30000-32767). Cek dengan `kubectl get svc` error saat sync.

## 3. Deploy Aplikasi

### Secret (dibuat langsung di cluster — tidak di git)

```bash
set -a; source .env; set +a
kubectl create ns immich

# Secret untuk bootstrap CNPG (initdb)
kubectl create secret generic immich-cnpg-secret \
  --namespace immich \
  --from-literal=username="$IMMICH_DB_USER" \
  --from-literal=password="$IMMICH_DB_PASSWORD"

# Secret untuk Immich app
kubectl create secret generic immich-db-credentials \
  --namespace immich \
  --from-literal=DB_HOSTNAME=immich-rw \
  --from-literal=DB_DATABASE_NAME="$IMMICH_DB_NAME" \
  --from-literal=DB_USERNAME="$IMMICH_DB_USER" \
  --from-literal=DB_PASSWORD="$IMMICH_DB_PASSWORD"
```

### ArgoCD Application

```bash
kubectl apply -f gitops/apps/cloudnative-pg/app.yaml
kubectl apply -f gitops/apps/immich-db/app.yaml
kubectl apply -f gitops/apps/immich/app.yaml
```

Urutan penting: **operator CNPG → Cluster → aplikasi Immich** (CNPG merender service `immich-rw`).

## 4. Konfigurasi Caddy

NodePort service `immich-server` di port `8283` (targetPort 2283). Di server proxy Caddy:

```
photos.sawangan.web.id {
    reverse_proxy 192.168.110.149:8283
}
```

Ganti IP sesuai node k3s (`kubectl get nodes -o wide`).

## Kendala yang Dihadapi & Solusinya

### 1. argocd CLI error "dial proxy: lookup tcp///..." saat login
- **Gejala**: `argocd login https://argocd.sawangan.web.id` gagal, endpoint tetap bisa di-curl (200).
- **Penyebab**: argocd CLI v3.4.5 tidak bisa parse server URL yang mengandung scheme `https://`.
- **Solusi**: login tanpa scheme: `argocd login "argocd.sawangan.web.id"`.

### 2. `argocd account generate-token` ditolak untuk admin
- **Gejala**: `account 'admin' does not have apiKey capability`.
- **Penyebab**: account `admin` bawaan tidak punya capability `apiKey` (sejak ArgoCD v2.9+).
- **Solusi**: pakai JWT session admin dari `~/.config/argocd/config` sebagai `ARGOCD_AUTH_TOKEN`.

### 3. CRD CNPG "metadata.annotations: Too long (262144 bytes)"
- **Gejala**: sync `cloudnative-pg` gagal; CRD `clusters` & `poolers` tidak ter-apply.
- **Penyebab**: chart CNPG 0.29.0 punya CRD sangat besar; ArgoCD menulis annotation `last-applied-configuration` yang melebihi limit 256KB.
- **Solusi**: sync option **`Replace=true`** pada Application (`kubectl patch app` atau edit `syncOptions`). CRD di-apply ulang tanpa annotation besar.

### 4. Immich CrashLoopBackOff — "permission denied to create extension"
- **Gejala**: `CREATE EXTENSION vector` gagal dengan user `UserImmich`.
- **Penyebab**: user DB owner bukan superuser; Immich butuh extension `vector`, `earthdistance` dll.
- **Solusi**: grant superuser ke `UserImmich`:
  ```sql
  ALTER ROLE "UserImmich" SUPERUSER;
  ```
  Dan tambahkan di `cluster.yaml` → `postInitApplicationSQL` (berlaku saat bootstrap).

### 5. Immich butuh Redis
- **Gejala**: `getaddrinfo ENOTFOUND redis` → CrashLoop.
- **Solusi**: deploy Redis (deployment + service) di namespace `immich`, set `REDIS_HOSTNAME=redis` dan `REDIS_PORT=6379` di configmap.

### 6. Immich "Temporarily Unavailable / maintenance mode"
- **Gejala**: domain menampilkan "Immich has been put into maintenance mode".
- **Penyebab**: entry `maintenance-mode` di tabel `system_metadata` (tersisa dari crashloop saat DB belum siap; action `select_database_restore`).
- **Solusi**: hapus dari DB lalu restart:
  ```sql
  DELETE FROM system_metadata WHERE key='maintenance-mode';
  ```
  ```bash
  kubectl rollout restart deploy/immich-server -n immich
  ```

### 7. NodePort di luar range
- **Gejala**: `nodePort: Invalid value: 32283: not in valid range`.
- **Penyebab**: k3s dikonfigurasi range `8000-9000`.
- **Solusi**: pakai `nodePort: 8283`.

### 8. Port Immich listen berubah (3001 → 2283)
- **Gejala**: service `targetPort: 3001` tidak connect; log menunjukkan listen di `2283`.
- **Penyebab**: Immich v3.1.0 default listen port `2283`.
- **Solusi**: `targetPort: 2283` (dan containerPort 2283).

### 9. DNS registry lambat di node k3s (ghcr.io / docker.io)
- **Gejala**: ImagePullBackOff, `dial tcp: lookup ghcr.io: Try again`.
- **Penyebab**: DNS node transient lambat.
- **Solusi**: bersabar / retry otomatis; image akhirnya ter-pull (CNPG butuh ~5 menit).

### 10. sops tidak menemukan age key saat decrypt
- **Gejala**: `failed to create reader for decrypting sops data key`.
- **Penyebab**: sops 3.13.2 di mesin ini tidak otomatis membaca `~/.config/sops/age/keys.txt`.
- **Solusi**: set `SOPS_AGE_KEY_FILE` saat decrypt (lihat bagian SOPS).

## Kebutuhan / Prasyarat Lingkungan

| Tool | Versi | Keterangan |
|---|---|---|
| k3s | v1.36.2+k3s1 | single-node, host `nurahmat` (192.168.110.149) |
| argocd CLI | v3.4.5 | Homebrew |
| kubectl | — | context `local-lab` |
| helm | — | untuk cek chart |
| sops | 3.13.2 | `/opt/homebrew/bin/sops` |
| age | v1.3.1 | key di `~/.config/sops/age/keys.txt` |
| Caddy | — | di server proxy (di luar k3s) |

**Port yang dibuka**: NodePort `8283` (k3s), `2283` (dalam cluster).

## SOPS — Enkripsi Kredensial

File `.env` berisi kredensial (password, PAT, token). Agar bisa di-commit/dibagikan secara aman, gunakan SOPS + age.

### Key & Konfigurasi

- Age private key: `~/.config/sops/age/keys.txt`
- Age public key (recipient): `age1amwd8g03gd794p97nceg8lqhfkea4n73dehnsux7ffkkvp3huv7sd6wmvz`
- Config `.sops.yaml` (project root):
  ```yaml
  creation_rules:
    - path_regex: \.env$
      age: age1amwd8g03gd794p97nceg8lqhfkea4n73dehnsux7ffkkvp3huv7sd6wmvz
  ```

### Enkripsi

```bash
cd /Users/mamatnurahmat/home-lab/argocd-app
sops --encrypt .env > .env.sops
```

`/usr/local/bin/sops` rusak (ELF x86-64 tidak jalan di Apple Silicon) — pakai `/opt/homebrew/bin/sops`.

### Dekripsi (untuk memakai nilai)

```bash
cd /Users/mamatnurahmat/home-lab/argocd-app
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
  sops --decrypt --input-type dotenv --output-type dotenv .env.sops > .env

# lalu source seperti biasa
set -a; source .env; set +a
```

**Penting**:
- `.env` **tidak boleh di-commit** (sudah ada di `.gitignore` repo gitops).
- `.env.sops` **aman untuk di-commit** — isi terenkripsi.
- `SOPS_AGE_KEY_FILE` wajib di-set saat decrypt di mesin ini.

## Verifikasi

```bash
# Status ArgoCD
kubectl get applications -n argocd
#   cloudnative-pg, immich, immich-db → Synced / Healthy

# Pod
kubectl get pods -n cnpg-system    # operator Running
kubectl get pods -n immich         # immich-1, immich-server, redis Running

# Cluster CNPG
kubectl get cluster -n immich immich   # "Cluster in healthy state"

# API Immich
curl http://192.168.110.149:8283/api/server/ping   # {"res":"pong"}
curl https://photos.sawangan.web.id/api/server/ping # {"res":"pong"}

# Koneksi DB
kubectl exec -n immich immich-1 -- bash -lc \
  'PGPASSWORD="p@55immich" psql -h immich-rw -U UserImmich -d immich -tAc "SELECT 1;"'
```
