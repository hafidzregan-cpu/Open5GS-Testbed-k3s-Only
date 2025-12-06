# Open5GS-Testbed

Testbed 5G Core Network yang komprehensif mengintegrasikan **Open5GS** dan **UERANSIM** untuk penelitian, testing, dan keperluan edukasi. Testbed ini mendukung multiple network slices dan menyediakan opsi deployment yang fleksibel sesuai dengan skenario testing yang berbeda.

## 📋 Ringkasan

Repository ini menyediakan lingkungan testing 5G standalone (SA) yang lengkap dengan:

- **Open5GS 5G Core Network**: Implementasi 5GC lengkap dengan AMF, SMF, UPF, NRF, dan semua fungsi control plane
- **UERANSIM**: Simulator 5G UE dan RAN (gNB) open-source untuk testing core network
- **Multi-Slice Support**: Tiga network slices yang pre-configured (eMBB, URLLC, mMTC)
- **Flexible Deployment**: Opsi deployment Native, Docker Compose, dan Kubernetes (K3s)

---

## 📋 Instalasi dan Setup

### Step 1: Persiapan Sistem

#### 1. Update Sistem dan Install Dependencies
Lakukan update sistem dan install beberapa dependencies dasar yang diperlukan untuk testbed Open5GS.
```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y \
    curl \
    git \
    iptables \
    iptables-persistent \
    net-tools \
    iputils-ping \
    traceroute \
    tcpdump \
    wireshark \
    wireshark-common
```

#### 2. Install Docker
Docker diperlukan untuk menjalankan beberapa komponen dalam container. Berikut langkah instalasinya:
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
Tambahkan repository Docker:
```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```
Install paket Docker:
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
```

#### 3. Install SCTP Library
Library SCTP diperlukan untuk komunikasi antara komponen 5G Core dan UERANSIM. Install dengan:
```bash
sudo apt-get update && sudo apt-get install -y libsctp1 lksctp-tools
```

#### 4. Install MongoDB
MongoDB digunakan sebagai database untuk Open5GS. Berikut langkah instalasinya:
Install GPG key dan repository MongoDB:
```bash
sudo apt-get install gnupg curl
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list
```
Install MongoDB dan aktifkan service:
```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```
Konfigurasi agar MongoDB dapat diakses dari luar (ubah bindIp):
```bash
sudo nano /etc/mongod.conf
# Ubah bindIp menjadi 0.0.0.0
sudo systemctl restart mongod
```

#### 5. Membuat Direktori Log
Direktori log diperlukan untuk menyimpan log dari Open5GS.
```bash
sudo mkdir -p /mnt/data/open5gs-logs
sudo chmod 777 /mnt/data/open5gs-logs
```

#### 6. Clone Repository
Clone repository testbed ke server Anda:
```bash
git clone https://github.com/rayhanegar/Open5GS-Testbed
```

---

## Step 2: Setup K3s Environment dengan Calico

#### 1. Navigasi ke Direktori K3s
Pindah ke direktori K3s yang berisi script setup:
```bash
cd ~/Open5GS-Testbed/open5gs/open5gs-k3s-calico
```

#### 2. Jalankan Setup Script
Pastikan script dapat dieksekusi dan jalankan setup:
```bash
chmod +x setup-k3s-environment-calico.sh
sudo ./setup-k3s-environment-calico.sh
```

Script akan melakukan beberapa konfigurasi otomatis:
- Install K3s (lightweight Kubernetes)
- Setup Calico CNI untuk networking
- Konfigurasi static IP pool (10.10.0.0/24)
- Setup persistent storage
- Enable SCTP kernel module
- Konfigurasi IP forwarding

#### 3. Verifikasi Instalasi K3s
Setelah setup selesai, verifikasi instalasi K3s dan status node:
```bash
# Cek status K3s
sudo systemctl status k3s

