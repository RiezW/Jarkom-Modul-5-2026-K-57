# Soal 14: Jam Kunjungan Sang Pinguin

## 1. Yang Diminta

1. Ping dan TCP 80 ke **Pen-Pen** (10.92.3.195) hanya boleh sesuai jadwal, dari mana pun asalnya. Jam komputer tidak boleh diubah.
2. Jadwal disimpan sebagai variabel di awal skrip Pen-Pen: `PILOT_START`, `PILOT_STOP`, `PILOT_DAYS`, `STAF_START`, `STAF_STOP`.
3. Uji dengan jadwal yang **cocok** dengan jam sekarang lalu yang **tidak cocok**, dari Unit-01, Lilith, dan Spy.

| Kelompok | Anggota (IP) | Jadwal (WIB) |
|---|---|---|
| Pilot | Unit-01/02/00 (10.92.4.74 - .76), Sachiel (10.92.3.130) | 08.00 - 16.00, Senin - Jumat |
| Staf Geofront | Lilith (10.92.4.82), Adam (10.92.4.83) | 17.00 - 23.00, setiap hari |
| Terduga Dummy System | Entry-Plug (.4.50), Dummy-Plug (.4.51), Spy (.4.102) | Selalu ditolak |
| Lainnya | Semua node lain | Bebas |

## 2. Variabel Jadwal di Awal Skrip

Variabel disisipkan tepat setelah baris `#!/bin/sh` pada `/root/pen-pen.sh`:

```bash
cat > /tmp/jadwal << 'EOF'
# Soal 14: jadwal kunjungan Pen-Pen (jam WIB)
PILOT_START="08:00"
PILOT_STOP="16:00"
PILOT_DAYS="Mon,Tue,Wed,Thu,Fri"
STAF_START="17:00"
STAF_STOP="23:00"
EOF
sed -i '1r /tmp/jadwal' /root/pen-pen.sh
```

![Variabel jadwal di awal skrip](<Assets/soal14_variabel jadwal.png>)

## 3. Aturan Jadwal

Bagian aturan ditambahkan di akhir `/root/pen-pen.sh`:

```bash
# Soal 14: aturan jadwal (iptables memakai jam UTC = WIB - 7)
utc() { h=${1%%:*}; h=${h#0}; printf "%02d:%s" $(( (h + 17) % 24 )) "${1#*:}"; }
geser() {
  o=""; for d in $(echo "$1" | tr ',' ' '); do
    case $d in Mon) p=Sun;; Tue) p=Mon;; Wed) p=Tue;; Thu) p=Wed;;
      Fri) p=Thu;; Sat) p=Fri;; Sun) p=Sat;; esac; o="$o,$p"; done
  echo "${o#,}"
}
P1=$(utc $PILOT_START); P2=$(utc $PILOT_STOP); HARI=$PILOT_DAYS
[ "${PILOT_START%%:*}" -lt 7 ] && HARI=$(geser $PILOT_DAYS)
S1=$(utc $STAF_START); S2=$(utc $STAF_STOP)
iptables -N JADWAL 2>/dev/null; iptables -F JADWAL
for r in "-p icmp --icmp-type echo-request" "-p tcp --dport 80"; do
  iptables -D INPUT $r -j JADWAL 2>/dev/null; iptables -I INPUT $r -j JADWAL
done
for ip in 10.92.4.50 10.92.4.51 10.92.4.102; do
  iptables -A JADWAL -s $ip -j REJECT
done
for src in "-m iprange --src-range 10.92.4.74-10.92.4.76" "-s 10.92.3.130"; do
  iptables -A JADWAL $src -m time --timestart $P1 --timestop $P2 \
    --weekdays $HARI -j ACCEPT
  iptables -A JADWAL $src -j REJECT
done
STAF="-m iprange --src-range 10.92.4.82-10.92.4.83"
iptables -A JADWAL $STAF -m time --timestart $S1 --timestop $S2 -j ACCEPT
iptables -A JADWAL $STAF -j REJECT
```

![Bagian aturan jadwal](<Assets/soal14_aturan jadwal.png>)

