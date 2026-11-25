# Boot Hook: Memulihkan Firewall dan Layanan Setelah Node Restart

## 1. Latar Belakang

Node Docker di GNS3 hanya menyimpan dua folder ketika di-stop lalu di-start lagi (`GNS3_VOLUMES=/etc/network:/root`). Semua yang hidup di memori hilang.

| Hal | Disimpan di | Setelah restart |
|---|---|---|
| IP, gateway, DNS (Soal 6) | `/etc/network/interfaces` | Kembali, karena init GNS3 menjalankan file ini saat boot, termasuk baris `up` |
| Skrip node (`/root/*.sh`) | `/root` | Kembali (file tetap ada) |
| Aturan `iptables` (Soal 11 - 16) | Memori kernel | **Hilang** |
| Listener MAGI (Soal 7) dan `web.sh` (Soal 8) | Proses | **Hilang** |

Node yang terdampak:

| Node | Yang perlu dipulihkan |
|---|---|
| Magi-Casper | Listener 5000 (Soal 7), aturan Soal 11 dan 12 (policy `INPUT DROP`) |
| Magi-Balthasar | Listener 5000 dan 5001 (Soal 7), chain `BRANKAS` (Soal 13) |
| Magi-Melchior | Listener 5000 (Soal 7) |
| Pen-Pen | `web.sh` port 80 (Soal 8), chain `JADWAL` (Soal 14) |
| Tokyo3-Core | `web.sh` port 80 dan 8080 (Soal 8), chain `BATASPING` dan `SCANNERV` (Soal 15, 16) |

Solusinya adalah **boot hook**: skrip `/root/boot.sh` yang dipanggil dari `/etc/network/interfaces`, sehingga ikut berjalan setiap node menyala.

## 2. Menyimpan Aturan Firewall

Setelah skrip node selesai dijalankan dan aturannya sudah benar, aturan disimpan ke file di `/root`:

```bash
iptables-save > /root/magi-casper.rules
```

File yang sama dibuat di setiap node: `magi-balthasar.rules`, `magi-melchior.rules` (kosong, karena Melchior tidak punya aturan), `pen-pen.rules`, dan `tokyo3-core.rules`.

Penjelasan:

- **`iptables-save`** mencetak seluruh isi tabel, termasuk chain buatan (`BRANKAS`, `JADWAL`, dan lainnya) dan **policy** chain. Policy `INPUT DROP` milik Casper (Soal 12) ikut tersimpan.
- Saat boot, aturan dipulihkan dengan **`iptables-restore`**, bukan dengan menjalankan ulang skrip node. Skrip node berisi `apt-get install` yang lambat dan butuh internet, sedangkan `iptables-restore` memasang seluruh tabel sekaligus dalam satu langkah.
- File `.rules` adalah **salinan pada satu waktu**. Jika aturan diubah (misalnya jadwal Soal 14), jalankan `iptables-save` lagi agar hasil boot sesuai.

## 3. Membuat `/root/boot.sh`

Contoh untuk **Magi-Casper**:

```bash
cat > /root/boot.sh << 'EOF'
#!/bin/sh
# Boot hook: pulihkan firewall dan layanan setelah node restart
N=magi-casper
[ -f /root/$N.rules ] && iptables-restore < /root/$N.rules
pkill -f listener.sh; pkill -f "nc -l -p"
nohup sh /root/listener.sh 5000 > /dev/null 2>&1 &
EOF
```

Perbedaan di node lain hanya pada nama `N` dan layanan yang dinyalakan:

| Node | `N` | Layanan |
|---|---|---|
| Magi-Casper | `magi-casper` | `listener.sh 5000` |
| Magi-Balthasar | `magi-balthasar` | `listener.sh 5000` dan `listener.sh 5001` |
| Magi-Melchior | `magi-melchior` | `listener.sh 5000` |
| Pen-Pen | `pen-pen` | `web.sh 80 "Welcome to $(hostname)"` |
| Tokyo3-Core | `tokyo3-core` | `web.sh 80 "Welcome to $(hostname)"` dan `web.sh 8080 "syifa imutz"` |

Pada Pen-Pen dan Tokyo3-Core, baris `pkill` menjadi `pkill -f web.sh; pkill -f "nc -l -p"`.

Penjelasan:

- **`[ -f ... ] &&`**: aturan hanya dipulihkan jika file `.rules` ada, sehingga boot tidak gagal ketika file belum dibuat.
- **`pkill`** sebelum `nohup`: skrip aman dijalankan berkali-kali tanpa membuat listener ganda.
- **`nohup ... &`**: layanan berjalan di latar belakang dan tidak ikut berhenti ketika proses boot selesai.

## 4. Memasang Hook di `/etc/network/interfaces`

Baris `up bash /root/boot.sh` disisipkan tepat setelah baris `up echo nameserver` dari Soal 6. Penyisipan dilakukan di **dua tempat**: file `interfaces` yang aktif dan heredoc Soal 6 di dalam skrip node. Tanpa yang kedua, menjalankan ulang skrip node akan menulis ulang `interfaces` dan menghapus hook.

```bash
H='/up echo nameserver/a\    up bash /root/boot.sh'
for f in /etc/network/interfaces /root/magi-casper.sh; do
  grep -q 'up bash /root/boot.sh' $f || sed -i "$H" $f
done
```

`grep -q ... ||` membuat baris hanya ditambahkan sekali walaupun perintah diulang. Pada node lain, `/root/magi-casper.sh` diganti dengan skrip node masing-masing.

## 5. Masalah: Layanan Web Tidak Hidup Setelah Boot

