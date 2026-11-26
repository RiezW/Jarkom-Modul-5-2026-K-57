# Soal 20: Backup dan Pemulihan Konfigurasi

## Yang Diminta

Pilih 3 node secara bebas, dengan syarat minimal satu node harus berupa router. Rekomendasi yang paling aman dan jelas:

- **Router 1: Shigeru** (`/root/script.sh`) — memiliki aturan `GERBANG` dari Soal 18 dan `ISOLASI` dari Soal 19.
- **Router 2: Fuyutsuki** (`/root/script.sh`) — memiliki tabel NAT `PREROUTING DNAT` dari Soal 17 serta static route.
- **Client 1: Magi-Casper** (`/root/magi-casper.sh`) — memiliki policy `INPUT DROP` dan aturan firewall dari Soal 11 dan Soal 12.

### Backup Firewall dengan `iptables-save`

Simpan seluruh aturan firewall aktif saat ini ke file `/root/<nama>.rules`, misalnya `/root/shigeru.rules`.

```bash
iptables-save > /root/shigeru.rules
iptables-save > /root/fuyutsuki.rules
iptables-save > /root/magi-casper.rules
```

### Simulasi Bencana atau Hancurkan

- Hapus semua aturan `iptables`, baik pada tabel `filter` maupun tabel `nat`.
- Hapus custom chain.
- Kembalikan default policy ke `ACCEPT` sementara agar konfigurasi benar-benar bersih.
- Hapus rute yang ditambahkan secara manual oleh skrip `/root/<nama>.sh`.
- Rute jaringan yang terhubung langsung atau bersifat `connected` tidak perlu dihapus.

```bash
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

ip route del <network>/<prefix>
```

### Pemulihan Pertama dan Kedua

Pemulihan dilakukan dua kali berturut-turut:

1. Pulihkan firewall dengan `iptables-restore`.
2. Pulihkan rute dan konfigurasi dengan menjalankan skrip node.
3. Ulangi proses yang sama sekali lagi untuk memastikan pemulihan bersifat idempotent.

```bash
iptables-restore < /root/<nama>.rules
bash /root/<nama>.sh

# Ulangi kembali setelah pemulihan pertama
iptables-restore < /root/<nama>.rules
bash /root/<nama>.sh
```

Jika `bash /root/<nama>.sh` dijalankan setelah `iptables-restore`, aturan di dalam skrip tidak boleh menggandakan aturan yang sudah dipulihkan. Skrip pada Soal 11–19 sudah menggunakan `-F` atau `-D` sebelum `-A` atau `-I`, sehingga relatif aman. Rute statis di dalam skrip juga harus aman dijalankan dua kali dan tidak menghasilkan rute dobel.

**Bukti wajib:**

- Hitung jumlah baris aturan setelah pemulihan pertama dan pemulihan kedua.
- Angka `iptables -S | wc -l` dan `iptables -t nat -S | wc -l` harus sama persis pada pemulihan pertama dan kedua.
- Pastikan tidak ada duplikasi atau penambahan aturan.
- Cek tabel routing dengan `ip route` dan pastikan tidak ada rute dobel.
- Buktikan bahwa setiap node kembali berfungsi normal sesuai soal-soal sebelumnya, misalnya melalui pengujian `ping` dan `nc`.

```bash
# Setelah pemulihan pertama
iptables -S | wc -l
iptables -t nat -S | wc -l

# Setelah pemulihan kedua
iptables -S | wc -l
iptables -t nat -S | wc -l

# Pemeriksaan rute
ip route
```

## Node 1: Shigeru (Router 1)

### 1. Simpan Aturan Awal

```bash
iptables-save > /root/shigeru.rules
```

### 2. Simulasi Bencana

Hapus seluruh aturan iptables, termasuk NAT, dan hapus rute statik:

```bash
iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT; iptables -P OUTPUT ACCEPT
iptables -F; iptables -X
iptables -t nat -F; iptables -t nat -X
ip route del default 2>/dev/null
ip route del 10.92.4.56/29 2>/dev/null
iptables -S; ip route
```
![alt text](<Assets/soal20_hapus rule node1.png>)

### 3. Pemulihan ke-1 dan Hitung Jumlah Baris

```bash
iptables-restore < /root/shigeru.rules
bash /root/script.sh
echo "Pemulihan 1 -> filter: $(iptables -S | wc -l) baris | nat: $(iptables -t nat -S | wc -l) baris"
ip route
```
![alt text](<Assets/soal20_pemulihan1 node1.png>)

### 4. Pemulihan ke-2 Berturut-turut dan Hitung Jumlah Baris

Jumlah baris wajib sama persis:

```bash
iptables-restore < /root/shigeru.rules
bash /root/script.sh
echo "Pemulihan 2 -> filter: $(iptables -S | wc -l) baris | nat: $(iptables -t nat -S | wc -l) baris"
iptables -S; ip route
```
![alt text](<Assets/soal20_pemulihan2 node1.png>)

### 5. Bukti Shigeru Berfungsi Kembali

```bash
ping -c 2 -W 1 10.92.4.90
```

Jalankan dari Magi-Casper (harus berhasil) dan dari Lilith (harus gagal).
![alt text](<Assets/soal20_uji ping node1.png>)
![alt text](<Assets/soal20_uji ping2 node1.png>)


## Node 2: Fuyutsuki (Router 2)

Fuyutsuki memiliki aturan NAT DNAT dari Soal 17.

### 1. Simpan Aturan Awal

```bash
iptables-save > /root/fuyutsuki.rules
```

### 2. Simulasi Bencana

Hapus seluruh aturan iptables, termasuk NAT, dan hapus rute statik:

```bash
iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT; iptables -P OUTPUT ACCEPT
iptables -F; iptables -X
iptables -t nat -F; iptables -t nat -X
ip route del default 2>/dev/null
ip route del 10.92.3.192/27 2>/dev/null
ip route del 10.92.4.80/29 2>/dev/null
iptables -t nat -S; ip route
```
![alt text](<Assets/soal20_hapus rule node2.png>)

### 3. Pemulihan ke-1 dan Hitung Jumlah Baris

```bash
iptables-restore < /root/fuyutsuki.rules
bash /root/script.sh
echo "Pemulihan 1 -> filter: $(iptables -S | wc -l) baris | nat: $(iptables -t nat -S | wc -l) baris"
ip route
```
![alt text](<Assets/soal20_pemulihan1 node2.png>)

### 4. Pemulihan ke-2 Berturut-turut dan Hitung Jumlah Baris

Jumlah baris wajib sama persis:

```bash
iptables-restore < /root/fuyutsuki.rules
bash /root/script.sh
echo "Pemulihan 2 -> filter: $(iptables -S | wc -l) baris | nat: $(iptables -t nat -S | wc -l) baris"
iptables -t nat -S; ip route
```
![alt text](<Assets/soal20_pemulihan2 node2.png>)

### 5. Bukti Fuyutsuki Berfungsi Kembali

Jalankan dari Magi-Casper:

```bash
ping -c 2 -W 1 10.92.3.195
```

Pengujian ke Pen-Pen harus berhasil.
![alt text](<Assets/soal20_uji ping node2.png>)


## Node 3: Magi-Casper (Host/Client)

Magi-Casper memiliki policy `INPUT DROP` dari Soal 12.

Cek dahulu baris rute di Magi-Casper:

```bash
cat /root/magi-casper.sh | grep route
```

Biasanya rute default mengarah ke `10.92.4.65`.

### 1. Simpan Aturan Awal

```bash
iptables-save > /root/magi-casper.rules
```

### 2. Simulasi Bencana

Hapus seluruh aturan iptables, termasuk NAT, dan hapus default route:

```bash
iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT; iptables -P OUTPUT ACCEPT
iptables -F; iptables -X
iptables -t nat -F; iptables -t nat -X
ip route del default 2>/dev/null
iptables -S; ip route
```
![alt text](<Assets/soal20_hapus rule node3.png>)

### 3. Pemulihan ke-1 dan Hitung Jumlah Baris

```bash
iptables-restore < /root/magi-casper.rules
bash /root/magi-casper.sh
echo "Pemulihan 1 -> filter: $(iptables -S | wc -l) baris | nat: $(iptables -t nat -S | wc -l) baris"
ip route
```
![alt text](<Assets/soal20_pemulihan1 node3.png>)

### 4. Pemulihan ke-2 Berturut-turut dan Hitung Jumlah Baris

Jumlah baris wajib sama persis:

```bash
iptables-restore < /root/magi-casper.rules
bash /root/magi-casper.sh
echo "Pemulihan 2 -> filter: $(iptables -S | wc -l) baris | nat: $(iptables -t nat -S | wc -l) baris"
iptables -S; ip route
```
![alt text](<Assets/soal20_pemulihan2 node3.png>)

### 5. Bukti Magi-Casper Berfungsi Kembali

Jalankan dari Magi-Casper:

```bash
ping -c 2 -W 2 8.8.8.8
```

Pengujian harus berhasil.
![alt text](<Assets/soal20_uji ping node3.png>)

Jalankan dari Lilith:

```bash
ping -c 2 -W 1 10.92.4.66
```

Pengujian dari Lilith tetap harus gagal atau terblokir.
![alt text](<Assets/soal20_uji ping2 node3.png>)