# Cek node Kubernetes
kubectl get nodes
```
Output yang diharapkan:
```
NAME        STATUS   ROLES           AGE   VERSION
<hostname>  Ready    control-plane   Xm    v1.2X.X
```

---

## Step 3: Build dan Import Container Images

Sebelum menjalankan build script, lakukan modifikasi berikut pada beberapa file agar proses build dan import berjalan lancar:

#### 1. Modifikasi build-import-containers.sh
Tambahkan `sudo` sebelum command docker pada file berikut:
`open5gs/open5gs-k3s-calico/build-import-containers.sh`

#### 2. Modifikasi Dockerfile untuk Ping Tools
Tambahkan instalasi ping tools pada setiap Dockerfile di direktori berikut:
`open5gs/open5gs-compose/*/Dockerfile`

Contoh perubahan:
```dockerfile
# Install system dependencies and Open5GS scp in a single layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends software-properties-common gnupg && \
    add-apt-repository ppa:open5gs/latest && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        open5gs-scp \
        open5gs-common \
        gosu \
        ca-certificates \
        netbase \
        iputils-ping \
        curl && \
    mkdir -p /var/log/open5gs /etc/open5gs/tls /etc/open5gs/custom /var/run/open5gs && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### 3. Modifikasi YAML: Ubah Address MongoDB
Ubah address MongoDB di file berikut dengan IP address host Anda:
- `open5gs/open5gs-compose/pcf/pcf.yaml`
- `open5gs/open5gs-compose/udr/udr.yaml`

Contoh perubahan:
```yaml
db_uri: mongodb://192.168.14.137:27017/open5gs
```

#### 4. Jalankan Build Script
Setelah modifikasi selesai, pastikan script dapat dieksekusi dan jalankan build untuk membuat dan mengimport image Open5GS:
```bash
chmod +x build-import-containers.sh
sudo ./build-import-containers.sh
```

#### 5. Verifikasi Image
Setelah proses build selesai, verifikasi image yang sudah diimport ke K3s:
```bash
sudo k3s crictl images
```

---

## Step 4: Deploy Open5GS ke K3s

Sebelum menjalankan deployment, lakukan modifikasi berikut pada beberapa file agar proses deployment berjalan lancar:

#### 1. Modifikasi deploy-k3s-calico.sh
Pastikan fungsi `deploy_pod` sudah benar untuk menghindari false positive error. Contoh cuplikan kode yang benar:
```bash
deploy_pod() {
    local name=$1
    local file=$2
    local label=$3

    POD_DEPLOY_START[$name]=$(get_timestamp_ms)
    print_info "Deploying $name..."

    kubectl apply -f "$file" &>/dev/null

    # Wait for pod to exist first (avoid race condition in parallel deployments)
    local retries=0
    until kubectl get pod -l "$label" -n open5gs &>/dev/null; do
        sleep 0.5
        retries=$((retries + 1))
        if [ $retries -gt 20 ]; then
            POD_DEPLOY_END[$name]=$(get_timestamp_ms)
            POD_READY_TIME[$name]=$(calc_duration ${POD_DEPLOY_START[$name]} ${POD_DEPLOY_END[$name]})
            print_error "$name pod failed to be created after ${POD_READY_TIME[$name]}ms"
            return 1
        fi
    done

    # Wait for pod to be ready (increased timeout for image pull)
    if kubectl wait --for=condition=ready pod -l "$label" -n open5gs --timeout=180s &>/dev/null; then
        POD_DEPLOY_END[$name]=$(get_timestamp_ms)
        POD_READY_TIME[$name]=$(calc_duration ${POD_DEPLOY_START[$name]} ${POD_DEPLOY_END[$name]})
        print_success "$name ready in ${POD_READY_TIME[$name]}ms"
        return 0
    else
        POD_DEPLOY_END[$name]=$(get_timestamp_ms)
        POD_READY_TIME[$name]=$(calc_duration ${POD_DEPLOY_START[$name]} ${POD_DEPLOY_END[$name]})
        print_error "$name failed to start after ${POD_READY_TIME[$name]}ms"
        return 1
    fi
}
```

#### 2. Modifikasi mongod-external.yaml
Ubah IP address MongoDB dengan IP address host Anda pada file berikut:
`open5gs/open5gs-k3s-calico/00-foundation/mongod-external.yaml`

Contoh perubahan:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: open5gs
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - port: 27017
    targetPort: 27017
---
apiVersion: v1
kind: Endpoints
metadata:
  name: mongodb
  namespace: open5gs
subsets:
- addresses:
  - ip: 192.168.14.137  # Ganti dengan IP MongoDB host Anda
  ports:
  - port: 27017
```

#### 3. Jalankan Deployment Script
Setelah modifikasi selesai, pastikan script dapat dieksekusi dan jalankan deployment:
```bash
chmod +x deploy-k3s-calico.sh
sudo ./deploy-k3s-calico.sh
```

Script akan melakukan:
- Membuat namespace `open5gs`
- Setup Calico IPPool
- Membuat service MongoDB
- Deploy Network Function (NF) sesuai urutan dependency
- Generate deployment report

#### 4. Monitor Proses Deployment
Pantau proses deployment untuk memastikan semua pod berjalan dengan baik:
```bash
kubectl get pods -n open5gs -w
```
Tunggu hingga semua pod berstatus `Running` (±2-3 menit).

#### 5. Verifikasi Deployment
Setelah deployment selesai, verifikasi semua pod sudah berjalan:
```bash
kubectl get pods -n open5gs
```
Output yang diharapkan (semua harus Running):
```
NAME      READY   STATUS    RESTARTS   AGE
nrf-0     1/1     Running   0          2m
scp-0     1/1     Running   0          2m
udr-0     1/1     Running   0          2m
udm-0     1/1     Running   0          2m
ausf-0    1/1     Running   0          2m
pcf-0     1/1     Running   0          2m
nssf-0    1/1     Running   0          2m
amf-0     1/1     Running   0          2m
smf-0     1/1     Running   0          2m
upf-0     1/1     Running   0          2m
```

---

## Verifikasi Deployment

### 1. Cek Status Semua NF
Verifikasi semua pod Open5GS sudah berjalan dengan baik:
```bash
# List semua pods dengan detail
kubectl get pods -n open5gs -o wide
```

Cek log untuk NF tertentu jika diperlukan:
```bash
# Check logs untuk NF tertentu
kubectl logs -n open5gs amf-0
kubectl logs -n open5gs smf-0
kubectl logs -n open5gs upf-0
```

<img width="994" height="232" alt="Cuplikan layar dari 2025-11-29 14-42-24" src="https://github.com/user-attachments/assets/2ee92cb0-f2ee-4950-ae63-8c1834268df9" />
---

### 2. Verifikasi Static IP Assignment
Jalankan script verifikasi untuk memastikan setiap NF mendapat static IP sesuai konfigurasi:
```bash
# Run verification script
sudo ./verify-static-ips.sh
```

Output yang diharapkan:
```
✓ nrf-0: 10.10.0.10
✓ scp-0: 10.10.0.200
✓ amf-0: 10.10.0.5
✓ smf-0: 10.10.0.4
✓ upf-0: 10.10.0.7
... (semua NF dengan IP yang sesuai)
```
<img width="894" height="482" alt="Cuplikan layar dari 2025-11-29 14-43-05" src="https://github.com/user-attachments/assets/cf8e75f9-1a28-4c0b-ac98-77387ff083db" />
<img width="1095" height="561" alt="Cuplikan layar dari 2025-11-29 14-43-21" src="https://github.com/user-attachments/assets/cdd20e64-3f15-414c-886a-c285c2e42f74" />


---

### 3. Verifikasi MongoDB Connectivity

#### Pre-requisite: Cache MongoDB Image
Sebelum menjalankan script verifikasi, pastikan image MongoDB sudah ter-cache di K3s untuk menghindari timeout saat pulling image:
```bash
# Pull dan import MongoDB image ke K3s containerd
sudo ctr -n k8s.io images pull docker.io/library/mongo:5.0

# Verifikasi image sudah tersedia
sudo crictl images | grep mongo
```

#### Konfigurasi Script
Ubah `MONGO_IP` pada script `open5gs/open5gs-k3s-calico/verify-mongodb.sh` sesuai dengan IP host Anda:
```bash
sudo nano verify-mongodb.sh
# Ubah MONGO_IP="192.168.14.137" dengan IP host Anda
```

#### Jalankan Verifikasi
```bash
# Run MongoDB verification
sudo ./verify-mongodb.sh
```

Output yang diharapkan:
```
=== MongoDB Connectivity Test ===
Test 1: Network connectivity to 192.168.14.137:27017
✓ Port 27017 is reachable

Test 2: MongoDB authentication
✓ Connected successfully

Test 3: Testing from within K3s cluster...
✓ MongoDB is accessible from within K3s cluster
```

<img width="676" height="404" alt="Cuplikan layar dari 2025-11-29 14-55-19" src="https://github.com/user-attachments/assets/f33064c1-a95a-4536-9567-f3c2677f10f8" />
<img width="626" height="443" alt="Cuplikan layar dari 2025-11-29 14-55-32" src="https://github.com/user-attachments/assets/1769f667-e1cc-4679-ad7c-0f7c3ac921f4" />


---

### 4. Cek Service Connectivity

#### Catatan Penting: Open5GS Menggunakan HTTP/2
Open5GS SBI (Service Based Interface) hanya mendukung **HTTP/2** secara langsung. Saat melakukan testing dengan `curl`, **harus** menggunakan flag `--http2-prior-knowledge`. 

**Perintah yang salah:**
```bash
# ✗ Akan gagal dengan exit code 52 (Empty reply from server)
curl http://nrf:7777/nnrf-nfm/v1/nf-instances
curl --http2 http://nrf:7777/nnrf-nfm/v1/nf-instances
```

**Perintah yang benar:**
```bash
# ✓ Menggunakan HTTP/2 langsung
curl --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances
```

#### Test NF Connectivity
Verifikasi konektivitas antar NF menggunakan NRF API:
```bash
# Test dari AMF pod ke NRF
kubectl exec -it -n open5gs amf-0 -- /bin/bash

# Di dalam pod:
curl --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances
```

Atau test langsung dari NRF pod:
```bash
# Test NRF API dari dalam NRF pod
kubectl exec -n open5gs nrf-0 -- \
    curl -s --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances
```

Output yang diharapkan berupa JSON response dengan daftar NF yang terdaftar:
```json
{
  "_links": {
    "item": [
      {"href": "http://10.10.0.10:7777/nnrf-nfm/v1/nf-instances/..."},
    ],
    "totalItemCount": 9
  }
}
```
<img width="1290" height="276" alt="Cuplikan layar dari 2025-11-29 15-00-58" src="https://github.com/user-attachments/assets/96046018-d641-4544-a4b1-2a66197979df" />


---

## 📋 Tugas 1: Konektivitas Dasar

### Objective

Verify bahwa Open5GS deployment berfungsi dengan benar dan dapat connect dengan UERANSIM.

### Prerequisites
- K3s deployment selesai
- Semua pods running
- UERANSIM binary tersedia

### Langkah-Langkah

#### 1.1 Persiapkan UERANSIM pada Host Eksternal

Navigasi ke direktori UERANSIM:
```bash
# Di mesin yang berbeda dari K3s (atau terminal baru dengan user biasa)
cd ~/Open5GS-Testbed/ueransim
```

**Modifikasi gNB Config:**

1. Dapatkan IP address host dan AMF pod:
```bash
# Cek IP address host
ip addr

# Cek IP address AMF pod
kubectl get pod amf-0 -n open5gs -o wide
```
<img width="1074" height="658" alt="Cuplikan layar dari 2025-11-29 15-02-34" src="https://github.com/user-attachments/assets/efd73277-fc8c-421f-bc9c-cabacaf4a8b7" />
<img width="1126" height="67" alt="Cuplikan layar dari 2025-11-29 15-02-59" src="https://github.com/user-attachments/assets/1af7a985-8efe-4491-b5a6-7d2e039d49e3" />



2. Edit file `ueransim/configs/open5gs-gnb-k3s.yaml` dengan perubahan berikut:

**a. Ubah semua gNB interfaces menggunakan IP host:**

```yaml
linkIp: 192.168.14.137       # Host IP
ngapIp: 192.168.14.137       # Host IP
gtpIp: 192.168.14.137        # Host IP
gtpAdvertiseIp: 192.168.14.137  # Host IP
```
<img width="982" height="79" alt="Cuplikan layar dari 2025-12-05 10-10-16" src="https://github.com/user-attachments/assets/685e4c52-2945-4cea-a554-167387dffc39" />


**b. Ubah AMF address ke IP AMF pod:**

```yaml
amfConfigs:
  - address: 10.10.0.5       # AMF POD IP (dari K3s)
    port: 38412
```
<img width="971" height="138" alt="Cuplikan layar dari 2025-12-05 10-10-32" src="https://github.com/user-attachments/assets/bd539118-d2b1-4d69-8daa-7c982e1c2cee" />


**Catatan:** gNB harus binding ke interface host karena berjalan langsung di host (bukan di dalam K3s cluster). Jika menggunakan pod IP, gNB akan gagal binding dengan error "Cannot assign requested address".

---

#### 1.2 Start gNB Simulator

Jalankan gNB simulator:
```bash
# Terminal 1 - gNB
cd ~/Open5GS-Testbed/ueransim
./build/nr-gnb -c configs/open5gs-gnb-k3s.yaml
```

Output yang diharapkan akan menunjukkan gNB initialized dan ready untuk menerima UE.


<img width="527" height="632" alt="Cuplikan layar dari 2025-11-29 16-05-53" src="https://github.com/user-attachments/assets/8d6da939-02ac-4a2c-b131-b2b35f00af73" />

---

#### 1.3 Start UE Simulator

**Modifikasi UE Config:**

1. Dapatkan IP address host (jika belum):
```bash
# Cek IP address host
ip addr
```
<img width="1074" height="658" alt="Cuplikan layar dari 2025-11-29 15-02-34" src="https://github.com/user-attachments/assets/52f3c01b-64e3-4bab-b650-4ea84a677335" />


2. Edit file `ueransim/configs/open5gs-ue-embb.yaml`:

**Ubah gnbSearchList menggunakan IP host:**

```yaml
gnbSearchList:
  - 192.168.14.137           # Host IP anda (di mana gNB binding)
```
<img width="519" height="71" alt="Cuplikan layar dari 2025-12-05 10-13-09" src="https://github.com/user-attachments/assets/ff7c494b-3ebc-460e-bce6-cc22e510c960" />


**Catatan:** UE perlu mencari gNB menggunakan IP host karena gNB binding ke interface host. Jika menggunakan localhost atau IP lain, UE akan gagal menemukan cell ("no cell in coverage").

3. Jalankan UE simulator:
```bash
# Terminal 2 - UE
cd ~/Open5GS-Testbed/ueransim
sudo ./build/nr-ue -c configs/open5gs-ue-embb.yaml
```

Output yang diharapkan menunjukkan UE berhasil registrasi dan PDU session established.
<img width="889" height="641" alt="Cuplikan layar dari 2025-11-29 15-54-15" src="https://github.com/user-attachments/assets/227da8ca-3bd7-4bd3-b7b8-d82c28de2314" />

---

#### 1.4 Test Basic Connectivity

Buka terminal baru untuk melakukan testing:
```bash
# Terminal 3 - Testing

# Test UE TUN interface
ip addr show uesimtun0
```
<img width="889" height="211" alt="Cuplikan layar dari 2025-11-29 15-55-25" src="https://github.com/user-attachments/assets/dffd3a46-bd19-4788-b250-4c106142f868" />


```bash
# Test gateway connectivity (UE -> UPF)
ping -I uesimtun0 -c 4 10.45.0.1
```
<img width="662" height="209" alt="Cuplikan layar dari 2025-11-29 15-55-50" src="https://github.com/user-attachments/assets/11a4ca95-a6a9-437e-b0e3-c4e420f7e5fe" />


```bash
# Test internet connectivity
ping -I uesimtun0 -c 4 8.8.8.8
```
<img width="644" height="178" alt="Cuplikan layar dari 2025-11-29 15-56-20" src="https://github.com/user-attachments/assets/f49f120b-7385-40a2-bc9c-963843bf4197" />


```bash
# Test DNS resolution
nslookup google.com 8.8.8.8
```
<img width="648" height="436" alt="Cuplikan layar dari 2025-11-29 15-56-47" src="https://github.com/user-attachments/assets/8b728c42-0375-4420-90f2-93ed633fb802" />


```bash
# Test HTTP/HTTPS
curl --interface uesimtun0 -I https://www.google.com
```
<img width="884" height="355" alt="Cuplikan layar dari 2025-11-29 15-57-15" src="https://github.com/user-attachments/assets/55e95544-cd1c-4a21-8192-902de791ae4d" />

---

#### 1.5 Dokumentasi Hasil

## Tugas 1: Konektivitas Dasar - Hasil Testing eMBB

**Tanggal**: [29 November 2025]
**Nama**: [Moh. Zukhruf Artha Hafidz]
**Status K3s**: [WORKING]

### gNB Registration
- Status: [SUCCESS]
- Time taken: [17ms]
- AMF Connection: [ESTABLISHED]

### UE Registration
- Status: [SUCCESS]
- Time taken: [751ms]
- IMSI: [imsi-001011000000001]
- TUN Interface: [uesimtun0]
- IP Address: [10.45.0.2]

### Connectivity Tests
| Test | Result | RTT (ms) |
|------|--------|----------|
| UPF Gateway (10.45.0.1) | ✓ PASS | 0.065 |
| Internet (8.8.8.8) | ✓ PASS | 142 |
| DNS Resolution | ✓ PASS | - |
| HTTP/HTTPS | ✓ PASS | - |

---

## Issues Encountered

### Issue 1: Calico Pod Initialization Failure
**Problem:** Pod Calico kube-controllers mengalami CrashLoopBackOff, calico-node stuck pada status Init:2/3

**Akar Masalah:** 
- Terjadi masalah bootstrap pada Calico (chicken-and-egg problem)
- Calico memerlukan akses ke Kubernetes API server (10.43.0.1) sebelum networking siap
- Ini menyebabkan deadlock pada saat initialization

---
### Issue 2: UERANSIM Binary Incompability
**Problem:** Binary UERANSIM tidak bisa dijalankan dengan error `./build/nr-gnb: version 'GLIBC_2.38' not found`

**Akar Masalah:** 
- Binary UERANSIM pre-compiled yang disediakan dibuat pada Ubuntu 24.04 dengan glibc 2.38
- Sistem yang digunakan adalah Ubuntu 22.04 dengan glibc 2.35
- Terjadi incompatibility pada library level

---
### Issue 3: UE Registration Failure [FIVEG_SERVICES_NOT_ALLOWED]
**Problem** Registrasi UE gagal dengan kode error 7, logs menunjukkan "Cannot find SUCI [404]"

**Akar Masalah:**
- Schema dokumen subscriber di MongoDB tidak sesuai dengan format yang diharapkan Open5GS
- Field `security` tidak tersarang dengan benar di dalam `subscriptionData`
- AMF tidak bisa menemukan data subscriber dan menolak registrasi UE

---
### Issue 4: K3s Setup Script Shell Errors
**Problem** Script K3s mengalami error pada logic perbandingan pod count di lines 560 dan 568

**Akar Masalah:**
- Logic perbandingan pod count di dalam script tidak bekerja sesuai yang diharapkan
- Command `grep -c` tidak return nilai yang konsisten
- Script gagal melakukan conditional check untuk pod readiness

---
### Issue 5: SCP Cannot Ping NRF
**Problem:** SCP pod tidak memiliki utilitas `ping` yang diperlukan untuk testing konektivitas jaringan antar NF.

**Error Message:**
```
/bin/sh: ping: not found
```
**Root Cause:** Image SCP tidak memiliki `iputils-ping` package terinstall.
Tambahkan `iputils-ping` ke dalam Dockerfile SCP:

---
### Issue 6: MongoDB Connection from K3s Cluster Fails
**Problem:** Pod Open5GS di dalam K3s cluster tidak dapat terhubung ke MongoDB yang berjalan di host.

**Error Message:**
```
MongoDB connection failed: Connection refused
```

**Root Cause:** MongoDB default binding hanya ke `127.0.0.1` (localhost), sehingga tidak dapat diakses dari pod K3s.

---
### Issue 7: UERANSIM gNB Binary Failed to Start
**Problem:** Binary UERANSIM gNB gagal start karena missing SCTP library dependency.

**Error Message:**
```
error while loading shared libraries: libsctp.so.1: cannot open shared object file
```

**Root Cause:** Library SCTP belum terinstall di host system.

---
### Issue 8: gNB Failing to Bind to Interfaces
**Problem:** gNB simulator gagal binding ke interfaces yang dikonfigurasi (`linkIp`, `ngapIp`, `gtpIp`, `gtpAdvertiseIp`).

**Error Message:**
```
[ERROR] Cannot assign requested address
```

**Root Cause:** gNB config menggunakan IP address pod K3s, padahal gNB berjalan langsung di host (bukan di dalam cluster). Interface binding harus menggunakan IP address host.

---
### Issue 9: UE Cannot Find Any Cells in Coverage
**Problem:** UE simulator gagal menemukan cell dari gNB.

**Error Message:**
```
[rrc] [error] Cell search failed, no cell in coverage
```

**Root Cause:** UE config `gnbSearchList` tidak sesuai dengan IP address dimana gNB sebenarnya binding (host IP).

### Issue 10: curl Error 52 When Testing NRF API
**Problem:** Saat testing NRF API dengan curl standard, mendapat error "Empty reply from server".

**Error Message:**
```
command terminated with exit code 52
```
---
**Root Cause:** Open5GS SBI menggunakan HTTP/2 secara eksklusif. Curl standard mengirim HTTP/1.1 yang ditolak oleh NRF.

### Issue 11: MongoDB Verification Timeout
**Problem:** Script `verify-mongodb.sh` timeout saat menjalankan test dari dalam K3s cluster.

**Root Cause:** Image `mongo:5.0` (275MB) belum ter-cache di K3s containerd, menyebabkan timeout saat pulling image.

---
## Resolution
### 1. Restart service K3s for re-initialization
**Resolution:**
1. Restart service K3s untuk melakukan re-initialization:
   ```bash
   sudo systemctl restart k3s
   ```
2. Tunggu proses initialization Calico pods selesai dengan proper sequence
3. Verifikasi pod Calico sudah mencapai status Running

---
### 2. Install compability version for ubuntu 22.04
**Resolution:**
1. Install tools untuk compile dari source:
   ```bash
   sudo apt-get install -y build-essential cmake libsctp-dev
   ```
2. Rebuild UERANSIM dari source code:
   ```bash
   cd ~/Open5GS-Testbed/ueransim
   make build
   ```
3. Verifikasi binary baru kompatibel dengan sistem:
   ```bash
   ./build/nr-gnb --help
   ./build/nr-ue --help
   ```

---
### 3. Add Right Schema Subscriber in MongoDB
**Resolution:**
1. Tambahkan subscriber ke MongoDB dengan schema yang benar:
   ```bash
   mongosh "mongodb://10.201.67.251:27017/open5gs" << 'EOF'
   db.subscribers.insertOne({
     "_id": "001011000000001",
     "imsi": "001011000000001",
     "subscriptionData": {
       "security": {
         "authenticationSubscriptions": [{
           "sequenceNumber": "000000000061",
           "authenticationMethod": "5G_AKA",
           "permanentKey": "0C0A34601D4F07677303652C0624006F",
           "operatorDeterminedBarring": 0,
           "opc": "FAB0F1D1A00002D31649A5AEB245B89D"
         }]
       },
       "accessAndMobilitySubscriptionData": {
         "offeredSBIFqdn": "nrmm.5g.mnc01.mcc001.3gppnetwork.org"
       },
       "sessionManagementSubscriptionData": [{
         "singleNssai": {"sst": 1, "sd": "000001"},
         "dnnConfigurations": {
           "embb.testbed": {"sscModes": {"defaultSscMode": "SSC_MODE_1"}}
         }
       }],
       "smfSelectionSubscriptionData": {},
       "securitySubscriptionData": {}
     }
   })
   EOF
   ```

2. Pastikan field `_id` disesuaikan dengan IMSI sebagai string value

3. Restart pods authentication (AUSF, UDM, UDR) untuk membersihkan cache data lama:
   ```bash
   kubectl delete pod -n open5gs ausf-0 udm-0 udr-0
   ```
   
---
### 4. Edit Code in lines 560 & 568
**Resolution:**
1. Perbaiki logic conditional di file `setup-k3s-environment-calico.sh` pada lines 560 dan 568
<img width="1145" height="299" alt="Cuplikan layar dari 2025-12-06 10-26-12" src="https://github.com/user-attachments/assets/ff0f3549-ed80-4be3-8742-fe88ec9f7ff3" />

2. Test ulang script untuk memastikan tidak ada error


---
### 5. Install Ping Utility in SCP Container
**Resolution:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        open5gs-scp \
        iputils-ping \
        curl && \
    apt-get clean
```
Rebuild dan reimport image.
```bash
cd ~/Open5GS-Testbed/open5gs/open5gs-k3s-calico
sudo ./build-import-containers.sh
```
---
### 6. Configure MongoDB to Accept External Connections
**Resolution:**
Edit konfigurasi MongoDB untuk binding ke semua interface:
```bash
sudo nano /etc/mongod.conf
```
Ubah `bindIp`:
```yaml
net:
  port: 27017
  bindIp: 0.0.0.0  # Ubah dari 127.0.0.1
```
Restart MongoDB:
```bash
sudo systemctl restart mongod
```

---

### 7. Install SCTP Library on Host
**Resolution:**
Install library SCTP yang diperlukan UERANSIM:
```bash
sudo apt-get update && sudo apt-get install -y libsctp1 lksctp-tools
```
Verifikasi instalasi:
```bash
ldconfig -p | grep sctp
```

---

### 8. Configure gNB to Use Host IP Address
**Resolution:**
Configure gNB to Use Host IP Address:
Edit file `ueransim/configs/open5gs-gnb-k3s.yaml`:
```yaml
linkIp: 192.168.14.137    # Ganti dengan IP host Anda
ngapIp: 192.168.14.137    # Ganti dengan IP host Anda  
gtpIp: 192.168.14.137     # Ganti dengan IP host Anda
gtpAdvertiseIp: 192.168.14.137  # Ganti dengan IP host Anda

amfConfigs:
  - address: 10.10.0.5    # IP address AMF pod di K3s
    port: 38412
```

---

### 9. Configure UE gnbSearchList to Use Host IP
**Resolution:**
Configure UE gnbSearchList to Use Host IP:
Edit file `ueransim/configs/open5gs-ue-embb.yaml`:
```yaml
gnbSearchList:
  - 192.168.14.137    # Ganti dengan IP host Anda (sama dengan gNB binding)
```

---

### 10. Use Correct HTTP/2 Flag for culr
**Resolution:**
Use Correct HTTP/2 Flag for curl:
```bash
# Correct command
curl --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances

# Wrong commands (will fail with exit code 52)
curl http://nrf:7777/nnrf-nfm/v1/nf-instances
curl --http2 http://nrf:7777/nnrf-nfm/v1/nf-instances
```

---
### 11. Pre-cache MongoDB Image Before Verification
**Resolution:**
Pre-cache MongoDB Image Before Verification:
```bash
# Pull image ke K3s containerd
sudo ctr -n k8s.io images pull docker.io/library/mongo:5.0

# Verifikasi image tersedia
sudo crictl images | grep mongo

# Sekarang jalankan script verifikasi
sudo ./verify-mongodb.sh
```

---