Penjelasan:

- Hanya `echo-request` dan TCP port 80 yang masuk ke chain `JADWAL`. Port lain (misalnya port 9000 pada Soal 17) tidak terpengaruh.
- **Konversi jam:** match `time` membaca jam **UTC**. Fungsi `utc` mengurangi 7 jam dari WIB, misalnya 08.00 WIB menjadi 01:00 UTC.
- **Konversi hari:** `--weekdays` juga dibaca dalam UTC. Jika jadwal dimulai sebelum 07.00 WIB, hari dalam UTC adalah **hari sebelumnya** (contoh: Kamis 00.50 WIB = Rabu 17.50 UTC). Fungsi `geser` menggeser daftar hari mundur satu hari untuk kasus ini.
- Terduga Dummy System selalu ditolak. Pilot dan Staf diterima hanya di dalam jadwal, di luar jadwal ditolak. Node lain tidak dicocokkan oleh aturan mana pun sehingga kembali ke `INPUT` dan diterima.

## 4. Pengujian Kondisi Cocok

Jam saat pengujian: **Rabu 17.50 UTC = Kamis 00.50 WIB**.

```bash
date -u; TZ=Asia/Jakarta date
```

![Jam di Pen-Pen](<Assets/soal14_jam penpen.png>)

Variabel diubah agar mencakup jam sekarang (00.00 - 02.00 WIB untuk Pilot dan Staf), lalu skrip dijalankan:

```bash
sed -i -e 's/^PILOT_START=.*/PILOT_START="00:00"/' -e 's/^PILOT_STOP=.*/PILOT_STOP="02:00"/' /root/pen-pen.sh
sed -i -e 's/^STAF_START=.*/STAF_START="00:00"/' -e 's/^STAF_STOP=.*/STAF_STOP="02:00"/' /root/pen-pen.sh
bash /root/pen-pen.sh > /root/soal14.log 2>&1; iptables -S JADWAL
```

Aturan berubah menjadi `--timestart 17:00:00 --timestop 19:00:00` dengan hari Pilot yang sudah digeser (`Sun,Mon,Tue,Wed,Thu`).

![Aturan saat jadwal cocok](<Assets/soal14_cocok rule.png>)

Ping dari ketiga node:

```bash
ping -c 2 -W 1 10.92.3.195
```

| Node | Hasil |
|---|---|
| Unit-01 (Pilot) | Berhasil |
| Lilith (Staf) | Berhasil |
| Spy (Terduga) | Ditolak (`Destination Port Unreachable`) |

![Unit-01 saat cocok](<Assets/soal14_cocok unit01.png>)

![Lilith saat cocok](<Assets/soal14_cocok lilith.png>)

![Spy saat cocok](<Assets/soal14_cocok spy.png>)

## 5. Pengujian Kondisi Tidak Cocok

Variabel dikembalikan ke jadwal asli (Pilot 08.00 - 16.00, Staf 17.00 - 23.00 WIB) yang tidak mencakup 00.50 WIB:

```bash
sed -i -e 's/^PILOT_START=.*/PILOT_START="08:00"/' -e 's/^PILOT_STOP=.*/PILOT_STOP="16:00"/' /root/pen-pen.sh
sed -i -e 's/^STAF_START=.*/STAF_START="17:00"/' -e 's/^STAF_STOP=.*/STAF_STOP="23:00"/' /root/pen-pen.sh
bash /root/pen-pen.sh > /root/soal14.log 2>&1; iptables -S JADWAL
```

![Aturan saat jadwal tidak cocok](<Assets/soal14_tidak cocok rule.png>)

| Node | Hasil |
|---|---|
| Unit-01 (Pilot) | Ditolak |
| Lilith (Staf) | Ditolak |
| Spy (Terduga) | Ditolak |

![Unit-01 saat tidak cocok](<Assets/soal14_tidak cocok unit01.png>)

![Lilith saat tidak cocok](<Assets/soal14_tidak cocok lilith.png>)

![Spy saat tidak cocok](<Assets/soal14_tidak cocok spy.png>)

Skrip dibiarkan memakai jadwal asli.
