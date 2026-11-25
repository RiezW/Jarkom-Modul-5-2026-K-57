# Soal 12: Teknisi yang Kelewat Semangat

## 1. Yang Diminta

1. Menjalankan `break_fw.sh` di **Magi-Casper** tanpa membaca isinya.
2. Gejala: Casper tidak bisa ping `8.8.8.8` dan tidak bisa `apt-get update`, Unit-00 tidak bisa tersambung ke TCP 5000 Casper. Ping ke Casper tetap ditolak (memang benar).
3. Memperbaiki **tanpa membatalkan tujuan hardening**: lalu lintas masuk yang tidak diizinkan tetap ditolak dan **policy INPUT tetap DROP**.
4. Bukti: `nc` dari Unit-00 ke Casper port 5000 berhasil dan `apt-get update` di Casper berhasil.

## 2. Menyiapkan dan Menjalankan Skrip

Agar isi skrip tidak terbaca, `break_fw.sh` diunduh langsung di Casper dari tautan soal, lalu hanya ukuran file dan dua karakter pertamanya (`#!`) yang diperiksa:

```bash
U="https://drive.usercontent.google.com/download"
curl -sL -o /root/break_fw.sh "$U?id=1Yb5kX1NKafOGDTbAE7WMYHZkQgpWLTvZ&confirm=t"
wc -c < /root/break_fw.sh; head -c 2 /root/break_fw.sh; echo
```

![Unduh break_fw.sh di Casper](<Assets/soal12_unduh script.png>)

Sebelum dijalankan, kondisi firewall dan rute disimpan sebagai pembanding:

```bash
iptables-save > /root/sebelum-fw.rules; ip route > /root/sebelum-fw.route
bash /root/break_fw.sh
```

![Menjalankan break_fw.sh](<Assets/soal12_jalankan break.png>)

## 3. Gejala Setelah Skrip Dijalankan

**Di Casper:** ping `8.8.8.8` gagal dan `apt-get update` tidak bisa menghubungi server Debian.

```bash
ping -c 2 -W 2 8.8.8.8
timeout 20 apt-get update; echo "exit=$?"
```

![Gejala di Casper](<Assets/soal12_gejala casper.png>)

**Di Unit-00:** koneksi ke Casper port 5000 menggantung sampai batas waktu 3 detik lalu gagal (`exit=1`), tanda paket dibuang (DROP).

```bash
time (echo "Unit-00 ke Casper" | nc -N -w 3 10.92.4.66 5000); echo "exit=$?"
```

![Gejala di Unit-00](<Assets/soal12_gejala unit00.png>)

## 4. Diagnosis

```bash
iptables -S
iptables -t nat -S; ip route; cat /etc/resolv.conf
```

![Aturan firewall setelah sabotase](<Assets/soal12_diagnosa iptables.png>)

```bash
iptables -t mangle -S; iptables -t raw -S
ps aux | grep -v grep | grep -E 'listener|nc -l'
diff /root/sebelum-fw.rules <(iptables-save) | grep '^[<>] -'
```

![Pemeriksaan tambahan](<Assets/soal12_diagnosa tambahan.png>)

Tabel `nat`, `mangle`, `raw`, rute, dan DNS normal. Listener port 5000 juga masih berjalan. Masalahnya ada di tabel `filter`:

| Temuan | Akibat |
|---|---|
| `-P INPUT DROP` hanya dengan `-A INPUT -i lo -j ACCEPT`, **tanpa aturan `ESTABLISHED,RELATED`** | Balasan untuk koneksi yang dimulai Casper sendiri (balasan ping, DNS, HTTP dari server Debian) ikut terbuang oleh policy DROP |
| `-A INPUT -p icmp -j DROP` (semua ICMP) | Selain `echo-request`, paket `echo-reply` untuk ping keluar Casper ikut dibuang |
| `-A OUTPUT -p udp --dport 53 -j DROP` | Query DNS keluar diblokir, sehingga `apt-get` tidak bisa me-resolve `deb.debian.org` |
| Tidak ada aturan `ACCEPT` untuk TCP 5000 | Koneksi Unit-00 jatuh ke policy DROP dan menggantung sampai timeout |

Aturan Soal 11 (`echo-request` DROP) juga hilang karena skrip mengosongkan chain `INPUT`.

## 5. Perbaikan

Bagian perbaikan ditambahkan ke `/root/magi-casper.sh`, lalu dijalankan:

```bash
cat >> /root/magi-casper.sh << 'EOF'

# Soal 12: perbaikan hardening, policy INPUT tetap DROP
iptables -P INPUT DROP
iptables -F INPUT
iptables -F OUTPUT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
iptables -A INPUT -p tcp --dport 5000 -j ACCEPT
EOF
```

```bash
sed -n '/Soal 12/,$p' /root/magi-casper.sh | bash
iptables -S
```

Penjelasan:

- **Policy INPUT tetap DROP**, sehingga semua lalu lintas masuk yang tidak disebut tetap ditolak (tujuan hardening terjaga).
- **`ESTABLISHED,RELATED`** mengizinkan balasan untuk koneksi yang dimulai Casper, sehingga ping keluar, DNS, dan `apt-get` kembali berjalan.
- **`echo-request` DROP** menggantikan aturan "semua ICMP" yang terlalu luas. Ping **ke** Casper tetap ditolak (Soal 11).
- **TCP 5000 ACCEPT** membuka layanan MAGI untuk klien.
- **`iptables -F OUTPUT`** menghapus blokir DNS keluar.

![Aturan setelah perbaikan](<Assets/soal12_perbaikan.png>)

## 6. Verifikasi

**Unit-00 ke Casper port 5000 berhasil (`exit=0`):**

```bash
echo "Unit-00 ke Casper" | nc -N -w 3 10.92.4.66 5000; echo "exit=$?"
```

![Unit-00 berhasil ke Casper 5000](<Assets/soal12_unit00 berhasil.png>)

**Casper bisa ping `8.8.8.8` dan `apt-get update` berhasil (`exit=0`):**

```bash
ping -c 2 -W 2 8.8.8.8
apt-get update; echo "exit=$?"
```

![Casper internet dan apt kembali normal](<Assets/soal12_casper internet.png>)

**Ping ke Casper tetap ditolak (hardening tetap berlaku):**

```bash
ping -c 2 -W 1 10.92.4.66
```

![Ping ke Casper tetap ditolak](<Assets/soal12_ping tetap ditolak.png>)
