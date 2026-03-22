# PTSN DevOps Tech Assessment - Cloud-Native On-Premise Ecosystem

**Kandidat:** Fathur Wiriansyah\
**Video Demonstrasi:** https://youtu.be/Yv3lpNgRLWE

---

## 1. Analisa Sistem Desain (Berdasarkan Diagram Soal)
Pada diagram infrastruktur, user dari company network mengakses aplikasi melalui sebuah VIP yang dijaga oleh dua load balancer dalam mode active-passive (Keepalived). Di belakangnya ada dua domain utama: Rancher/Kubernetes management cluster di sisi kiri dan production workload cluster di sisi kanan. Selain itu ada Vault cluster untuk manajemen secrets, HA PostgreSQL berbasis Patroni, dan S3 storage sebagai target backup. Diagram juga memperlihatkan bahwa user masih dapat melakukan direct SSH ke server. Dari sudut pandang alur, rancangan ini berusaha membangun pusat kendali cluster, menjalankan workload production secara terpisah, mengelola secret secara terpusat, dan menyimpan backup ke storage terpisah. Namun, terdapat beberapa area krusial yang perlu ditingkatkan untuk memenuhi standar keamanan dan efisiensi Enterprise:
* **Management Cluster Memakai 2 Master Node:** Pada diagram Rancher management cluster terlihat hanya ada RKE2 master01 dan master02. Untuk control-plane berbasis etcd, jumlah server ideal harus ganjil. Sepemahaman saya, minimal haru ada 3 server nodes. Dua master bukan desain HA yang sehat, karena ketika satu node gagal, quorum bisa hilang atau cluster masuk kondisi tidak stabil. Rekomendasi saya management cluster jangan menggunakan 2 master. Ubah menjadi 3 master.
* **Direct SSH access:** Ini bukan best practice untuk infrastruktur modern. Akses administratif seharusnya melalui PAM, MFA, session recording, dan audit trail yang kuat. Rekomendasi saya, ganti dengan PAM, MFA, , session logging, dan least privilege. SSH key management juga harus terpusat dan diaudit.

Pada diagram CI/CD, alur dimulai dari Gitflow: feature/, hotfix/, develop, release/, dan main/master. Push atau pull request memicu pipeline CI untuk menjalankan unit test, build image, lalu push image ke registry. Setelah itu pipeline CD mendorong deployment ke Dev, lalu jika memenuhi syarat branch tertentu dilanjutkan ke Staging, Integration Test, UAT, dan terakhir Production. Secara konseptual ini adalah model branch-based promotion pipeline. Model ini masih valid untuk banyak organisasi, tetapi belum identik dengan GitOps modern, karena GitOps mensyaratkan desired state yang deklaratif, versioned, dipull otomatis, dan terus direkonsiliasi oleh agent di cluster.
* **Kurangnya Shift-Left Security:** Pipeline CI langsung melakukan *Build Image* dan *Push* ke Registry setelah Unit Test. Tidak ada pemindaian kerentanan (CVE). Hal ini berisiko meloloskan *image* yang rentan ke fase *production*. Rekomendasi saya bisa tambahkan SAST, SCA, Secret Scanning, dan Image Scanning.
* **Pendekatan CD (Push vs Pull):** Diagram menunjukkan Jenkins melakukan *push deployment* langsung ke kluster. Dalam GitOps, desired state seharusnya berada di Git secara deklaratif, lalu ditarik dan direkonsiliasi terus-menerus oleh agent seperti Argo CD. Saran saya deployment ke cluster lebih baik dilakukan oleh Argo CD ketimbang full jenkins.

---

## 2. Arsitektur MVP yang Dibangun

| Komponen | Spesifikasi | 
| --- | --- |
| **Server mgmt-01** | 6GB RAM / 80GB Storage / 4 CPUs | 
| **Server cluster-01** | 6GB RAM / 100GB Storage / 4 CPUs |

Untuk menjawab kebutuhan ekosistem *cloud-native* terpusat, MVP ini dibangun dengan *stack* teknologi berikut:
* **Manajemen Kluster:** Rancher (mengelola kluster RKE2).
* **GitOps CI/CD:** Jenkins (CI & Image Build) dikombinasikan dengan ArgoCD (CD) untuk *pull-based deployment* dan *Zero Downtime* via strategi `RollingUpdate`.
* **Autoscaling:** K8s Horizontal Pod Autoscaler (HPA) dan Readiness Probes diimplementasikan pada setiap layanan aplikasi.
* **Secret Management:** HashiCorp Vault. Kredensial database diinject ke Kubernetes menggunakan External Secrets Operator (ESO) tanpa meninggalkan *plaintext* di repositori Git.
* **IAM / SSO:** Keycloak (OIDC) diimplementasikan sebagai fondasi Centralized Authentication. Keseluruhan ekosistem infrastruktur dan CI/CD, meliputi Rancher (Cluster Management), HashiCorp Vault (Secrets), Jenkins (CI), dan ArgoCD (CD), telah terintegrasi penuh secara OIDC.
* **Shift-Left Security:** Integrasi Trivy Image Scanner pada tahap CI Jenkins sebelum *image* didorong ke Docker Hub.

---

## 3. Kompromi Arsitektur (Resource Constraints)
Menyadari keterbatasan sumber daya perangkat keras (VM lokal) sesuai panduan teknis, eksekusi MVP ini melakukan beberapa penyesuaian fungsional tanpa menghilangkan esensi logika arsitekturnya:
1. **HA Infrastructure:** Dirancang untuk K8s 3 Control-Plane & 3 Worker, namun dieksekusi sebagai *Single-Node* RKE2 Cluster.
2. **HA Database:** Idealnya menggunakan Patroni + PostgreSQL. Namun pada MVP ini dieksekusi sebagai *single-replica* pod untuk menghemat komputasi RAM.
3. **PAM (Privileged Access Management):** Implementasi *JumpServer* (PAM) ditiadakan dalam demo fisik karena beban RAM yang ekstrim (>95% usage). Sebagai solusi, keamanan akses tetap dijamin melalui **Identity-based Access Control** di level Rancher yang terintegrasi OIDC.

---

