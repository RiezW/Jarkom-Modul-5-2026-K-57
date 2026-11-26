# Soal 18: Gerbang Monolith dan Radar

## 1. Yang Diminta

1. Jadikan **Shigeru** sebagai firewall stateful untuk melindungi zona **Monolith** (`eth2`) dan zona **Radar** (`eth1` melalui Makoto).
2. Semua koneksi baru yang datang dari arah **Gendou** (`eth0`) ditolak, kecuali ping dari subnet Terminal.
3. Balasan untuk koneksi yang sudah terdaftar atau berkaitan tetap diizinkan.
4. Bukti: ping dari Magi-Casper ke Zeruel berhasil, ping dari Lilith ke Zeruel gagal, dan ping keluar dari Zeruel berhasil.

## 2. Memasang Aturan di Shigeru

Bagian ini ditambahkan ke `/root/script.sh` Shigeru:

```bash
cat >> /root/script.sh << 'EOF'

# Soal 18: gerbang Monolith dan Radar berbasis arah interface eth0
iptables -N GERBANG 2>/dev/null; iptables -F GERBANG
iptables -D FORWARD -j GERBANG 2>/dev/null
iptables -I FORWARD 1 -j GERBANG
iptables -A GERBANG -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A GERBANG -i eth0 -s 10.92.4.64/29 -p icmp --icmp-type echo-request -j ACCEPT
iptables -A GERBANG -i eth0 -j DROP
EOF
```

Jalankan skrip dan verifikasi aturannya:

```bash
bash /root/script.sh
```
```bash
sed -n '/Soal 18/,$p' /root/script.sh | bash
iptables -S | grep -E 'FORWARD|GERBANG'
```
![alt text](<Assets/soal18_ketika scritp shigeru jalan.png>)

Penjelasan:

- `ESTABLISHED,RELATED` menerima balasan dan paket lanjutan dari koneksi yang sudah dikenal.
- Hanya ping ICMP masuk dari subnet Terminal `10.92.4.64/29` melalui `eth0` yang diizinkan.
- Semua paket baru yang masih masuk melalui `eth0` dibuang. Aturan ini melindungi Monolith dan Radar dari koneksi masuk melalui arah Gendou.

## 3. Pengujian dari Magi-Casper

Magi-Casper (`10.92.4.66`) berada di subnet Terminal. Ping ke Zeruel (`10.92.4.90`) wajib berhasil:

```bash
ping -c 3 -W 1 10.92.4.90
```
![alt text](<Assets/soal18_ping dari magi-casper.png>)

Hasil yang diharapkan: **0% packet loss**.

## 4. Pengujian dari Lilith

Lilith (`10.92.4.82`) tidak berada di subnet Terminal. Ping dari Lilith ke Zeruel wajib ditolak:

```bash
ping -c 3 -W 1 10.92.4.90
```
![alt text](<Assets/soal18_ping dari lilith.png>)

Hasil yang diharapkan: **100% packet loss**.

## 5. Pengujian Koneksi Keluar dari Zeruel

Zeruel dapat tetap berkomunikasi ke luar. Jalankan dari Zeruel (`10.92.4.90`):

```bash
ping -c 2 -W 1 10.92.4.82
ping -c 2 -W 2 8.8.8.8
```
![alt text](<Assets/soal18_ping dari zeruel.png>)

Hasil yang diharapkan: ping ke Lilith dan `8.8.8.8` masing-masing berhasil.

## 6. Melihat Counter Rule

Counter dapat dilihat di Shigeru untuk membuktikan bahwa paket Terminal diterima dan paket lain dibuang:

```bash
iptables -Z GERBANG
iptables -L GERBANG -v -n
```
![alt text](<Assets/soal18_counter rule shigeru.png>)