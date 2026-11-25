# Soal 8: Mengenali Kondisi Normal Pen-Pen dan Tokyo3-Core

## 1. Yang Diminta

1. Pen-Pen dan Tokyo3-Core melayani koneksi sebelum dilindungi firewall.
2. Port 80 menjawab **`Welcome to {hostname}`**.
3. Port 8080 di Tokyo3-Core menjawab **`syifa imutz`**.
4. Bukti `nc` dari Lilith ke ketiga port tersebut.
5. Bukti bahwa port yang tidak dilayani (misalnya 22) **langsung gagal**, tidak menggantung.

| Node | IP | Port | Jawaban |
|---|---|---|---|
| Pen-Pen | 10.92.3.195 | 80 | `Welcome to Pen-Pen` |
| Tokyo3-Core | 10.92.4.106 | 80 | `Welcome to Tokyo3-Core` |
| Tokyo3-Core | 10.92.4.106 | 8080 | `syifa imutz` |

## 2. Menambahkan Bagian Soal 8 ke Skrip

Skrip `/root/pen-pen.sh` dan `/root/tokyo3-core.sh` dari Soal 6 ditambah bagian baru. Contoh untuk **Pen-Pen**:

```bash
cat >> /root/pen-pen.sh << 'EOF'

# Soal 8: layanan sederhana
apt-get update
apt-get install -y netcat-traditional
update-alternatives --set nc /bin/nc.traditional
cat > /root/web.sh << 'EOL'
#!/bin/sh
# pakai: sh /root/web.sh PORT "PESAN"
while true; do
    nc -l -p $1 -c "echo '$2'"
    sleep 0.2
done
EOL
pkill -f web.sh; pkill -f "nc -l -p"
nohup sh /root/web.sh 80 "Welcome to $(hostname)" > /dev/null 2>&1 &
EOF
```

```bash
sh /root/pen-pen.sh > /root/soal8-install.log 2>&1
```

Untuk **Tokyo3-Core** sama, ditambah satu layanan untuk port 8080:

```bash
nohup sh /root/web.sh 80 "Welcome to $(hostname)" > /dev/null 2>&1 &
nohup sh /root/web.sh 8080 "syifa imutz" > /dev/null 2>&1 &
```

Penjelasan:

- **`$(hostname)`** diisi otomatis dengan nama node (`Pen-Pen` atau `Tokyo3-Core`), sehingga jawabannya sesuai format `Welcome to {hostname}`.
- **`nc -l -p PORT -c "echo 'PESAN'"`**: setiap ada klien yang tersambung, netcat-traditional menjalankan `echo` untuk mengirim pesan, lalu koneksi ditutup ketika `echo` selesai.
- **Loop `while true`**: setelah satu klien selesai, listener langsung dibuka lagi untuk klien berikutnya.
- **`update-alternatives`**: sama seperti Soal 7, memastikan `nc` memakai netcat-traditional.

![Bagian Soal 8 pada skrip Pen-Pen](<Assets/soal8_script penpen.png>)

![Bagian Soal 8 pada skrip Tokyo3-Core](<Assets/soal8_script tokyo3core.png>)

Catatan: kedua gambar di atas masih menampilkan versi awal `web.sh` (`echo "$2" | nc -l -p $1 -q 1`). Versi tersebut diganti dengan `-c` saat memasang boot hook, lihat [boothook.md](boothook.md). Perilaku dan jawaban setiap port tetap sama.

Pada akhir kedua gambar, `ss -ltn` menunjukkan port 80 (Pen-Pen) serta port 80 dan 8080 (Tokyo3-Core) dalam status `LISTEN`.

## 3. Uji dari Lilith ke Port yang Dilayani

```bash
nc -w 3 10.92.3.195 80 < /dev/null; echo "exit=$?"
nc -w 3 10.92.4.106 80 < /dev/null; echo "exit=$?"
nc -w 3 10.92.4.106 8080 < /dev/null; echo "exit=$?"
```

`< /dev/null` membuat Lilith tidak mengirim apa pun dan hanya menerima jawaban server.

Hasil:

- Pen-Pen port 80: `Welcome to Pen-Pen`
- Tokyo3-Core port 80: `Welcome to Tokyo3-Core`
- Tokyo3-Core port 8080: `syifa imutz`

Ketiganya selesai dengan `exit=0`.

![nc dari Lilith ke port yang dilayani](<Assets/soal8_nc dari lilith.png>)

## 4. Uji Port yang Tidak Dilayani

```bash
time nc -v -w 5 10.92.4.106 22 < /dev/null; echo "exit=$?"
time nc -v -w 5 10.92.3.195 22 < /dev/null; echo "exit=$?"
```

Hasil: `Connection refused` dengan waktu `real` sekitar **0.005 detik**, jauh di bawah batas tunggu 5 detik, dan `exit=1`.

![Port 22 langsung ditolak](<Assets/soal8_port 22 ditolak.png>)

Alasannya: tidak ada program yang mendengarkan port 22, sehingga kernel di server langsung membalas paket `SYN` dengan paket `RST`. Balasan `RST` ini sampai ke Lilith karena firewall Gendou (Soal 5) mengizinkan lalu lintas antar-LAN. Hasil seperti ini menjadi kondisi normal yang akan dibandingkan dengan perilaku setelah firewall dipasang pada soal berikutnya (misalnya Soal 14 dan 16).
