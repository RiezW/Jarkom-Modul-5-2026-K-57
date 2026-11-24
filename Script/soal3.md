# Soal 3: Update Script Routing

Simpan konfigurasi setiap router sebagai `script.sh`. Jalankan dengan `bash /root/script.sh`.

## 1. Gendou (Router Pusat)

Tambahkan rute statis menuju jaringan di belakang tujuh router cabang.

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth1 eth2 eth3 eth4 eth5 eth6 eth7 eth8; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.109/30 dev eth1
ip addr add 10.92.4.113/30 dev eth2
ip addr add 10.92.4.117/30 dev eth3
ip addr add 10.92.4.121/30 dev eth4
ip addr add 10.92.4.125/30 dev eth5
ip addr add 10.92.4.129/30 dev eth6
ip addr add 10.92.4.133/30 dev eth7
ip addr add 10.92.4.105/30 dev eth8

# Cabang Yui
ip route replace 10.92.0.0/23 via 10.92.4.110
ip route replace 10.92.3.0/25 via 10.92.4.110
ip route replace 10.92.4.0/27 via 10.92.4.110
ip route replace 10.92.4.32/28 via 10.92.4.110
ip route replace 10.92.4.136/30 via 10.92.4.110
ip route replace 10.92.4.140/30 via 10.92.4.110
ip route replace 10.92.4.144/30 via 10.92.4.110

# Cabang Hikari
ip route replace 10.92.2.0/24 via 10.92.4.114
ip route replace 10.92.3.224/27 via 10.92.4.114
ip route replace 10.92.4.148/30 via 10.92.4.114

# Cabang Atsuka
ip route replace 10.92.3.128/26 via 10.92.4.118
ip route replace 10.92.4.48/29 via 10.92.4.118
ip route replace 10.92.4.96/30 via 10.92.4.118
ip route replace 10.92.4.152/29 via 10.92.4.118

# Cabang Kaji
ip route replace 10.92.4.100/30 via 10.92.4.122

# Cabang Misato
ip route replace 10.92.4.72/29 via 10.92.4.126

# Cabang Shigeru
ip route replace 10.92.4.56/29 via 10.92.4.130
ip route replace 10.92.4.88/29 via 10.92.4.130
ip route replace 10.92.4.160/30 via 10.92.4.130

# Cabang Fuyutsuki
ip route replace 10.92.3.192/27 via 10.92.4.134
ip route replace 10.92.4.64/29 via 10.92.4.134
ip route replace 10.92.4.80/29 via 10.92.4.134
ip route replace 10.92.4.164/30 via 10.92.4.134
ip route replace 10.92.4.168/30 via 10.92.4.134
```

```bash
bash /root/script.sh
```

## 2. Router Cabang Menengah

### Yui

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1 eth2 eth3 eth4; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.110/30 dev eth0
ip addr add 10.92.3.1/25 dev eth1
ip addr add 10.92.4.137/30 dev eth2
ip addr add 10.92.4.141/30 dev eth3
ip addr add 10.92.4.145/30 dev eth4

ip route replace default via 10.92.4.109
ip route replace 10.92.4.32/28 via 10.92.4.138
ip route replace 10.92.0.0/23 via 10.92.4.142
ip route replace 10.92.4.0/27 via 10.92.4.146
```

```bash
bash /root/script.sh
```

### Hikari

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1 eth2; do
    ip addr flush dev $i
    ip link set dev $i up
done

ip addr add 10.92.4.114/30 dev eth0
ip addr add 10.92.4.149/30 dev eth1
ip addr add 10.92.2.1/24 dev eth2

ip route replace default via 10.92.4.113
ip route replace 10.92.3.224/27 via 10.92.4.150
```

```bash
bash /root/script.sh
```

### Atsuka

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1 eth2 eth3; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.118/30 dev eth0
ip addr add 10.92.4.153/30 dev eth1
ip addr add 10.92.4.157/30 dev eth2
ip addr add 10.92.4.97/30 dev eth3

ip route replace default via 10.92.4.117
ip route replace 10.92.3.128/26 via 10.92.4.154
ip route replace 10.92.4.48/29 via 10.92.4.158
```

```bash
bash /root/script.sh
```

