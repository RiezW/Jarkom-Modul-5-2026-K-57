# Soal 16: Pencuri yang Mencoba Setiap Pintu

## 1. Yang Diminta

1. Setiap koneksi TCP dari **Dummy-Plug** (10.92.4.51) ke **port 22, 23, dan 3306** di Tokyo3-Core harus **dicatat** dengan prefix `SCAN-NERV:` lalu **ditolak seketika**.
2. Port 80 dan 8080 tetap bisa diakses, dan klien lain tidak terganggu.
3. Bukti: `nc -w` dari Dummy-Plug ke 22, 23, 3306 langsung ditolak, ke 80 dan 8080 berhasil, serta **counter rule pencatatan**.

## 2. Memasang Aturan

Bagian ini ditambahkan ke `/root/tokyo3-core.sh`:

```bash
cat >> /root/tokyo3-core.sh << 'EOF'

# Soal 16: catat dan tolak scan Dummy-Plug ke port 22, 23, 3306
iptables -N SCANNERV 2>/dev/null; iptables -F SCANNERV
S="-s 10.92.4.51 -p tcp -m multiport --dports 22,23,3306"
iptables -D INPUT $S -j SCANNERV 2>/dev/null
iptables -A INPUT $S -j SCANNERV
iptables -A SCANNERV -j LOG --log-prefix "SCAN-NERV: "
iptables -A SCANNERV -p tcp -j REJECT --reject-with tcp-reset
EOF
```

```bash
sed -n '/Soal 16/,$p' /root/tokyo3-core.sh | bash
iptables -S | grep -E 'INPUT|SCANNERV|BATASPING'
```

![Aturan Soal 16](<Assets/soal16_script tokyo3core.png>)

Penjelasan:

- Hanya koneksi TCP **dari Dummy-Plug** ke **22, 23, 3306** yang masuk ke chain `SCANNERV`. Port 80 dan 8080 serta klien lain tidak dicocokkan.
- **`LOG`** mencatat paket ke log kernel dengan prefix `SCAN-NERV: `. Target `LOG` tidak menghentikan paket, sehingga paket lanjut ke aturan berikutnya.
- **`REJECT --reject-with tcp-reset`** membalas dengan `RST`, sehingga koneksi langsung ditolak. Urutannya harus `LOG` dulu, baru `REJECT`.
- Karena log kernel sering tidak terlihat di dalam container, bukti pencatatan diambil dari counter aturan `LOG`.

## 3. Pengujian dari Dummy-Plug

**Port 22, 23, 3306 (ditolak seketika, sekitar 0.002 - 0.005 detik):**

```bash
for p in 22 23 3306; do time nc -v -w 3 10.92.4.106 $p </dev/null; done
```

![Scan Dummy-Plug ditolak](<Assets/soal16_scan ditolak.png>)

**Port 80 dan 8080 (berhasil):**

```bash
for p in 80 8080; do nc -w 3 10.92.4.106 $p </dev/null; echo "$p exit=$?"; done
```

![Port 80 dan 8080 tetap bisa diakses](<Assets/soal16_port 80 8080.png>)

## 4. Counter Aturan Pencatatan

Counter direset sebelum pengujian (`iptables -Z SCANNERV`). Setelah pengujian:

```bash
iptables -L SCANNERV -v -n
```

Aturan `LOG` dan `REJECT` masing-masing mencatat **3 paket**, satu untuk setiap port yang dicoba (22, 23, 3306).

![Counter SCANNERV](<Assets/soal16_counter scannerv.png>)
