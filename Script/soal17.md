# Soal 17: Sihir Hitam di Jalur Paket

## 1. Yang Diminta

1. Di **Fuyutsuki** (gateway Switch-Terminal), alihkan setiap koneksi **TCP** dari Magi-Casper (10.92.4.66) ke Zeruel (10.92.4.90) agar sampai di **Pen-Pen** (10.92.3.195).
2. Casper tidak boleh sadar: balasan yang diterima harus tetap terlihat datang dari Zeruel.
3. **Ping tidak dialihkan.**
4. Bukti: pesan dari Casper ke IP Zeruel port 9000 muncul di **Pen-Pen, bukan di Zeruel**, serta tabel `conntrack` di Fuyutsuki.

## 2. Memasang DNAT di Fuyutsuki

Sesuai konvensi kelompok, skrip router memakai `/root/script.sh` karena soal tidak menyebut nama file. Bagian ini ditambahkan ke `/root/script.sh` Fuyutsuki:

```bash
cat >> /root/script.sh << 'EOF'

# Soal 17: alihkan TCP Casper -> Zeruel ke Pen-Pen
echo nameserver 8.8.8.8 > /etc/resolv.conf
command -v conntrack >/dev/null || { apt-get update; apt-get install -y conntrack; }
R="-s 10.92.4.66 -d 10.92.4.90 -p tcp -j DNAT --to-destination 10.92.3.195"
iptables -t nat -D PREROUTING $R 2>/dev/null
iptables -t nat -A PREROUTING $R
EOF
```

```bash
sed -n '/Soal 17/,$p' /root/script.sh | bash > /root/soal17.log 2>&1
iptables -t nat -S PREROUTING; which conntrack
```

![Aturan DNAT di Fuyutsuki](<Assets/soal17_script fuyutsuki.png>)

Penjelasan:

- **DNAT di `PREROUTING`** mengubah alamat tujuan paket pertama koneksi dari Zeruel menjadi Pen-Pen sebelum keputusan routing, sehingga Fuyutsuki meneruskannya ke arah Maya (Switch-Medic).
- Balasan dari Pen-Pen kembali melalui Fuyutsuki. **Connection tracking** mengembalikan alamat sumbernya menjadi Zeruel (10.92.4.90), sehingga Casper tidak sadar.
- **`-p tcp`** membuat hanya TCP yang dialihkan. Ping (ICMP) tetap sampai ke Zeruel.
- `conntrack` dipasang untuk menampilkan tabel koneksi sebagai bukti.

## 3. Menyalakan Listener Port 9000

**Pen-Pen** (netcat-traditional):

```bash
> /root/pesan-9000.txt; nc -l -p 9000 > /root/pesan-9000.txt &
```

**Zeruel** (netcat-openbsd, bawaan image):

```bash
> /root/pesan-9000.txt; nc -l 9000 > /root/pesan-9000.txt &
```

## 4. Mengirim Pesan dari Casper ke IP Zeruel

```bash
echo "Pesan Casper untuk Zeruel" | nc -q 1 -w 3 10.92.4.90 9000; echo "exit=$?"
```

![Casper mengirim ke IP Zeruel](<Assets/soal17_kirim casper.png>)

**Pesan muncul di Pen-Pen:**

```bash
cat /root/pesan-9000.txt
```

![Pesan diterima Pen-Pen](<Assets/soal17_pesan penpen.png>)

**Zeruel tidak menerima apa pun (file kosong):**

```bash
cat /root/pesan-9000.txt
```

![Zeruel tidak menerima pesan](<Assets/soal17_zeruel kosong.png>)

## 5. Tabel Conntrack di Fuyutsuki dan Bukti Ping Tidak Dialihkan

```bash
conntrack -L -p tcp --orig-port-dst 9000
conntrack -L -p icmp
```

- **TCP port 9000:** arah permintaan `src=10.92.4.66 dst=10.92.4.90` (alamat Zeruel yang diketik Casper), sedangkan arah balasan `src=10.92.3.195` (Pen-Pen yang sebenarnya menjawab). Selisih kedua arah inilah hasil DNAT.
- **ICMP:** arah balasan `src=10.92.4.90`, artinya ping dijawab Zeruel sendiri dan tidak dialihkan.

![Conntrack di Fuyutsuki](<Assets/soal17_conntrack fuyutsuki.png>)

Ping dari Casper ke Zeruel berhasil:

```bash
ping -c 2 10.92.4.90
```

![Ping Casper ke Zeruel tidak dialihkan](<Assets/soal17_ping tidak dialihkan.png>)
