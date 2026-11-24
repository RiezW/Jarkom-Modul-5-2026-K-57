# Soal 10: Pengujian Gangguan NAT

## 1. Menjalankan Skrip Pengujian

Siapkan dan jalankan `break_nat.sh` pada Misato dan Gendou. Di masing-masing node, buka file, masukkan skrip pengujian, simpan, lalu jalankan:

```bash
nano /root/break_nat.sh
bash /root/break_nat.sh
```

## 2. Merekam Gejala Sebelum Perbaikan

### Dari Unit-01

Unit-01 berada pada jaringan Switch-Cage dengan alamat `10.92.4.74`. Uji akses internet dan koneksi ke Lilith:

```bash
ping -c 3 8.8.8.8
ping -c 3 10.92.4.82
```

Hasil yang diharapkan: ping ke `8.8.8.8` gagal, tetapi ping ke Lilith (`10.92.4.82`) berhasil.

![Hasil pengujian ping dari Unit-01](<Assets/soal10_test unit1.png>)

### Dari Spy

Spy berada pada jaringan Switch-Intel dengan alamat `10.92.4.102`. Uji akses internet:

```bash
ping -c 3 8.8.8.8
```

![Hasil pengujian ping dari Spy](<Assets/soal10_test spy.png>)

## 3. Mengumpulkan Diagnostik

Jalankan perintah berikut pada Misato dan Gendou, lalu simpan hasilnya. Jangan membuka isi file `break_nat.sh`.

```bash
iptables -S
iptables -t nat -S
iptables -t mangle -S
ip route
```

### Output Misato

![Output diagnostik Misato](<Assets/soal10_dianostic misato.png>)

### Output Gendou

![Output diagnostik Gendou](<Assets/soal10_dianostic gendou.png>)

## 4. Analisis Penyebab Gangguan

### Aturan SNAT Palsu pada Misato

Pada chain `POSTROUTING` tabel NAT Misato terdapat aturan:

```text
-A POSTROUTING -s 10.92.4.72/29 ! -d 10.92.0.0/16 -o eth0 -j SNAT --to-source 172.31.99.1
```

Aturan ini mengubah alamat sumber paket dari Switch-Cage (`10.92.4.72/29`) menjadi alamat palsu `172.31.99.1` saat paket keluar menuju jaringan di luar `10.92.0.0/16`. Gendou hanya mengizinkan forwarding dari sumber `10.92.0.0/16`, sementara kebijakan default chain `FORWARD` adalah `DROP`. Karena itu, paket dengan alamat sumber `172.31.99.1` dibuang di Gendou.

Ping dari Unit-01 ke Lilith (`10.92.4.82`) tetap berhasil karena tujuan tersebut termasuk dalam `10.92.0.0/16`. Kondisi `! -d 10.92.0.0/16` tidak terpenuhi, sehingga aturan SNAT tidak diterapkan dan alamat sumber tetap `10.92.4.74`.

Spy (`10.92.4.102`) berada di jaringan Switch-Intel di belakang Kaji, bukan di subnet Switch-Cage di belakang Misato. Karena itu, trafik Spy tidak cocok dengan aturan SNAT palsu di Misato.

### Aturan `RETURN` pada Gendou

Tabel NAT Gendou juga memiliki aturan berikut sebelum aturan MASQUERADE:

```text
-A POSTROUTING -s 10.92.4.64/29 -o eth0 -j RETURN
```

Aturan ini membuat paket dari subnet `10.92.4.64/29` (Switch-Terminal/MAGI) melewati aturan MASQUERADE berikutnya saat keluar melalui `eth0`. Akibatnya, alamat sumbernya tidak diterjemahkan oleh aturan MASQUERADE Gendou.

### Lokasi Source NAT Normal

Dalam kondisi normal, Gendou menerapkan MASQUERADE pada trafik dari `10.92.0.0/16` yang keluar melalui `eth0`. Alamat sumber diterjemahkan menjadi alamat `eth0` Gendou yang terhubung ke NAT1.

Pada gangguan ini, Misato menerapkan SNAT lebih dahulu dan mengubah alamat sumber menjadi `172.31.99.1` sebelum paket tiba di Gendou.

## 5. Langkah Perbaikan

### Di Misato

Hapus aturan SNAT palsu:

```bash
iptables -t nat -D POSTROUTING -s 10.92.4.72/29 ! -d 10.92.0.0/16 -o eth0 -j SNAT --to-source 172.31.99.1
```

Jika Misato memang tidak menggunakan aturan NAT lain, alternatifnya adalah mengosongkan tabel NAT:

```bash
iptables -t nat -F
```

### Di Gendou

Hapus aturan `RETURN` yang melewati MASQUERADE:

```bash
iptables -t nat -D POSTROUTING -s 10.92.4.64/29 -o eth0 -j RETURN
```

Sebagai alternatif, jalankan kembali skrip konfigurasi Gendou untuk memulihkan aturan NAT yang semestinya:

```bash
bash /root/gendo.sh
```

## 6. Verifikasi Setelah Perbaikan

### Di Misato dan Gendou

Periksa kembali aturan NAT:

```bash
iptables -t nat -S
```

Pastikan aturan SNAT palsu sudah tidak ada di Misato:

![Hasil verifikasi aturan NAT di Misato](<Assets/soal10_verifikasi misato.png>)

Pastikan aturan `RETURN` yang menghalangi MASQUERADE sudah tidak ada di Gendou:

![Hasil verifikasi aturan NAT di Gendou](<Assets/soal10_verifikasi gendou.png>)

### Dari Unit-01

Uji kembali koneksi ke internet:

```bash
ping -c 3 8.8.8.8
```

Pastikan ping berhasil dengan **0% packet loss**.

![Hasil verifikasi koneksi internet dari Unit-01](<Assets/soal10_verifikasi unit1.png>)
