# 🚗 OtoKlik — Bengkel Mobil
> Laravel 12 + Docker + Ansible + DNS + DHCP  
> SMKN 1 Cibinong | Jurusan SIJA | Ujian Kompetensi Keahlian 2026

---

## 📁 Struktur File

```
repo-ujikombengkel/
├── Dockerfile                    ← Build image PHP Laravel
├── docker-compose.yml            ← Orchestrasi container (app, nginx, db)
├── .env.example                  ← Template konfigurasi environment
├── docker/
│   └── nginx/
│       └── default.conf          ← Konfigurasi Nginx + Load Balancer
└── playbooks/
    ├── hosts                     ← Ansible inventory (VM 2)
    ├── docker.yml                ← Install Docker di VM 2
    ├── dns.yml                   ← Setup BIND9 DNS di VM 2
    └── dhcp.yml                  ← Setup DHCP Server di VM 2
```

---

## 🖥️ Topologi Jaringan

| VM | IP | Service |
|---|---|---|
| VM 1 - ujikom-controller | 192.168.1.10 | Docker, Laravel, Nginx, Ansible |
| VM 2 - ujikom-managed | 192.168.1.11 | DNS (BIND9), DHCP |
| Client Windows | 192.168.1.x (DHCP) | Browser → http://ujikom-putri.net |

---

## 🚀 Cara Deploy

### 1. Clone repo
```bash
git clone https://github.com/username/repo-ujikombengkel.git
cd repo-ujikombengkel
```

### 2. Siapkan .env
```bash
cp .env.example .env
nano .env   # sesuaikan APP_KEY
```

### 3. Build & jalankan container
```bash
docker compose up -d --build
```

### 4. Setup Laravel
```bash
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate --force
docker compose exec app php artisan db:seed --force
docker compose exec app php artisan storage:link
docker compose exec app chmod -R 775 storage bootstrap/cache
docker compose exec app chown -R www-data:www-data storage bootstrap/cache
```

### 5. Buat user admin via tinker
```bash
docker compose exec app php artisan tinker
>>> $user = App\Models\User::create(['name' => 'Admin', 'email' => 'admin@gmail.com', 'password' => bcrypt('12345'), 'role' => 'admin']);
>>> exit
```

### 6. Jalankan Ansible (dari VM 1)
```bash
# Copy SSH key ke VM 2 dulu (sekali saja)
ssh-copy-id root@192.168.1.11

# Copy file hosts ke /etc/ansible/
cp playbooks/hosts /etc/ansible/hosts

# Jalankan semua playbook sekaligus
ansible-playbook playbooks/docker.yml playbooks/dns.yml playbooks/dhcp.yml
```

---

## ✅ Verifikasi

```bash
# Cek container berjalan (harus ada 3 replica app)
docker compose ps

# Cek DNS resolve
nslookup ujikom-putri.net 192.168.1.11

# Cek website
curl -I http://ujikom-putri.net
```

---

## 📝 Catatan

- `scale: 3` → 3 replica app untuk load balancing
- Volume: `volume-ujikom` → data MySQL persisten
- Network: `network-ujikom` → isolasi container
- Healthcheck tersedia di `docker-compose.yml` (nonaktif by default, hapus `#` untuk mengaktifkan)
