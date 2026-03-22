# Dokumentasi Instalasi dan Eksekusi Proyek
## PTSN DevOps Tech Assessment – Cloud-Native On-Premise Ecosystem

**Kandidat:** Fathur Wiriansyah  
**Aplikasi demo:** BooksLib  
**Basis dokumentasi:** rekaman demo yang diberikan, repo aplikasi, dan repo GitOps/infrastruktur.

---

## 1. Ringkasan Proyek

Proyek ini adalah implementasi MVP ekosistem DevOps berbasis *cloud-native on-premise* untuk mendemonstrasikan integrasi beberapa komponen utama, yaitu:

- **Rancher** sebagai pusat manajemen cluster Kubernetes.
- **RKE2** sebagai distribusi Kubernetes untuk workload cluster.
- **Jenkins** sebagai CI untuk build, scan, push image, dan update manifest GitOps.
- **Argo CD** sebagai *pull-based deployment engine* untuk GitOps.
- **HashiCorp Vault** sebagai pusat manajemen secret.
- **External Secrets Operator (ESO)** untuk menarik secret dari Vault ke Kubernetes.
- **Keycloak** sebagai *Identity Provider* dan fondasi SSO/OIDC.
- **BooksLib** sebagai aplikasi mikroservis yang menjadi workload demo.


---

## 2. Tujuan Dokumentasi

Dokumen ini dibuat untuk memenuhi kebutuhan:

1. Menjelaskan langkah instalasi dan eksekusi proyek.
2. Menjelaskan alur CI/CD dan deployment GitOps yang digunakan pada demo.
3. Menjelaskan kompromi arsitektur yang diambil selama implementasi demo karena keterbatasan perangkat keras.
4. Menyediakan alur verifikasi hasil deployment agar mudah direproduksi.

---

## 3. Topologi Lingkungan Demo

### 3.1 Node yang digunakan

| Node | Peran | Spesifikasi |
|---|---|---|
| `mgmt-01` | Management node | 6 GB RAM / 80 GB storage / 4 vCPU |
| `cluster-01` | Workload cluster | 6 GB RAM / 100 GB storage / 4 vCPU |

### 3.2 Peran masing-masing node

#### `mgmt-01`
Node ini berfungsi sebagai pusat kontrol ekosistem demo, yang berisi:

- Keycloak
- Vault
- Jenkins
- Rancher
- akses ke GitHub repository

#### `cluster-01`
Node ini menjalankan cluster RKE2 dan menjadi lokasi workload aplikasi, termasuk:

- Argo CD
- External Secrets Operator
- PostgreSQL
- Seluruh workload BooksLib
- Horizontal Pod Autoscaler (HPA)

### 3.3 Akses layanan pada demo


| Layanan | Akses |
|---|---|
| Keycloak | `http://192.168.56.10:8081` |
| Vault | `http://192.168.56.10:8200` |
| BooksLib Web UI | `http://192.168.56.20:30080` |
| Auth API | `http://192.168.56.20:30081` |
| Books API | `http://192.168.56.20:30082` |
| Reviews API | `http://192.168.56.20:30083` |

> Pada demo ini akses memakai **IP langsung** dan **NodePort** untuk menyederhanakan lab lokal dan menghindari ketergantungan pada DNS/hostname.

---

## 4. Arsitektur Solusi

### 4.1 Arsitektur aplikasi

Aplikasi **BooksLib** dipisah menjadi mikroservis independen agar tiap komponen dapat dibangun dan dideploy secara terpisah. Frontend berkomunikasi ke tiga backend service melalui endpoint API yang di-*inject* saat proses build Jenkins.

### 4.2 Arsitektur deployment

Pipeline bekerja dengan pola berikut:

1. Jenkins mengambil source code aplikasi dari repo **`sncyber-ops/bookslib`**.
2. Jenkins membangun image untuk empat service.
3. Jenkins menjalankan *vulnerability scan* menggunakan **Trivy**.
4. Jenkins melakukan push image ke Docker Hub.
5. Jenkins melakukan update tag image pada repo GitOps **`Wrianzz/sn-devops-test`**.
6. Argo CD membaca perubahan tersebut dan melakukan sinkronisasi otomatis ke cluster.
7. Kubernetes menjalankan rollout baru menggunakan strategi **RollingUpdate**.

### 4.3 Arsitektur secret management

- Secret database tidak ditulis langsung dalam manifest deployment.
- Secret disimpan di **Vault** pada path `secret/bookslib/postgres`.
- **External Secrets Operator** mengambil secret tersebut dan membuat `postgres-secret` di Kubernetes.
- Service PostgreSQL, Auth Service, Books Service, dan Reviews Service membaca secret dari Kubernetes Secret hasil sinkronisasi ESO.

---