### Shigeru

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1 eth2; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.130/30 dev eth0
ip addr add 10.92.4.161/30 dev eth1
ip addr add 10.92.4.89/29 dev eth2

ip route replace default via 10.92.4.129
ip route replace 10.92.4.56/29 via 10.92.4.162
```

```bash
bash /root/script.sh
```

### Fuyutsuki

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1 eth2 eth3; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.134/30 dev eth0
ip addr add 10.92.4.65/29 dev eth1
ip addr add 10.92.4.165/30 dev eth2
ip addr add 10.92.4.169/30 dev eth3

ip route replace default via 10.92.4.133
ip route replace 10.92.3.192/27 via 10.92.4.166
ip route replace 10.92.4.80/29 via 10.92.4.170
```

```bash
bash /root/script.sh
```

## 3. Router Leaf / Ujung

Tambahkan default route ke router induk sebagai baris terakhir pada skrip masing-masing router.

### Mari

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.138/30 dev eth0
ip addr add 10.92.4.33/28 dev eth1

ip route replace default via 10.92.4.137
```

```bash
bash /root/script.sh
```

### Rei

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.142/30 dev eth0
ip addr add 10.92.0.1/23 dev eth1

ip route replace default via 10.92.4.141
```

```bash
bash /root/script.sh
```

### Shinji

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.146/30 dev eth0
ip addr add 10.92.4.1/27 dev eth1

ip route replace default via 10.92.4.145
```

```bash
bash /root/script.sh
```

### Kaworu

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.150/30 dev eth0
ip addr add 10.92.3.225/27 dev eth1

ip route replace default via 10.92.4.149
```

```bash
bash /root/script.sh
```

### Kensuke

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.154/30 dev eth0
ip addr add 10.92.3.129/26 dev eth1

ip route replace default via 10.92.4.153
```

```bash
bash /root/script.sh
```

### Ritsuko

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.158/30 dev eth0
ip addr add 10.92.4.49/29 dev eth1

ip route replace default via 10.92.4.157
```

```bash
bash /root/script.sh
```

### Kaji

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.122/30 dev eth0
ip addr add 10.92.4.101/30 dev eth1

ip route replace default via 10.92.4.121
```

```bash
bash /root/script.sh
```

### Misato

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.126/30 dev eth0
ip addr add 10.92.4.73/29 dev eth1

ip route replace default via 10.92.4.125
```

```bash
bash /root/script.sh
```

### Makoto

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.162/30 dev eth0
ip addr add 10.92.4.57/29 dev eth1

ip route replace default via 10.92.4.161
```

```bash
bash /root/script.sh
```

### Maya

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.166/30 dev eth0
ip addr add 10.92.3.193/27 dev eth1

ip route replace default via 10.92.4.165
```

```bash
bash /root/script.sh
```

### Toji

```bash
nano script.sh
```

```bash
#!/bin/bash
sysctl -w net.ipv4.ip_forward=1

for i in eth0 eth1; do
    ip addr flush dev "$i"
    ip link set dev "$i" up
done

ip addr add 10.92.4.170/30 dev eth0
ip addr add 10.92.4.81/29 dev eth1

ip route replace default via 10.92.4.169
```

```bash
bash /root/script.sh
```



## 4. Konfigurasi IP Sementara pada Host

Jalankan konfigurasi berikut pada console masing-masing host.

### Armisael

```bash
ip link set eth0 up
ip addr add 10.92.4.91/29 dev eth0
ip route add default via 10.92.4.89
```

### Lilith

```bash
ip link set eth0 up
ip addr add 10.92.4.82/29 dev eth0
ip route add default via 10.92.4.81
```

### Sachiel

```bash
ip link set eth0 up
ip addr add 10.92.3.130/26 dev eth0
ip route add default via 10.92.3.129
```

### Comm-Net

```bash
ip link set eth0 up
ip addr add 10.92.2.2/24 dev eth0
ip route add default via 10.92.2.1
```

## 5. Pengujian Koneksi dari Armisael

Setelah IP sementara dikonfigurasi pada semua host, jalankan ping berikut dari console Armisael:

```bash
ping -c 3 10.92.4.82
ping -c 3 10.92.3.130
ping -c 3 10.92.2.2
```

![alt text](<Assets/soal3_ping armisael.png>)