# Soal 13: Brankas Balthasar

## 1. Yang Diminta

1. Hanya **Casper** dan semua **Pilot** yang boleh tersambung ke **TCP 5000 dan 5001** di Magi-Balthasar (10.92.4.67).
2. Node lain, termasuk yang satu switch, harus **langsung ditolak**, bukan menggantung sampai timeout.
3. Lalu lintas lain tidak terganggu, dan Balthasar tetap bisa ping `8.8.8.8`.
4. Bukti: `nc` ke port 5000 dan 5001, **Unit-01 berhasil**, **Lilith dan Melchior ditolak seketika**.

| Diizinkan | IP |
|---|---|
| Magi-Casper | 10.92.4.66 |
| Pilot: Unit-01, Unit-02, Unit-00 | 10.92.4.74 - 10.92.4.76 |
| Pilot: Sachiel | 10.92.3.130 |

## 2. Memasang Aturan di Balthasar

Aturan dipasang di Balthasar sendiri karena Melchior berada di switch yang sama dan lalu lintasnya tidak melewati router. Bagian ini ditambahkan ke `/root/magi-balthasar.sh`:

```bash
cat >> /root/magi-balthasar.sh << 'EOF'

# Soal 13: brankas Balthasar (port 5000 dan 5001)
iptables -N BRANKAS 2>/dev/null; iptables -F BRANKAS
M="-p tcp -m multiport --dports 5000,5001"
iptables -D INPUT $M -j BRANKAS 2>/dev/null
iptables -A INPUT $M -j BRANKAS
iptables -A BRANKAS -s 10.92.4.66 -j ACCEPT
iptables -A BRANKAS -m iprange --src-range 10.92.4.74-10.92.4.76 -j ACCEPT
iptables -A BRANKAS -s 10.92.3.130 -j ACCEPT
iptables -A BRANKAS -p tcp -j REJECT --reject-with tcp-reset
EOF
```

```bash
sed -n '/Soal 13/,$p' /root/magi-balthasar.sh | bash
```

![Bagian Soal 13 pada skrip Balthasar](<Assets/soal13_script balthasar.png>)

Penjelasan:

- Hanya paket TCP ke port 5000 dan 5001 yang masuk ke chain `BRANKAS`. Lalu lintas lain (ping, balasan internet) tidak menyentuh chain ini, dan policy `INPUT` tetap `ACCEPT`.
- Casper dan Pilot diterima. Sachiel termasuk Pilot walaupun berada di cabang lain (Switch-Bunker).
- Sisanya ditolak dengan **`REJECT --reject-with tcp-reset`**: Balthasar langsung membalas paket `RST`, sehingga klien langsung mendapat `Connection refused`, bukan menunggu timeout seperti pada `DROP`.

```bash
iptables -S | grep -E 'INPUT|BRANKAS'
```

![Aturan BRANKAS](<Assets/soal13_rule balthasar.png>)

## 3. Pengujian

**Unit-01 ke port 5000 dan 5001 (berhasil):**

```bash
for p in 5000 5001; do echo "Unit-01 ke $p" | nc -N -w 3 10.92.4.67 $p; echo "$p exit=$?"; done
```

![Unit-01 berhasil](<Assets/soal13_unit01 berhasil.png>)

**Lilith ke port 5000 dan 5001 (ditolak seketika, sekitar 0.003 detik):**

```bash
for p in 5000 5001; do time nc -vN -w 3 10.92.4.67 $p </dev/null; done
```

![Lilith ditolak](<Assets/soal13_lilith ditolak.png>)

**Melchior (satu switch dengan Balthasar) ke port 5000 dan 5001 (ditolak seketika):**

```bash
for p in 5000 5001; do time nc -v -w 3 10.92.4.67 $p </dev/null; done
```

![Melchior ditolak](<Assets/soal13_melchior ditolak.png>)

**Di Balthasar:** pesan dari Unit-01 diterima di kedua port, dan ping ke `8.8.8.8` tetap berhasil.

```bash
tail -1 /root/pesan-5000.txt; tail -1 /root/pesan-5001.txt
ping -c 2 -W 2 8.8.8.8
```

![Pesan diterima dan internet Balthasar normal](<Assets/soal13_pesan dan internet balthasar.png>)