## 5. Repository yang Digunakan

### 5.1 Repository aplikasi

Repo aplikasi:

```text
https://github.com/sncyber-ops/bookslib.git
```

Struktur utamanya:

```text
bookslib/
├── auth-service/
├── books-service/
├── reviews-service/
├── frontend/
├── init.sql
├── docker-compose.yml
└── .env
```

### 5.2 Repository GitOps / infrastruktur demo

Repo GitOps:

```text
https://github.com/Wrianzz/sn-devops-test.git
```

Struktur utama:

```text
sn-devops-test/
├── k8s/
│   ├── auth-deployment.yaml
│   ├── books-deployment.yaml
│   ├── frontend-deployment.yaml
│   ├── hpa.yaml
│   ├── postgres.yaml
│   ├── reviews-deployment.yaml
│   └── vault-eso.yaml
├── Jenkinsfile
├── README.md
└── keycloak-compose.yaml
```

## 6. Langkah Instalasi dan Eksekusi

Bagian ini adalah alur instalasi dan eksekusi

### Langkah 1 — Clone repository GitOps / infrastruktur

```bash
git clone https://github.com/Wrianzz/sn-devops-test.git
cd sn-devops-test
```

### Langkah 2 — Jalankan Keycloak + PostgreSQL pada management node

File `keycloak-compose.yaml` menunjukkan bahwa Keycloak dan PostgreSQL dijalankan dengan Docker Compose pada host management.

```bash
docker compose -f keycloak-compose.yaml up -d
```

### Langkah 3 — Verifikasi Keycloak

Buka:

```text
http://192.168.56.10:8081
```

Login admin default dari compose:

- **Username:** `admin`
- **Password:** `admin123`

### Langkah 4 — Buat / verifikasi client OIDC di Keycloak

Buat client:

- `argocd`
- `jenkins`
- `rancher`
- `vault`

OIDC dipakai untuk sentralisasi autentikasi antarkomponen.

### Langkah 5 — Siapkan secret database di Vault

Manifest `vault-eso.yaml` mengharuskan ESO membaca secret dari path:

```text
secret/bookslib/postgres
```

Isi minimal secret harus menyuplai pasangan key yang dibutuhkan deployment:

- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`



### Langkah 6 — Buat token akses Vault untuk ESO

Karena `SecretStore` membaca `tokenSecretRef` bernama `vault-token`, maka di cluster perlu dibuat secret.




## 6.2 Menjalankan alur CI/CD Jenkins → GitOps → Argo CD

### Langkah 1 — Buat pipeline Jenkins dari file `Jenkinsfile`

Repo `sn-devops-test` sudah menyediakan pipeline deklaratif Jenkins.

Credential minimal yang perlu dibuat di Jenkins:

- `dockerhub-cred`
- `github-cred`

### Langkah 2 — Jalankan pipeline

Pipeline akan melakukan tahapan berikut:

1. Checkout source code dari repo aplikasi.
2. Rewrite `frontend/.env` agar mengarah ke IP cluster:
   - `VITE_AUTH_API=http://192.168.56.20:30081`
   - `VITE_BOOKS_API=http://192.168.56.20:30082`
   - `VITE_REVIEWS_API=http://192.168.56.20:30083`
3. Build image:
   - `wrianzz/bookslib-frontend:<BUILD_NUMBER>`
   - `wrianzz/bookslib-auth-service:<BUILD_NUMBER>`
   - `wrianzz/bookslib-books-service:<BUILD_NUMBER>`
   - `wrianzz/bookslib-reviews-service:<BUILD_NUMBER>`
4. Scan image dengan Trivy.
5. Push image ke Docker Hub.
6. Update tag image pada manifest GitOps.
7. Commit & push perubahan ke branch `main` repo GitOps.

### Langkah 3 — Pastikan Argo CD memonitor repo GitOps

Pastikan aplikasi Argo CD bernama **`bookslib`** berada dalam status:

- **Health:** Healthy
- **Sync Status:** Synced

## 7. Verifikasi Hasil Demo

Setelah deployment sukses, lakukan verifikasi berikut:

### 7.1 Verifikasi di Rancher

- Semua deployment berada pada status **Active**.
- Replica service utama berjalan sesuai target.
- Pod PostgreSQL tersedia.
- Namespace dan workload terlihat sehat.

### 7.2 Verifikasi di Argo CD

- Application `bookslib` berstatus **Healthy**.
- Sync status **Synced**.
- Resource tree menunjukkan frontend, auth, books, reviews, postgres, HPA, dan resource ESO.

### 7.3 Verifikasi di Vault

- Secret `bookslib/postgres` tersedia.
- Path API terbaca dari mount `secret` KV v2.

### 7.4 Verifikasi di aplikasi BooksLib

