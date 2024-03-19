# VLAN и LACP


Yастройка выполняется с помощью Ansible (см. папку "Ansible").
### Конфигурация VLAN

`VLAN` конфигурируется через `netplan` файл:

```yaml
network:
    version: 2
    ethernets:
        enp0s3:
            dhcp4: true
        enp0s8: {}
    vlans:
        vlan2:
          id: 2
          link: enp0s8
          dhcp4: no
          addresses: [10.10.10.1/24]
```

Указывается `vlan id` и адрес интерфейса. После этого можно видеть следующий список устройств и параметров:
```bash
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:60:08:0f:26:bd brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 73734sec preferred_lft 73734sec
    inet6 fe80::60:8ff:fe0f:26bd/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:34:f6:29 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fe34:f629/64 scope link 
       valid_lft forever preferred_lft forever
5: vlan2@enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:34:f6:29 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global vlan2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe34:f629/64 scope link 
       valid_lft forever preferred_lft forever
```

**Проверка**

`ping` c `testClient2`:

```bash
vagrant@testClient2:~$ ping 10.10.10.1 | while read pong; do echo "$(date): $pong"; done
Mon Mar 18 09:30:17 PM UTC 2023: PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
Mon Mar 18 09:30:17 PM UTC 2023: 64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.972 ms
Mon Mar 18 09:30:18 PM UTC 2023: 64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.26 ms
Mon Mar 18 09:30:19 PM UTC 2023: 64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.42 ms

```

`tcpdump` на `testServer2`:

```bash
root@testServer2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v[v]... for full protocol Marode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:30:17.597738 IP 10.10.10.254 > testServer2: ICMP echo request, id 6382, seq 1, length 64
21:30:17.597805 IP testServer2 > 10.10.10.254: ICMP echo reply, id 6382, seq 1, length 64
21:30:18.598809 IP 10.10.10.254 > testServer2: ICMP echo request, id 6382, seq 2, length 64
21:30:18.598894 IP testServer2 > 10.10.10.254: ICMP echo reply, id 6382, seq 2, length 64
21:30:19.600928 IP 10.10.10.254 > testServer2: ICMP echo request, id 6382, seq 3, length 64
21:30:19.601001 IP testServer2 > 10.10.10.254: ICMP echo reply, id 6382, seq 3, length 64

```

На `testServer1` эти пакеты не доходят.

### LACP

Конфигурация интерфейсов на `inetRouter` и `centralRouter` для `netplan` выглядит следующим образом:
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      dhcp6: false
    enp0s9:
      dhcp4: false
      dhcp6: false
  bonds:
    bond0:
      addresses: [{{ bond_address }}]
      interfaces:
        - enp0s8
        - enp0s9
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
        fail-over-mac-policy: active
  version: 2
```

Здесь `bond` в режиме `active-backup` задействован.


**Проверка**

Будем пинговать `centralRouter` с `inetRouter` и отключать интерфейсы на `officeRouter`.

```bash
vagrant@inetRouter:~$ ping -i 4 192.168.255.2 | while read pong; do echo "$(date): $pong"; done
Mon Mar 18 09:38:48 PM UTC 2023: PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
Mon Mar 18 09:38:48 PM UTC 2023: 64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=1.19 ms
Mon Mar 18 09:38:52 PM UTC 2023: 64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=1.19 ms
# выключили enp0s9 на centralRouter (пинги еще идут)
Mon Mar 18 09:38:56 PM UTC 2023: 64 bytes from 192.168.255.2: icmp_seq=3 ttl=64 time=1.38 ms
# выключили enp0s8 на centralRouter (пинги не идут)
Mon Mar 18 09:39:32 PM UTC 2023: From 192.168.255.1 icmp_seq=11 Destination Host Unreachable
Mon Mar 18 09:39:36 PM UTC 2023: From 192.168.255.1 icmp_seq=12 Destination Host Unreachable
# подняли enp0s9 на centralRouter (пинги пошли)
Mon Mar 18 09:39:37 PM UTC 2023: 64 bytes from 192.168.255.2: icmp_seq=13 ttl=64 time=771 ms
Mon Mar 18 09:39:41 PM UTC 2023: 64 bytes from 192.168.255.2: icmp_seq=14 ttl=64 time=1.45 ms
```


```bash
root@centralRouter:~# ip link set enp0s9 down && date
Mon Mar 18 09:38:53 PM UTC 2023

root@centralRouter:~# ip link set enp0s8 down && date
Mon Mar 18 09:38:59 PM UTC 2023

root@centralRouter:~# ip link set enp0s9 up && date
Mon Mar 18 09:39:37 PM UTC 2023
```
