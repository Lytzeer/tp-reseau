# TP3 : On va router des trucs

Au menu de ce TP, on va revoir un peu ARP et IP histoire de **se mettre en jambes dans un environnement avec des VMs**.

Puis on mettra en place **un routage simple, pour permettre à deux LANs de communiquer**.

![Reboot the router](./pics/reboot.jpeg)

## Sommaire

- [TP3 : On va router des trucs](#tp3--on-va-router-des-trucs)
  - [Sommaire](#sommaire)
  - [0. Prérequis](#0-prérequis)
  - [I. ARP](#i-arp)
    - [1. Echange ARP](#1-echange-arp)
    - [2. Analyse de trames](#2-analyse-de-trames)
  - [II. Routage](#ii-routage)
    - [1. Mise en place du routage](#1-mise-en-place-du-routage)
    - [2. Analyse de trames](#2-analyse-de-trames-1)
    - [3. Accès internet](#3-accès-internet)
  - [III. DHCP](#iii-dhcp)
    - [1. Mise en place du serveur DHCP](#1-mise-en-place-du-serveur-dhcp)
    - [2. Analyse de trames](#2-analyse-de-trames-2)

## 0. Prérequis

➜ Pour ce TP, on va se servir de VMs Rocky Linux. 1Go RAM c'est large large. Vous pouvez redescendre la mémoire vidéo aussi.  

➜ Vous aurez besoin de deux réseaux host-only dans VirtualBox :

- un premier réseau `10.3.1.0/24`
- le second `10.3.2.0/24`
- **vous devrez désactiver le DHCP de votre hyperviseur (VirtualBox) et définir les IPs de vos VMs de façon statique**

➜ Les firewalls de vos VMs doivent **toujours** être actifs (et donc correctement configurés).

➜ **Si vous voyez le p'tit pote 🦈 c'est qu'il y a un PCAP à produire et à mettre dans votre dépôt git de rendu.**

## I. ARP

Première partie simple, on va avoir besoin de 2 VMs.

| Machine  | `10.3.1.0/24` |
|----------|---------------|
| `john`   | `10.3.1.11`   |
| `marcel` | `10.3.1.12`   |

```schema
   john               marcel
  ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     │
  └─────┘    └───┘    └─────┘
```

> Référez-vous au [mémo Réseau Rocky](../../cours/memo/rocky_network.md) pour connaître les commandes nécessaire à la réalisation de cette partie.

### 1. Echange ARP

🌞**Générer des requêtes ARP**

- effectuer un `ping` d'une machine à l'autre

```
[root@localhost ~]# ping 10.3.1.12
PING 10.3.1.12 (10.3.1.12) 56(84) bytes of data.
64 bytes from 10.3.1.12: icmp_seq=1 ttl=64 time=0.269 ms
64 bytes from 10.3.1.12: icmp_seq=2 ttl=64 time=0.540 ms
64 bytes from 10.3.1.12: icmp_seq=3 ttl=64 time=0.358 ms
64 bytes from 10.3.1.12: icmp_seq=4 ttl=64 time=0.462 ms
64 bytes from 10.3.1.12: icmp_seq=5 ttl=64 time=0.311 ms
64 bytes from 10.3.1.12: icmp_seq=6 ttl=64 time=0.330 ms
64 bytes from 10.3.1.12: icmp_seq=7 ttl=64 time=0.317 ms
64 bytes from 10.3.1.12: icmp_seq=8 ttl=64 time=0.509 ms
64 bytes from 10.3.1.12: icmp_seq=9 ttl=64 time=0.679 ms
64 bytes from 10.3.1.12: icmp_seq=10 ttl=64 time=0.326 ms
64 bytes from 10.3.1.12: icmp_seq=11 ttl=64 time=0.284 ms
64 bytes from 10.3.1.12: icmp_seq=12 ttl=64 time=0.206 ms
64 bytes from 10.3.1.12: icmp_seq=13 ttl=64 time=0.520 ms
64 bytes from 10.3.1.12: icmp_seq=14 ttl=64 time=0.398 ms
64 bytes from 10.3.1.12: icmp_seq=15 ttl=64 time=0.489 ms

--- 10.3.1.12 ping statistics ---
15 packets transmitted, 15 received, 0% packet loss, time 14684ms
rtt min/avg/max/mdev = 0.206/0.399/0.679/0.124 ms
```
- observer les tables ARP des deux machines

>Marcel

```
[root@localhost ~]# ip neigh show
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:3a REACHABLE
10.3.1.11 dev enp0s8 lladdr 08:00:27:57:2a:fb STALE
```

>John

```
[root@localhost ~]# ip neigh show
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:3a REACHABLE
10.3.1.12 dev enp0s8 lladdr 08:00:27:63:45:99 STALE
```
- repérer l'adresse MAC de `john` dans la table ARP de `marcel` et vice-versa


>MAC de John
```
08:00:27:57:2a:fb
```
>MAC de Marcel
```
08:00:27:63:45:99
```
- prouvez que l'info est correcte (que l'adresse MAC que vous voyez dans la table est bien celle de la machine correspondante)
  - une commande pour voir la MAC de `marcel` dans la table ARP de `john`
```
[root@localhost ~]# ip neigh show 10.3.1.12
10.3.1.12 dev enp0s8 lladdr 08:00:27:63:45:99 STALE
```
  - et une commande pour afficher la MAC de `marcel`, depuis `marcel`
```
[root@localhost ~]# ip a
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 
    link/ether 08:00:27:63:45:99 brd ff:ff:ff:ff:ff:ff
```

### 2. Analyse de trames

🌞**Analyse de trames**

- utilisez la commande `tcpdump` pour réaliser une capture de trame
```
sudo tcpdump -i enp0s8 -w tp3_arp.pcapng not port 22
```
- videz vos tables ARP, sur les deux machines, puis effectuez un `ping`
```
sudo ip neigh flush all
```
```
ping 10.3.1.11
```

🦈 **Capture réseau `tp3_arp.pcapng`** qui contient un ARP request et un ARP reply

> **Si vous ne savez pas comment récupérer votre fichier `.pcapng`** sur votre hôte afin de l'ouvrir dans Wireshark, et me le livrer en rendu, demandez-moi.

## II. Routage

Vous aurez besoin de 3 VMs pour cette partie. **Réutilisez les deux VMs précédentes.**

| Machine  | `10.3.1.0/24` | `10.3.2.0/24` |
|----------|---------------|---------------|
| `router` | `10.3.1.254`  | `10.3.2.254`  |
| `john`   | `10.3.1.11`   | no            |
| `marcel` | no            | `10.3.2.12`   |

> Je les appelés `marcel` et `john` PASKON EN A MAR des noms nuls en réseau 🌻

```schema
   john                router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └───┘    └─────┘    └───┘    └─────┘
```

### 1. Mise en place du routage

🌞**Activer le routage sur le noeud `router`**

```
firewall-cmd --get-active-zone
public
  interfaces: enp0s8 enp0s9

firewall-cmd --add-masquerade --zone=public --permanent
success
```

🌞**Ajouter les routes statiques nécessaires pour que `john` et `marcel` puissent se `ping`**
>Jhon
```
[root@localhost ~]# ip route add 10.3.2.0/24 via 10.3.1.254 dev enp0s8

[root@localhost ~]# ip route show
10.3.2.0/24 via 10.3.1.254 dev enp0s8
```
>Marcel
```
[root@localhost ~]# ip route add 10.3.1.0/24 via 10.3.2.254 dev enp0s8

[root@localhost ~]ip route show
10.3.1.0/24 via 10.3.2.254 dev enp0s8
```
> Ping Marcel -> John
```
[root@localhost ~]# ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=63 time=0.645 ms
64 bytes from 10.3.1.11: icmp_seq=2 ttl=63 time=0.674 ms
64 bytes from 10.3.1.11: icmp_seq=3 ttl=63 time=0.467 ms
64 bytes from 10.3.1.11: icmp_seq=4 ttl=63 time=0.729 ms
64 bytes from 10.3.1.11: icmp_seq=5 ttl=63 time=0.555 ms
64 bytes from 10.3.1.11: icmp_seq=6 ttl=63 time=0.714 ms

--- 10.3.1.11 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 5127ms
rtt min/avg/max/mdev = 0.467/0.630/0.729/0.092 ms
```
> Ping John -> Marcel
```
[root@localhost ~]# ping 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=0.742 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=0.659 ms
64 bytes from 10.3.2.12: icmp_seq=3 ttl=63 time=0.656 ms
64 bytes from 10.3.2.12: icmp_seq=4 ttl=63 time=0.529 ms
64 bytes from 10.3.2.12: icmp_seq=5 ttl=63 time=0.712 ms
64 bytes from 10.3.2.12: icmp_seq=6 ttl=63 time=0.991 ms
64 bytes from 10.3.2.12: icmp_seq=7 ttl=63 time=0.642 ms
64 bytes from 10.3.2.12: icmp_seq=8 ttl=63 time=0.758 ms
64 bytes from 10.3.2.12: icmp_seq=9 ttl=63 time=0.834 ms
64 bytes from 10.3.2.12: icmp_seq=10 ttl=63 time=0.654 ms

--- 10.3.2.12 ping statistics ---
10 packets transmitted, 10 received, 0% packet loss, time 9460ms
rtt min/avg/max/mdev = 0.529/0.717/0.991/0.119 ms
```

![THE SIZE](./pics/thesize.png)

### 2. Analyse de trames

🌞**Analyse des échanges ARP**

- videz les tables ARP des trois noeuds
```
[root@localhost ~]# ip neigh flush all
```
- effectuez un `ping` de `john` vers `marcel`
```
[root@localhost ~]# ping 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=0.740 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=0.973 ms
64 bytes from 10.3.2.12: icmp_seq=3 ttl=63 time=0.496 ms
64 bytes from 10.3.2.12: icmp_seq=4 ttl=63 time=1.04 ms
64 bytes from 10.3.2.12: icmp_seq=5 ttl=63 time=0.973 ms

--- 10.3.2.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4159ms
rtt min/avg/max/mdev = 0.496/0.845/1.043/0.202 ms
```
- regardez les tables ARP des trois noeuds
>Table ARP John
```
[root@localhost ~]# ip neigh show
10.3.1.254 dev enp0s8 lladdr 08:00:27:ad:d5:3e STALE
10.3.1.1 dev enp0s8 lladdr 0a:00:27:00:00:0d DELAY
```
>Table ARP Router
```
[root@localhost ~]# ip neigh show
10.3.1.11 dev enp0s8 lladdr 08:00:27:57:2a:fb STALE
10.3.1.12 dev enp0s9 lladdr 0a:00:27:63:45:99 STALE
```
>Table ARP Marcel
```
[root@localhost ~]# ip neigh show
10.3.2.254 dev enp0s8 lladdr 08:00:27:3e:ba:36 STALE
10.3.2.1 dev enp0s8 lladdr 0a:00:27:00:00:3f DELAY
```
- répétez l'opération précédente (vider les tables, puis `ping`), en lançant `tcpdump` sur `marcel`
```
[root@localhost ~]# ip neigh flush all
```
```
[root@localhost ~]# tcpdump -i enp0s8 -w tp3_routage_marcel.pcapng not port 22
```
```
[root@localhost ~]# ping 10.3.2.12
```
- **écrivez, dans l'ordre, les échanges ARP qui ont eu lieu, puis le ping et le pong, je veux TOUTES les trames** utiles pour l'échange

Par exemple (copiez-collez ce tableau ce sera le plus simple) :

| ordre | type trame  | IP source | MAC source              | IP destination | MAC destination            |
|-------|-------------|-----------|-------------------------|----------------|----------------------------|
| 1     | Requête ARP | x         | `john` `08:00:27:8c:72:c6` | x              | Broadcast `FF:FF:FF:FF:FF` |
| 2     | Réponse ARP | x         | `marcel` `08:00:27:31:a0:e4`                       | x              | `john` `08:00:27:8c:72:c6`    |
| ...   | ...         | ...       | ...                     |                |                            |
| 3     | Ping        | 10.3.2.254         | `john` `08:00:27:8c:72:c6`                       | 10.3.2.12              | `marcel` `08:00:27:31:a0:e4`                          |
| 4     | Pong        | 10.3.2.12         | `marcel` `08:00:27:31:a0:e4`                       | 10.3.2.254              | `john` `08:00:27:8c:72:c6`                          |

> Vous pourriez, par curiosité, lancer la capture sur `john` aussi, pour voir l'échange qu'il a effectué de son côté.

🦈 **Capture réseau `tp3_routage_marcel.pcapng`**

### 3. Accès internet

🌞**Donnez un accès internet à vos machines**

- ajoutez une carte NAT en 3ème inteface sur le `router` pour qu'il ait un accès internet
- ajoutez une route par défaut à `john` et `marcel`
>John
```
ip route add default via 10.3.1.254 dev enp0s8
```
>Marcel
```
ip route add default via 10.3.2.254 dev enp0s8
```
  - vérifiez que vous avez accès internet avec un `ping`
  - le `ping` doit être vers une IP, PAS un nom de domaine
```
[root@localhost ~]# ping 142.251.5.101
PING 142.251.5.101 (142.251.5.101) 56(84) bytes of data.
64 bytes from 142.251.5.101: icmp_seq=1 ttl=110 time=19.0 ms
64 bytes from 142.251.5.101: icmp_seq=2 ttl=110 time=17.6 ms
64 bytes from 142.251.5.101: icmp_seq=3 ttl=110 time=17.3 ms
64 bytes from 142.251.5.101: icmp_seq=4 ttl=110 time=16.5 ms
64 bytes from 142.251.5.101: icmp_seq=5 ttl=110 time=45.0 ms
64 bytes from 142.251.5.101: icmp_seq=6 ttl=110 time=16.8 ms

--- 142.251.5.101 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 5011ms
rtt min/avg/max/mdev = 16.470/22.037/45.017/10.307 ms
```
- donnez leur aussi l'adresse d'un serveur DNS qu'ils peuvent utiliser
  - vérifiez que vous avez une résolution de noms qui fonctionne avec `dig`
```
[root@localhost ~]# dig gitlab.com

; <<>> DiG 9.16.23-RH <<>> gitlab.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32674
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;gitlab.com.                    IN      A

;; ANSWER SECTION:
gitlab.com.             300     IN      A       172.65.251.78

;; Query time: 40 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Tue Oct 11 21:54:26 CEST 2022
;; MSG SIZE  rcvd: 55
```
  - puis avec un `ping` vers un nom de domaine
```
[root@localhost ~]# ping gitlab.com
PING gitlab.com (172.65.251.78) 56(84) bytes of data.
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=1 ttl=56 time=13.6 ms
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=2 ttl=56 time=18.6 ms
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=3 ttl=56 time=17.0 ms
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=4 ttl=56 time=16.6 ms
64 bytes from 172.65.251.78 (172.65.251.78): icmp_seq=5 ttl=56 time=23.5 ms

--- gitlab.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4008ms
rtt min/avg/max/mdev = 13.620/17.859/23.516/3.249 ms
```

🌞**Analyse de trames**

- effectuez un `ping 8.8.8.8` depuis `john`
- capturez le ping depuis `john` avec `tcpdump`
```
[root@localhost ~]# tcpdump -i enp0s8 -w tp3_routage_internet.pcapng not port 22
```
- analysez un ping aller et le retour qui correspond et mettez dans un tableau :

| ordre | type trame | IP source          | MAC source              | IP destination | MAC destination |     |
|-------|------------|--------------------|-------------------------|----------------|-----------------|-----|
| 1     | ping       | `john` `10.3.1.12` | `john` `AA:BB:CC:DD:EE` | `8.8.8.8`      | ?               |     |
| 2     | pong       | ...                | ...                     | ...            | ...             | ... |

🦈 **Capture réseau `tp3_routage_internet.pcapng`**

## III. DHCP

On reprend la config précédente, et on ajoutera à la fin de cette partie une 4ème machine pour effectuer des tests.

| Machine  | `10.3.1.0/24`              | `10.3.2.0/24` |
|----------|----------------------------|---------------|
| `router` | `10.3.1.254`               | `10.3.2.254`  |
| `john`   | `10.3.1.11`                | no            |
| `bob`    | oui mais pas d'IP statique | no            |
| `marcel` | no                         | `10.3.2.12`   |

```schema
   john               router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └─┬─┘    └─────┘    └───┘    └─────┘
   john        │
  ┌─────┐      │
  │     │      │
  │     ├──────┘
  └─────┘
```

### 1. Mise en place du serveur DHCP

🌞**Sur la machine `john`, vous installerez et configurerez un serveur DHCP** (go Google "rocky linux dhcp server").

- installation du serveur sur `john`
```
default-lease-time 900;
max-lease-time 10800;
ddns-update-style none;
authoritative;
subnet 10.3.1.0 netmask 255.255.255.0 {
  range 10.3.1.15 10.3.1.100;
  option routers 10.3.1.254;
  option subnet-mask 255.255.255.0;
  option domain-name-servers 10.3.1.254;
}
```
```
[root@localhost ~]# firewall-cmd --permanent --add-port=67/udp
success
```
```
[root@localhost ~]# systemctl start dhcpd
```
```
[root@localhost ~]# systemctl enable dhcpd
Created symlink /etc/systemd/system/multi-user.target.wants/dhcpd.service → /usr/lib/systemd/system/dhcpd.service.
```
- créer une machine `bob`
- faites lui récupérer une IP en DHCP à l'aide de votre serveur
```
NAME=enp0s8
DEVICE=enp0s8

BOOTPROTO=dhcp
ONBOOT=yes
```
```
nmcli con reload
```
```
nmcli con up enp0s8
```
```
systemctl restart NetworkManager
```

> Il est possible d'utilise la commande `dhclient` pour forcer à la main, depuis la ligne de commande, la demande d'une IP en DHCP, ou renouveler complètement l'échange DHCP (voir `dhclient -h` puis call me et/ou Google si besoin d'aide).

🌞**Améliorer la configuration du DHCP**

- ajoutez de la configuration à votre DHCP pour qu'il donne aux clients, en plus de leur IP :
  - une route par défaut
```
ip route add default via 10.3.1.254 dev enp0s8
```
  - un serveur DNS à utiliser
```
default-lease-time 900;
max-lease-time 10800;
ddns-update-style none;
authoritative;
subnet 10.3.1.0 netmask 255.255.255.0 {
  range 10.3.1.15 10.3.1.100;
  option routers 10.3.1.254;
  option subnet-mask 255.255.255.0;
  option domain-name-servers 8.8.8.8;
}
```
- récupérez de nouveau une IP en DHCP sur `bob` pour tester :
  - `marcel` doit avoir une IP
    - vérifier avec une commande qu'il a récupéré son IP
```
[root@localhost ~]# ip a
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:40:38:9f brd ff:ff:ff:ff:ff:ff
    inet 10.3.1.16/24 brd 10.3.1.255 scope global dynamic noprefixroute enp0s8
       valid_lft 808sec preferred_lft 808sec
    inet6 fe80::a00:27ff:fe40:389f/64 scope link
       valid_lft forever preferred_lft forever
```
    - vérifier qu'il peut `ping` sa passerelle
```
[root@localhost ~]# ping 10.3.1.254
PING 10.3.1.254 (10.3.1.254) 56(84) bytes of data.
64 bytes from 10.3.1.254: icmp_seq=1 ttl=64 time=0.432 ms
64 bytes from 10.3.1.254: icmp_seq=2 ttl=64 time=0.539 ms
64 bytes from 10.3.1.254: icmp_seq=3 ttl=64 time=0.506 ms
64 bytes from 10.3.1.254: icmp_seq=4 ttl=64 time=0.545 ms
64 bytes from 10.3.1.254: icmp_seq=5 ttl=64 time=0.460 ms

--- 10.3.1.254 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4360ms
rtt min/avg/max/mdev = 0.432/0.496/0.545/0.044 ms
```
  - il doit avoir une route par défaut
    - vérifier la présence de la route avec une commande
```
[root@localhost ~]# ip route show
default via 10.3.1.254 dev enp0s8 proto dhcp src 10.3.1.16 metric 100
```
    - vérifier que la route fonctionne avec un `ping` vers une IP
```
[root@localhost ~]# ping 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=0.568 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=0.917 ms
64 bytes from 10.3.2.12: icmp_seq=3 ttl=63 time=0.714 ms
64 bytes from 10.3.2.12: icmp_seq=4 ttl=63 time=0.871 ms
64 bytes from 10.3.2.12: icmp_seq=5 ttl=63 time=1.35 ms

--- 10.3.2.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4203ms
rtt min/avg/max/mdev = 0.568/0.884/1.353/0.264 ms
```
  - il doit connaître l'adresse d'un serveur DNS pour avoir de la résolution de noms
    - vérifier avec la commande `dig` que ça fonctionne
```
[root@localhost ~]# dig google.com

; <<>> DiG 9.16.23-RH <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17312
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             54      IN      A       216.58.209.238

;; Query time: 28 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Thu Oct 13 15:30:42 CEST 2022
;; MSG SIZE  rcvd: 55
```
    - vérifier un `ping` vers un nom de domaine
```
[root@localhost ~]# ping google.com
PING google.com (216.58.209.238) 56(84) bytes of data.
64 bytes from par10s29-in-f238.1e100.net (216.58.209.238): icmp_seq=1 ttl=247 time=22.7 ms
64 bytes from par10s29-in-f238.1e100.net (216.58.209.238): icmp_seq=2 ttl=247 time=24.8 ms
64 bytes from par10s29-in-f14.1e100.net (216.58.209.238): icmp_seq=3 ttl=247 time=24.0 ms
64 bytes from par10s29-in-f14.1e100.net (216.58.209.238): icmp_seq=4 ttl=247 time=28.6 ms
64 bytes from par10s29-in-f238.1e100.net (216.58.209.238): icmp_seq=5 ttl=247 time=41.9 ms

--- google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4087ms
rtt min/avg/max/mdev = 22.650/28.392/41.852/7.013 ms
```

### 2. Analyse de trames

🌞**Analyse de trames**

- lancer une capture à l'aide de `tcpdump` afin de capturer un échange DHCP
```
tcpdump -i enp0s8  -w tp3_dhcp.pcap not port 22
```
- demander une nouvelle IP afin de générer un échange DHCP
```
dhclient
```
- exportez le fichier `.pcapng`

🦈 **Capture réseau `tp3_dhcp.pcapng`**