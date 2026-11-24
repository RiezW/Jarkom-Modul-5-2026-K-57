# Soal 6: IP Statis dan Gateway Semua Node Non-Router

## 1. Yang Diminta

1. Memberi **IP statis dan gateway** ke semua node non-router (25 node) sesuai aturan alamat di glosarium.
2. Konfigurasi harus **bertahan setelah node direstart**.
3. Bukti: `ip -br a` dan ping ke gateway di **Unit-01**.

## 2. Aturan Pemberian Alamat

- Gateway adalah IP usable pertama subnet (sudah dipasang di router pada Soal 2).
- Node di switch mendapat IP usable berikutnya sesuai urutan nama node di gambar topologi, **dari kiri ke kanan**.
- Pada Switch-Medic, Medic-Bay berada di kiri Pen-Pen pada gambar, sehingga Medic-Bay mendapat `.194` dan Pen-Pen `.195`.

| Node | IP / Prefix | Netmask | Gateway |
|---|---|---|---|
| Quarters | 10.92.0.2/23 | 255.255.254.0 | 10.92.0.1 |
| Comm-Net | 10.92.2.2/24 | 255.255.255.0 | 10.92.2.1 |
| Hangar-Crew | 10.92.3.2/25 | 255.255.255.128 | 10.92.3.1 |
| Sachiel | 10.92.3.130/26 | 255.255.255.192 | 10.92.3.129 |
| Medic-Bay | 10.92.3.194/27 | 255.255.255.224 | 10.92.3.193 |
| Pen-Pen | 10.92.3.195/27 | 255.255.255.224 | 10.92.3.193 |
| Relay-Node | 10.92.3.226/27 | 255.255.255.224 | 10.92.3.225 |
| Garden-staff | 10.92.4.2/27 | 255.255.255.224 | 10.92.4.1 |
| Dock-Crew | 10.92.4.34/28 | 255.255.255.240 | 10.92.4.33 |
| Entry-Plug | 10.92.4.50/29 | 255.255.255.248 | 10.92.4.49 |
| Dummy-Plug | 10.92.4.51/29 | 255.255.255.248 | 10.92.4.49 |
| Radar-Array | 10.92.4.58/29 | 255.255.255.248 | 10.92.4.57 |
| Magi-Casper | 10.92.4.66/29 | 255.255.255.248 | 10.92.4.65 |
| Magi-Balthasar | 10.92.4.67/29 | 255.255.255.248 | 10.92.4.65 |
| Magi-Melchior | 10.92.4.68/29 | 255.255.255.248 | 10.92.4.65 |
| Unit-01 | 10.92.4.74/29 | 255.255.255.248 | 10.92.4.73 |
| Unit-02 | 10.92.4.75/29 | 255.255.255.248 | 10.92.4.73 |
| Unit-00 | 10.92.4.76/29 | 255.255.255.248 | 10.92.4.73 |
| Lilith | 10.92.4.82/29 | 255.255.255.248 | 10.92.4.81 |
| Adam | 10.92.4.83/29 | 255.255.255.248 | 10.92.4.81 |
| Zeruel (node `Zaruel`) | 10.92.4.90/29 | 255.255.255.248 | 10.92.4.89 |
| Armisael | 10.92.4.91/29 | 255.255.255.248 | 10.92.4.89 |
| Archive-Vault | 10.92.4.98/30 | 255.255.255.252 | 10.92.4.97 |
| Spy | 10.92.4.102/30 | 255.255.255.252 | 10.92.4.101 |
| Tokyo3-Core | 10.92.4.106/30 | 255.255.255.252 | 10.92.4.105 |

## 3. Catatan Sebelum Mulai

Setelah project GNS3 dinyalakan ulang, konfigurasi router hasil Soal 2 sampai 5 yang dibuat dengan perintah `ip` dan `iptables` hilang, karena perintah tersebut hanya berlaku selama container hidup. Router selain Gendou kembali normal karena `/etc/network/interfaces` miliknya menjalankan `up bash /root/script.sh` saat boot. Gendou belum punya baris tersebut, sehingga skripnya dijalankan ulang secara manual:

```bash
bash /root/gendo.sh
```

## 4. Membuat Skrip Konfigurasi pada Setiap Node

Setiap node mendapat skrip `/root/<nama-node>.sh` (sesuai aturan praktikum nomor 5). Skrip ini melakukan dua hal:

1. Menulis `/etc/network/interfaces`. File ini dibaca oleh init GNS3 setiap kali container dinyalakan, sehingga IP **bertahan setelah restart**.
2. Langsung memasang IP, gateway, dan DNS dengan perintah `ip` agar berlaku saat itu juga tanpa restart.

Contoh untuk **Unit-01**. Node lain sama, hanya nilai `IP`, `PFX`, `MASK`, dan `GW` yang diganti sesuai tabel di atas.