- Halaman web dapat diakses.
- Pengguna dapat menambah buku.
- Buku muncul di katalog.
- Pengguna dapat menghapus buku.
- Pengguna dapat menambahkan ulasan dan rating.

---
## 8. Kompromi Arsitektur pada Demo

Karena demo dijalankan pada VM lokal dengan resource terbatas, beberapa kompromi arsitektur dilakukan agar sistem tetap bisa berjalan stabil dan tetap merepresentasikan desain intinya.

### 8.1 Cluster HA disederhanakan menjadi single-node RKE2

**Ideal production design:**

- 3 control-plane node
- 3 worker node
- quorum etcd yang sehat
- toleransi kegagalan node yang lebih baik

**Implementasi demo:**

- hanya 1 node RKE2 aktif untuk workload

**Alasan kompromi:**

- keterbatasan RAM dan CPU jika seluruh komponen HA dijalankan sekaligus di laptop/VM lokal

**Dampak:**

- tidak ada failover control-plane
- bukan representasi penuh high-availability production
- tetapi cukup untuk mendemonstrasikan GitOps, autoscaling, secret management, dan integrasi OIDC

### 8.2 Database HA tidak memakai Patroni pada eksekusi demo

**Ideal production design:**

- PostgreSQL HA berbasis Patroni
- replikasi dan failover database

**Implementasi demo:**

- PostgreSQL single replica pod

**Alasan kompromi:**

- komponen HA database cukup berat untuk resource lab kecil

**Dampak:**

- tidak ada failover database otomatis
- tetap cukup untuk menunjukkan integrasi aplikasi dan alur secret injection dari Vault

### 8.3 PAM / JumpServer tidak dijalankan secara penuh

**Ideal production design:**

- akses administratif melalui PAM/Bastion
- MFA
- audit session recording

**Implementasi demo:**

- PAM fisik tidak diaktifkan sebagai komponen terpisah

**Alasan kompromi:**

- beban RAM dan kompleksitas lab meningkat tajam bila seluruh komponen keamanan enterprise dijalankan bersamaan

**Mitigasi pada demo:**

- kontrol akses tetap didorong ke layer identitas melalui Keycloak OIDC
- pendekatan *identity-based access control* tetap dipertahankan untuk komponen utama

### 8.4 Akses aplikasi memakai IP langsung dan NodePort

**Ideal production design:**

- akses melalui load balancer / ingress / DNS internal
- terminasi TLS yang lebih konsisten

**Implementasi demo:**

- frontend dan API diekspos menggunakan NodePort dan IP langsung

**Alasan kompromi:**

- menyederhanakan pengujian lab lokal
- mengurangi kebutuhan setup DNS, ingress controller, dan sertifikat tambahan

**Dampak:**

- akses lebih sederhana untuk demonstrasi
- tetapi tidak merepresentasikan jalur akses enterprise yang sepenuhnya production-grade

### 8.5 Beberapa komponen management dicentralisasi pada satu node

**Ideal production design:**

- pemisahan lebih tegas antara management plane dan komponen pendukung

**Implementasi demo:**

- beberapa layanan management seperti Keycloak, Jenkins, Vault, dan akses Rancher dipusatkan pada node management

**Alasan kompromi:**

- efisiensi resource dan kemudahan observasi saat presentasi/demo

**Dampak:**

- jejak resource lebih hemat
- isolasi antar-komponen tidak sekuat implementasi enterprise penuh

---


## 9. Nilai Teknis 

Walaupun dilakukan beberapa kompromi, demo ini tetap berhasil menunjukkan poin-poin penting berikut:

1. **CI/CD modern** dengan pemisahan jelas antara build dan deploy.
2. **GitOps** karena desired state deployment diletakkan pada repo dan direkonsiliasi Argo CD.
3. **Shift-left security** melalui Trivy scan sebelum image didorong ke registry.
4. **Centralized authentication** melalui Keycloak OIDC.
5. **Secret management yang lebih aman** karena kredensial tidak ditaruh di manifest secara plaintext.
6. **Resource-aware deployment** melalui readiness probe dan HPA.
7. **Zero-downtime oriented rollout** dengan strategi RollingUpdate pada frontend.

---


## 10. Kesimpulan

MVP ini berhasil membangun demonstrasi ekosistem DevOps yang cukup lengkap dalam skala lab lokal. Nilai utamanya bukan pada kemewahan jumlah node, melainkan pada integrasi antarkomponen yang tetap terjaga:

- identitas terpusat,
- secret terpusat,
- CI/CD modern,
- deployment GitOps,
- dan aplikasi mikroservis yang benar-benar berjalan.

Dengan keterbatasan resource, keputusan penyederhanaan yang diambil masih rasional dan tetap menjaga esensi desain arsitektur yang diminta.

