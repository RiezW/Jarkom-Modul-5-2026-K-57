# Soal 7: Membangunkan MAGI

## 1. Yang Diminta

1. Memasang **iptables** dan **netcat-traditional** pada ketiga MAGI melalui `/root/<nama>.sh`.
2. Menyalakan listener sesuai glosarium:

| Node | IP | Port TCP |
|---|---|---|
| Magi-Casper | 10.92.4.66 | 5000 |
| Magi-Balthasar | 10.92.4.67 | 5000 dan 5001 |
| Magi-Melchior | 10.92.4.68 | 5000 |

3. Dari **Lilith** (10.92.4.82), mengirim pesan ke ketiganya dan ke port 5001 Balthasar, lalu menampilkan pesan yang diterima di tiap node.

## 2. Menambahkan Bagian Soal 7 ke Skrip MAGI

Skrip `/root/magi-<nama>.sh` dari Soal 6 ditambah bagian baru di bagian bawah. Contoh untuk **Magi-Casper**:

```bash
cat >> /root/magi-casper.sh << 'EOF'

# Soal 7: paket dan listener
apt-get update
apt-get install -y iptables netcat-traditional
update-alternatives --set nc /bin/nc.traditional
cat > /root/listener.sh << 'EOL'
#!/bin/sh
# pakai: sh /root/listener.sh PORT
while true; do
    nc -l -p $1 >> /root/pesan-$1.txt
    sleep 1
done
EOL
pkill -f listener.sh; pkill -f "nc -l -p"
nohup sh /root/listener.sh 5000 > /dev/null 2>&1 &
EOF
```

```bash
sh /root/magi-casper.sh > /root/soal7-install.log 2>&1
```

Untuk **Magi-Melchior** isinya sama (port 5000). Untuk **Magi-Balthasar** ditambah satu listener lagi:

```bash
nohup sh /root/listener.sh 5000 > /dev/null 2>&1 &
nohup sh /root/listener.sh 5001 > /dev/null 2>&1 &
```

Penjelasan:

- **`update-alternatives --set nc /bin/nc.traditional`**: image debinet sudah membawa `netcat-openbsd`, dan setelah `netcat-traditional` dipasang perintah `nc` masih menunjuk ke versi OpenBSD (prioritasnya lebih tinggi). Baris ini memaksa `nc` memakai netcat-traditional sesuai permintaan soal.
- **`/root/listener.sh`**: `nc -l` versi traditional berhenti setelah satu koneksi selesai. Loop `while true` membuat listener langsung terbuka lagi, sehingga bisa menerima banyak pesan. Setiap pesan ditambahkan ke `/root/pesan-<port>.txt`.
- **`pkill` sebelum menyalakan listener**: mencegah listener dobel jika skrip dijalankan ulang.
- **`nohup ... &`**: listener berjalan di latar belakang dan tidak ikut berhenti saat console ditutup.

![Bagian Soal 7 pada skrip Casper](<Assets/soal7_script casper.png>)

![Bagian Soal 7 pada skrip Balthasar](<Assets/soal7_script balthasar.png>)

## 3. Memastikan Listener Aktif

Di setiap MAGI:

```bash
readlink -f $(which nc)
ps aux | grep -v grep | grep -E 'listener|nc -l'
```

Hasil: `nc` menunjuk ke `/usr/bin/nc.traditional`, dan proses `nc -l -p` berjalan sesuai port masing-masing.

![Listener Casper](<Assets/soal7_listener casper.png>)

![Listener Balthasar](<Assets/soal7_listener balthasar.png>)

![Listener Melchior](<Assets/soal7_listener melchior.png>)

## 4. Mengirim Pesan dari Lilith

Lilith memakai `nc` bawaan image (netcat-openbsd). Opsi `-N` menutup koneksi setelah pesan selesai dikirim, dan `-w 2` memberi batas waktu 2 detik.

```bash
kirim() {
  echo "Halo dari Lilith ke $1:$2" | nc -N -w 2 $1 $2
  echo "exit=$?"
}
kirim 10.92.4.66 5000
kirim 10.92.4.67 5000
kirim 10.92.4.67 5001
kirim 10.92.4.68 5000
```

Hasil: keempat pengiriman selesai dengan `exit=0`.

![Pengiriman pesan dari Lilith](<Assets/soal7_kirim lilith.png>)

## 5. Pesan yang Diterima

**Magi-Casper**

```bash
cat /root/pesan-5000.txt
```

![Pesan diterima Casper](<Assets/soal7_pesan casper.png>)

**Magi-Balthasar** (port 5000 dan 5001)

```bash
cat /root/pesan-5000.txt
cat /root/pesan-5001.txt
```

![Pesan diterima Balthasar](<Assets/soal7_pesan balthasar.png>)

**Magi-Melchior**

```bash
cat /root/pesan-5000.txt
```

![Pesan diterima Melchior](<Assets/soal7_pesan melchior.png>)

Setiap MAGI menerima pesan dari Lilith pada port yang benar, termasuk port 5001 pada Balthasar.
