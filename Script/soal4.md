## 1. Memperbarui Skrip Gendou

Buka `/root/script.sh` di Gendou dengan perintah berikut:

```bash
nano script.sh
```
```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth1 eth2 eth3 eth4 eth5 eth6 eth7 eth8; do
    ip addr flush dev $i
    ip link set dev $i up
done

ip addr add 10.92.4.109/30 dev eth1
ip addr add 10.92.4.113/30 dev eth2
ip addr add 10.92.4.117/30 dev eth3
ip addr add 10.92.4.121/30 dev eth4
ip addr add 10.92.4.125/30 dev eth5
ip addr add 10.92.4.129/30 dev eth6
ip addr add 10.92.4.133/30 dev eth7
ip addr add 10.92.4.105/30 dev eth8

# Cabang Yui (10.92.4.110)
ip route replace 10.92.0.0/23 via 10.92.4.110
ip route replace 10.92.3.0/25 via 10.92.4.110
ip route replace 10.92.4.0/27 via 10.92.4.110
ip route replace 10.92.4.32/28 via 10.92.4.110
ip route replace 10.92.4.136/30 via 10.92.4.110
ip route replace 10.92.4.140/30 via 10.92.4.110
ip route replace 10.92.4.144/30 via 10.92.4.110

# Cabang Hikari (10.92.4.114)
ip route replace 10.92.2.0/24 via 10.92.4.114
ip route replace 10.92.3.224/27 via 10.92.4.114
ip route replace 10.92.4.148/30 via 10.92.4.114

# Cabang Atsuka (10.92.4.118) — Link Kensuke & Ritsuko diringkas ke /29
ip route replace 10.92.3.128/26 via 10.92.4.118
ip route replace 10.92.4.48/29 via 10.92.4.118
ip route replace 10.92.4.96/30 via 10.92.4.118
ip route replace 10.92.4.152/29 via 10.92.4.118

# Cabang Kaji (10.92.4.122)
ip route replace 10.92.4.100/30 via 10.92.4.122

# Cabang Misato (10.92.4.126)
ip route replace 10.92.4.72/29 via 10.92.4.126

# Cabang Shigeru (10.92.4.130)
ip route replace 10.92.4.56/29 via 10.92.4.130
ip route replace 10.92.4.88/29 via 10.92.4.130
ip route replace 10.92.4.160/30 via 10.92.4.130

# Cabang Fuyutsuki (10.92.4.134)
ip route replace 10.92.3.192/27 via 10.92.4.134
ip route replace 10.92.4.64/29 via 10.92.4.134
ip route replace 10.92.4.80/29 via 10.92.4.134
ip route replace 10.92.4.164/30 via 10.92.4.134
ip route replace 10.92.4.168/30 via 10.92.4.134

# NAT Masquerade hanya untuk prefix internal 10.92.0.0/16 keluar via eth0 (NAT1)
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -s 10.92.0.0/16 -o eth0 -j MASQUERADE
```
```bash
bash /root/script.sh
```

## 2. Mengatur DNS pada Armisael dan Lilith

Untuk pengujian Soal 4, jalankan perintah berikut di console Armisael dan Lilith:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

## 3. Menguji Koneksi

### Dari Armisael dan Lilith

Jalankan kedua perintah berikut di masing-masing console:

```bash
ping -c 3 8.8.8.8
ping -c 3 deb.debian.org
```
![alt text](<Assets/soal4_ping armisael.png>)
![alt text](<Assets/soal4_ping lilith.png>)

### Dari Gendou

Jalankan perintah berikut di console Gendou untuk melihat counter aturan NAT:

```bash
iptables -t nat -L POSTROUTING -v -n
```
![alt text](<Assets/soal4_counter rule nat.png>)