# Soal 15: Banjir Ping di Pusat Komando

## 1. Yang Diminta

1. **Tokyo3-Core** (10.92.4.106) hanya menjawab maksimal **5 ping per detik**, kelebihannya dibuang.
2. Aturan tidak mengganggu ping di bawah batas, dan tidak membatasi ping yang dikirim Tokyo3-Core sendiri.
3. Bukti: ping rapat dari Lilith (`ping -i 0.1 -c50`) sebagian hilang dan counter rule pembuangan naik, sedangkan ping biasa semuanya berhasil (jeda minimal 3 detik).

## 2. Memasang Aturan

Bagian ini ditambahkan ke `/root/tokyo3-core.sh`:

```bash
cat >> /root/tokyo3-core.sh << 'EOF'

# Soal 15: batasi ping masuk maksimal 5 per detik
iptables -N BATASPING 2>/dev/null; iptables -F BATASPING
P="-p icmp --icmp-type echo-request"
iptables -D INPUT $P -j BATASPING 2>/dev/null
iptables -I INPUT 1 $P -j BATASPING
iptables -A BATASPING -m limit --limit 5/second --limit-burst 5 -j ACCEPT
iptables -A BATASPING -j DROP
EOF
```

```bash
sed -n '/Soal 15/,$p' /root/tokyo3-core.sh | bash
iptables -S | grep -E 'INPUT|BATASPING'
```

![Aturan Soal 15](<Assets/soal15_script tokyo3core.png>)

Penjelasan:

- **Jenis paket:** hanya `echo-request` (ping yang datang) yang masuk ke chain. Ping yang dikirim Tokyo3-Core keluar dan balasannya (`echo-reply`) tidak terpengaruh.
- **Urutan:** aturan lompat ke `BATASPING` dipasang di posisi pertama `INPUT` (`-I INPUT 1`), sehingga tidak ada aturan lain yang menerima ping lebih dulu.
- **`--limit 5/second --limit-burst 5`:** paket yang masih dalam kuota diterima, kelebihannya jatuh ke aturan berikutnya dan dibuang (`DROP`). Burst 5 menjaga batas tetap 5 per detik sejak paket pertama.

## 3. Pengujian Ping Rapat

Counter direset, lalu Lilith mengirim 50 ping dengan jeda 0.1 detik (10 ping per detik):

```bash
iptables -Z BATASPING
```

```bash
ping -i 0.1 -c 50 -q 10.92.4.106
```

Hasil: 50 terkirim, 30 diterima, **40% packet loss**.

![Ping rapat dari Lilith](<Assets/soal15_ping rapat lilith.png>)

Counter di Tokyo3-Core: **30 paket ACCEPT** dan **20 paket DROP**, sesuai jumlah yang hilang di Lilith.

```bash
iptables -L BATASPING -v -n
```

![Counter BATASPING](<Assets/soal15_counter batasping.png>)

## 4. Pengujian Ping Biasa

Setelah jeda 3 detik, ping biasa (1 per detik) semuanya berhasil:

```bash
sleep 3; ping -c 5 10.92.4.106
```

![Ping biasa dari Lilith](<Assets/soal15_ping biasa lilith.png>)

## 5. Ping Keluar dari Tokyo3-Core Tidak Dibatasi

Tokyo3-Core mengirim ping rapat ke Lilith, hasilnya 0% packet loss:

```bash
ping -i 0.1 -c 30 -q 10.92.4.82
```

![Ping keluar Tokyo3-Core](<Assets/soal15_ping keluar tokyo3core.png>)