```bash
cat > /root/unit-01.sh << 'EOF'
#!/bin/sh
IP=10.92.4.74; PFX=29; MASK=255.255.255.248; GW=10.92.4.73
cat > /etc/network/interfaces << EON
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address $IP
    netmask $MASK
    gateway $GW
    up echo nameserver 8.8.8.8 > /etc/resolv.conf
EON
ip addr flush dev eth0
ip addr add $IP/$PFX dev eth0
ip link set eth0 up
ip route replace default via $GW
echo nameserver 8.8.8.8 > /etc/resolv.conf
EOF
```

```bash
sh /root/unit-01.sh
```

Penjelasan singkat:

- `ip addr flush dev eth0` menghapus IP sementara dari Soal 3 agar tidak dobel.
- `ip route replace` dipakai (bukan `add`) supaya skrip aman dijalankan berkali-kali.
- `nameserver 8.8.8.8` dipasang agar node bisa me-resolve nama domain saat memasang paket (Soal 7 dan 8).

![Isi skrip Unit-01](<Assets/soal6_script unit01.png>)

Daftar skrip yang dibuat:

| Node | Skrip |
|---|---|
| Quarters, Comm-Net, Hangar-Crew, Sachiel | `/root/quarters.sh`, `/root/comm-net.sh`, `/root/hangar-crew.sh`, `/root/sachiel.sh` |
| Medic-Bay, Pen-Pen, Relay-Node, Garden-staff, Dock-Crew | `/root/medic-bay.sh`, `/root/pen-pen.sh`, `/root/relay-node.sh`, `/root/garden-staff.sh`, `/root/dock-crew.sh` |
| Entry-Plug, Dummy-Plug, Radar-Array | `/root/entry-plug.sh`, `/root/dummy-plug.sh`, `/root/radar-array.sh` |
| Magi-Casper, Magi-Balthasar, Magi-Melchior | `/root/magi-casper.sh`, `/root/magi-balthasar.sh`, `/root/magi-melchior.sh` |
| Unit-01, Unit-02, Unit-00 | `/root/unit-01.sh`, `/root/unit-02.sh`, `/root/unit-00.sh` |
| Lilith, Adam, Zeruel, Armisael | `/root/lilith.sh`, `/root/adam.sh`, `/root/zeruel.sh`, `/root/armisael.sh` |
| Archive-Vault, Spy, Tokyo3-Core | `/root/archive-vault.sh`, `/root/spy.sh`, `/root/tokyo3-core.sh` |

## 5. Pengujian di Unit-01

Jalankan skrip, lalu periksa alamat, rute, dan ping ke gateway:

```bash
sh /root/unit-01.sh
ip -br a
ip route
ping -c 3 10.92.4.73
```

Hasil: `eth0` memiliki `10.92.4.74/29`, default route mengarah ke `10.92.4.73` (Misato), dan ping ke gateway 0% packet loss.

![IP dan ping gateway Unit-01](<Assets/soal6_ip dan ping unit01.png>)

Isi `/etc/network/interfaces` yang ditulis oleh skrip:

```bash
cat /etc/network/interfaces
```

![Isi /etc/network/interfaces Unit-01](<Assets/soal6_interfaces unit01.png>)

## 6. Pengujian Bertahan Setelah Restart

1. Pada topologi GNS3, klik kanan **Unit-01** lalu pilih **Stop**.

![Stop Unit-01](<Assets/soal6_stop unit01.png>)

2. Klik kanan **Unit-01** lagi lalu pilih **Start**.

![Start Unit-01](<Assets/soal6_start unit01.png>)

3. Buka console Unit-01 dan periksa kembali:

```bash
ps -o pid,etime,args -p 1
ip -br a show eth0
ping -c 3 10.92.4.73
```

`ps` menunjukkan proses init container baru berjalan beberapa detik (container benar-benar baru dinyalakan), tetapi `eth0` tetap memiliki `10.92.4.74/29` dan ping ke gateway tetap berhasil. Artinya konfigurasi dari `/etc/network/interfaces` dipasang ulang otomatis saat boot.

![Unit-01 setelah restart](<Assets/soal6_setelah restart unit01.png>)

## 7. Pengecekan Semua Node dari Gendou

Untuk memastikan ke-25 node sudah mendapat IP yang benar dan dapat dijangkau, ping semua IP node dari Gendou:

```bash
H="0.2 2.2 3.2 3.130 3.194 3.195 3.226 4.2 4.34 4.50 4.51 4.58 4.66"
H="$H 4.67 4.68 4.74 4.75 4.76 4.82 4.83 4.90 4.91 4.98 4.102 4.106"
for i in $H; do ping -c1 -W1 10.92.$i >/dev/null && echo "10.92.$i OK"; done
```

Hasil: ke-25 IP menjawab `OK`.

![Cek semua host dari Gendou](<Assets/soal6_cek semua host.png>)
