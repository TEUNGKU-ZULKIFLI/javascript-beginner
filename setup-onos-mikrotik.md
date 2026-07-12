# Cara Implementasi RouterBoard (RB951G-2HnD) ke SDN ONOS (ONOS 2.6 latest Docker)

**Dokumentasi Historikal & Panduan Standar Operasional (SOP)**

Panduan ini menjelaskan cara mengintegrasikan perangkat jaringan lawas (MikroTik RouterOS v6) dengan arsitektur modern Software-Defined Networking (SDN) menggunakan ONOS Controller yang berjalan di atas Docker.

## ⚠️ Latar Belakang Masalah & *Troubleshooting* Utama

Sebelum masuk ke konfigurasi, penting untuk memahami masalah kompatibilitas *port* yang sering membuat integrasi ini gagal:

* **Keterbatasan MikroTik v6:** RouterOS versi 6 (seperti v6.49.20) memiliki pengaturan OpenFlow yang kaku. Sistem secara *hardcode* selalu mencoba terhubung ke controller melalui port lawas OpenFlow, yaitu **6633**. MikroTik akan menolak sintaks modifikasi port pada controller (memunculkan pesan error `invalid value for argument controller`).
* **Standar Baru ONOS 2.6:** Controller ONOS versi modern secara default mendengarkan koneksi OpenFlow pada port standar baru, yaitu **6653**.
* **Solusi Cerdas:** Alih-alih melakukan *port forwarding* manual menggunakan `iptables` di sistem operasi Host, kita menyelesaikan ini secara elegan langsung di dalam parameter eksekusi Docker. Kita memetakan (*mapping*) port masuk `6633` di mesin Host agar diteruskan ke port `6653` di dalam kontainer ONOS (Parameter: `-p 6633:6653`).

---

## TAHAP 1: Persiapan Sisi MikroTik (RouterOS)

### 1. Unduh Paket OpenFlow

MikroTik tidak menyertakan OpenFlow secara bawaan. Anda harus mengunduh paket ekstrak (Extra Packages).

* Kunjungi situs resmi MikroTik (halaman *Downloads*).
* Sesuaikan arsitektur router Anda (Untuk RB951G-2HnD, gunakan **mipsbe**).
* Unduh file `.zip` (Extra packages), ekstrak, dan cari file bernama `openflow-xxx.npk`.

### 2. Instalasi Paket ke Router

* Buka Winbox, masuk ke menu **Files**.
* Klik ikon **Upload** (atau *drag-and-drop*) file `.npk` tersebut ke daftar file.
* **Reboot** router Anda (`/system reboot`). Proses *booting* akan sedikit lebih lama karena sedang melakukan instalasi.

### 3. Verifikasi Instalasi

* Setelah menyala, pastikan menu **OpenFlow** sudah muncul pada *sidebar* sebelah kiri di Winbox.

### 4. Bersihkan Konfigurasi Port (*Data Plane*)

**Langkah Krusial:** Port fisik yang akan dialihkan ke ONOS SDN (*Data Plane*) harus bersih dari intervensi CPU MikroTik.

* Tentukan port mana yang akan dijadikan *Switch SDN* (Contoh: `ether2-Host` dan `ether4-Host`).
* Pastikan **TIDAK ADA IP Address** atau konfigurasi DHCP Server yang terpasang di port tersebut.
* *Catatan:* Jangan pernah memasukkan port yang digunakan untuk *Control Plane* (jalur koneksi ke Server ONOS) ke dalam Switch OpenFlow.

---

## TAHAP 2: Konfigurasi Switch OpenFlow di MikroTik

Anda dapat memilih salah satu dari dua metode di bawah ini (CLI atau GUI) untuk mendaftarkan switch dan port. Asumsi IP Server ONOS adalah `192.168.40.2`.

### Metode A: Via Terminal (CLI)

Buka New Terminal di Winbox dan jalankan perintah berikut:

1. Membuat virtual switch OpenFlow:

```routeros
/openflow add name=sw-onos controllers=192.168.40.2 disabled=no datapath-id=1/64:D1:54:9F:B8:28

```

2. Mendaftarkan port fisik ke dalam switch:

```routeros
/openflow port add interface=ether2-Host switch=sw-onos disabled=no 
/openflow port add interface=ether4-Host switch=sw-onos disabled=no 

```

### Metode B: Via Winbox (GUI)

1. **Membuat Switch:**
* Masuk ke menu **OpenFlow** > tab **Switches** > klik tombol **Add (+)**.
* **Name:** `sw-onos`
* **Datapath ID:** `1/64:D1:54:9F:B8:28`
* **Passive Port:** `{none}`
* **Controllers:** `192.168.40.2`
* Pastikan tidak tercentang *Disable*, lalu klik **Apply** dan **OK**.


2. **Mendaftarkan Port:**
* Pindah ke tab **Ports** > klik tombol **Add (+)**.
* **Interface:** Pilih `ether2-Host`
* **Switch:** Pilih `sw-onos`
* Klik **Apply** dan **OK**. Ulangi langkah ini untuk `ether4-Host`.



---

## TAHAP 3: Instalasi dan Setup Sisi ONOS (Docker)

Pastikan server/PC Anda sudah terinstal Docker Engine.

### 1. Pull Image ONOS

Unduh *image* versi 2.6 terbaru dari *repository* resmi:

```bash
docker pull onosproject/onos:2.6-latest

```

### 2. Jalankan Kontainer ONOS (Solusi Pemetaan Port)

Eksekusi *script* di bawah ini. Perhatikan parameter `-p 6633:6653` yang berfungsi sebagai jembatan kompatibilitas antara port lawas MikroTik dan port modern ONOS.

```bash
docker run -d -p 6633:6653 -p 8181:8181 -p 8101:8101 -e ONOS_APPS=openflow,reactive.fwd,gui2 --name onos onosproject/onos:2.6-latest

```

### 3. Akses Konsol ONOS (Karaf)

Tunggu sekitar 1-2 menit hingga ONOS selesai melakukan *booting* layanan, lalu masuk ke konsol *command line* menggunakan SSH:

```bash
# Login menggunakan user karaf
ssh -p 8101 karaf@127.0.0.1
# Password: karaf

# ATAU Login menggunakan user onos
ssh -p 8101 onos@127.0.0.1
# Password: rocks

```

### 4. Verifikasi Topologi di ONOS

Gunakan perintah berikut di dalam konsol Karaf untuk memastikan jaringan berjalan dengan baik:

* `apps -a -s` : Melihat daftar aplikasi dasar (pastikan `openflow` dan `reactive.fwd` berstatus *Active*).
* `devices` : Memastikan MikroTik RouterBoard sudah dikenali oleh ONOS (Status harus *Available = true*).
* `ports` : Melihat daftar port (`ether2` dan `ether4`) yang tadi didaftarkan di MikroTik.
* `hosts` : Menampilkan perangkat (PC/Laptop) *end-user* yang terhubung ke port MikroTik. *(Catatan: Host baru akan muncul setelah perangkat PC secara aktif mengirimkan paket jaringan seperti ping atau ARP)*.