Pada uji restart pertama, aturan firewall dan listener MAGI kembali, tetapi **port 80 dan 8080 di Pen-Pen dan Tokyo3-Core tidak pernah `LISTEN`**. Proses `web.sh` terlihat hidup, tetapi tidak ada `nc` yang bertahan.

Diagnosis di Tokyo3-Core:

```bash
ls -l /proc/79/exe; grep -E 'State|NSpid' /proc/79/status
ps -eo pid,ppid,lstart,cmd | head -20
```

![Diagnosis web.sh saat boot](<Assets/boothook_diagnosis busybox.png>)

Temuan:

- **`/proc/79/exe -> /gns3/bin/busybox`**: `web.sh` yang dijalankan saat boot memakai `sh` milik **busybox GNS3**, bukan `dash` milik Debian.
- Init GNS3 menjalankan perintah boot dengan **`/gns3/bin` di depan `PATH`**. Akibatnya `sh` dan `nc` di dalam `boot.sh` mengarah ke applet busybox, bukan `/bin/sh` dan `/usr/bin/nc` (netcat-traditional).
- Proses `nc` (PID 2207 dan 2208) baru dibuat pada detik yang sama dengan `ps`, sedangkan loop `web.sh` sudah berjalan sejak boot. Artinya `nc` busybox langsung keluar karena opsinya tidak didukung, lalu loop mengulang terus tanpa pernah membuka port.
- Listener MAGI tidak terdampak karena hanya memakai `nc -l -p PORT`, yang juga didukung `nc` busybox.

Perbaikan: `PATH` sistem dipasang di awal `boot.sh` pada kelima node.

```bash
sed -i '2a # Pakai PATH sistem agar sh dan nc bukan applet busybox milik GNS3 (/gns3/bin)\nexport PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin' /root/boot.sh
```

Selama proses diagnosis, `web.sh` juga diubah dari `echo "$2" | nc -l -p $1 -q 1` menjadi `nc -l -p $1 -c "echo '$2'"` (di `/root/web.sh` dan bagian Soal 8 skrip node). Dengan `-c`, pesan dikirim oleh proses `echo` baru untuk setiap koneksi, sehingga tidak bergantung pada stdin dan timer `-q`. Perubahan ini tetap dipakai, dan [soal8.md](soal8.md) sudah disesuaikan.

```bash
sed -i 's/echo "$2" | nc -l -p $1 -q 1/nc -l -p $1 -c "echo \x27$2\x27"/' /root/web.sh /root/pen-pen.sh
```

Di Tokyo3-Core, `/root/pen-pen.sh` diganti `/root/tokyo3-core.sh`.

## 6. Pengujian Restart

Kelima node di-**Stop** dari peta GNS3 (klik kanan node, `Stop`), lalu dinyalakan lagi dengan tombol **Start all nodes**. Setelah boot, di setiap node dijalankan:

```bash
cat /root/boot.sh
grep -n 'up bash /root/boot.sh' /etc/network/interfaces /root/*.sh
echo "boot: $(ps -o lstart= -p 1)"
echo "rules: $(iptables-save | grep -c '^-A')"
ss -ltn | grep LISTEN
pgrep -af 'listener|web.sh|nc -l'
```

Baris `boot:` adalah waktu mulai PID 1 (init container), sebagai bukti node benar-benar baru menyala (18.31 WIB).

**Magi-Casper:**

![Boot hook Casper](<Assets/boothook_casper.png>)

**Magi-Balthasar:**

![Boot hook Balthasar](<Assets/boothook_balthasar.png>)

**Magi-Melchior:**

![Boot hook Melchior](<Assets/boothook_melchior.png>)

**Pen-Pen:**

![Boot hook Pen-Pen](<Assets/boothook_penpen.png>)

**Tokyo3-Core:**

![Boot hook Tokyo3-Core](<Assets/boothook_tokyo3core.png>)

| Node | Aturan `-A` | Port `LISTEN` | Keterangan |
|---|---|---|---|
| Magi-Casper | 4 | 5000 | Aturan Soal 11 dan 12 kembali |
| Magi-Balthasar | 5 | 5000, 5001 | Chain `BRANKAS` kembali |
| Magi-Melchior | 0 | 5000 | Tidak punya aturan firewall |
| Pen-Pen | 11 | 80 | Chain `JADWAL` kembali, `web.sh` memakai `/usr/bin/dash` |
| Tokyo3-Core | 6 | 80, 8080 | Chain `BATASPING` dan `SCANNERV` kembali |

Uji layanan dari Magi-Melchior (termasuk "Lainnya" pada Soal 14, sehingga tidak dibatasi jadwal Pen-Pen):

```bash
for t in '10.92.3.195 80' '10.92.4.106 80' '10.92.4.106 8080'; do
  echo "== $t"; nc -w 3 $t </dev/null
done
```

Ketiga port menjawab seperti pada Soal 8.

![Uji layanan setelah restart](<Assets/boothook_uji layanan melchior.png>)

## 7. Catatan

- Setiap kali aturan firewall di node berubah, perbarui salinannya: `iptables-save > /root/<node>.rules`.
- Router (Gendou, Fuyutsuki, dan lainnya) tidak memakai boot hook ini. Setelah router restart, skripnya dijalankan ulang secara manual (`bash /root/gendo.sh` untuk Gendou, `bash /root/script.sh` untuk router lain). Aturan DNAT Soal 17 berada di dalam `/root/script.sh` Fuyutsuki, sehingga ikut kembali saat skrip itu dijalankan.
