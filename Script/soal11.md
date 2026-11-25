# Soal 11: Casper Harus Jadi Hantu

## 1. Yang Diminta

1. Tidak ada perangkat lain yang boleh mem-ping **Magi-Casper** (10.92.4.66), termasuk tetangga satu switch.
2. Casper sendiri tetap boleh mem-ping siapa saja.
3. Layanan **TCP 5000** Casper tetap hidup.
4. Bukti: ping ke Casper dari Balthasar dan Lilith **gagal**, ping dari Casper ke keduanya **berhasil**.

## 2. Kondisi Sebelum Aturan

Sebelum aturan dipasang, Lilith dan Balthasar masih bisa mem-ping Casper:

```bash
ping -c 2 -W 1 10.92.4.66
```

![Lilith ping Casper sebelum aturan](<Assets/soal11_sebelum lilith.png>)

![Balthasar ping Casper sebelum aturan](<Assets/soal11_sebelum balthasar.png>)

## 3. Memasang Aturan di Casper

Aturan dipasang **di Casper sendiri** (chain `INPUT`), bukan di router. Balthasar berada di switch yang sama dengan Casper, sehingga ping dari Balthasar tidak pernah melewati router Fuyutsuki. Hanya firewall di Casper yang bisa menolaknya.

Bagian ini ditambahkan ke `/root/magi-casper.sh`:

```bash
cat >> /root/magi-casper.sh << 'EOF'

# Soal 11: Casper tidak bisa di-ping
iptables -D INPUT -p icmp --icmp-type echo-request -j DROP 2>/dev/null
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
EOF
```

Menjalankan bagian Soal 11 saja:

```bash
sed -n '/Soal 11/,$p' /root/magi-casper.sh | bash
iptables -S INPUT
```

Penjelasan:

- Yang dibuang hanya `echo-request` (paket ping yang **datang**). Balasan ping milik Casper sendiri bertipe `echo-reply`, sehingga ping keluar dari Casper tetap berhasil.
- Port TCP 5000 tidak disentuh, jadi listener Soal 7 tetap melayani.
- Baris `-D` sebelum `-A` mencegah aturan dobel jika skrip dijalankan ulang.

![Aturan Soal 11 di Casper](<Assets/soal11_rule casper.png>)

## 4. Pengujian

**Ping dari Balthasar ke Casper (gagal, 100% packet loss):**

```bash
ping -c 3 -W 1 10.92.4.66
```

![Balthasar gagal ping Casper](<Assets/soal11_balthasar gagal.png>)

**Ping dari Lilith ke Casper (gagal), dan TCP 5000 masih hidup (`exit=0`):**

```bash
ping -c 3 -W 1 10.92.4.66
echo "tes 5000" | nc -N -w 2 10.92.4.66 5000; echo "exit=$?"
```

![Lilith gagal ping Casper, TCP 5000 tetap hidup](<Assets/soal11_lilith gagal.png>)

**Ping dari Casper ke Balthasar dan Lilith (berhasil):**

```bash
ping -c 2 -W 1 10.92.4.67
ping -c 2 -W 1 10.92.4.82
```

![Casper tetap bisa ping keluar](<Assets/soal11_casper ping keluar.png>)
