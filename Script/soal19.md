# Soal 19: Isolasi pada Switch-Monolith

## 1. Yang Diminta

1. Mengisolasi total **Armisael** (`10.92.4.91`) agar tidak bisa berkomunikasi dengan siapa pun (termasuk gateway maupun tetangga satu switch), tanpa memblokir **Zeruel** (`10.92.4.90`).
2. Isolasi **tidak boleh dikerjakan di dalam Armisael**.
3. Menyalakan listener TCP port `9000` di Armisael, Zeruel, dan Lilith, serta menggunakan TCP port `53` untuk Seele (`8.8.8.8`).
4. Bukti: `ping` dan `nc` dari **Armisael** ke Seele, Zeruel, dan Lilith **gagal**, sedangkan `ping` dan `nc` dari **Zeruel** ke Seele dan Lilith **berhasil**.

## 2. Mengisolasi Armisael melalui VLAN

Buka **Switch-Monolith** di GNS3 GUI, lalu pilih **Configure**.

1. Pilih baris **Port 2** pada tabel Ports atau isi kotak **Port** dengan nilai `2`.
2. Ubah VLAN dari `1` menjadi `99`.
3. Klik **Add**.
4. Pastikan hanya **Port 2** yang berada pada VLAN `99`, sedangkan **Port 0** dan **Port 1** tetap pada VLAN `1`.
![alt text](<Assets/soal19_konfigurasi switch-monolith.png>)



## 3. Memasang Pengamanan Lapis Kedua di Shigeru

Tambahkan aturan berikut ke `/root/script.sh` Shigeru:

```bash
cat >> /root/script.sh << 'EOF'

# Soal 19: isolasi total Armisael (10.92.4.91)
iptables -N ISOLASI 2>/dev/null; iptables -F ISOLASI

for c in INPUT OUTPUT FORWARD; do
  iptables -D $c -j ISOLASI 2>/dev/null
  iptables -I $c 1 -j ISOLASI
done

iptables -A ISOLASI -s 10.92.4.91 -j DROP
iptables -A ISOLASI -d 10.92.4.91 -j DROP
EOF
```

Jalankan skrip dan verifikasi aturannya:

```bash
bash /root/script.sh
```
```bash
sed -n '/Soal 19/,$p' /root/script.sh | bash
iptables -S | grep -E 'INPUT|OUTPUT|FORWARD|ISOLASI'
```

Penjelasan:

- Port 2 dipisahkan ke VLAN `99` pada lapisan Layer 2.
- Chain `ISOLASI` pada Shigeru memblokir paket yang bersumber atau ditujukan ke `10.92.4.91`.
- Aturan diterapkan pada `INPUT`, `OUTPUT`, dan `FORWARD`.
- Aturan tidak dipasang di dalam Armisael.

## 4. Menyalakan Listener TCP Port 9000

Jalankan listener pada Armisael, Zeruel, dan Lilith:

```bash
nc -l 9000 >/dev/null 2>&1 &
```

Jika Netcat memerlukan opsi `-p`, gunakan:

```bash
nc -l -p 9000 >/dev/null 2>&1 &
```

## 5. Pengujian dari Armisael

Semua pengujian berikut wajib **gagal**:

```bash
ping -c 2 -W 1 8.8.8.8
ping -c 2 -W 1 10.92.4.90
ping -c 2 -W 1 10.92.4.82

nc -zv -w 2 8.8.8.8 53
nc -zv -w 2 10.92.4.90 9000
nc -zv -w 2 10.92.4.82 9000
```

Hasil yang diharapkan:

- Ping menghasilkan `100% packet loss`.
- Koneksi Netcat berakhir dengan timeout atau ditolak.
![alt text](<Assets/soal19_uji dari armisael.png>)

## 6. Pengujian dari Zeruel

Jalankan pengujian berikut dari Zeruel:

```bash
ping -c 2 -W 1 8.8.8.8
ping -c 2 -W 1 10.92.4.82

nc -zv -w 2 8.8.8.8 53
nc -zv -w 2 10.92.4.82 9000
```

Hasil yang diharapkan:

- Ping menghasilkan `0% packet loss`.
- Koneksi Netcat ke port yang diuji berhasil.
![alt text](<Assets/soal19_uji dari zaruel.png>)