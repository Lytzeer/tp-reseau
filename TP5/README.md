# VPN

Générer la clé ssh :
```
ssh-keygen -t rsa -b 4096
ssh-copy-id rocky@162.19.205.174
```

Création de l'utilisateur :
```
adduser matheo

passwd matheo

usermod -aG wheel matheo
```

Désactivation de la connexion en root
```
nano /etc/ssh/sshd_config

PermitRootLogin no
```

Redémarre ssh
```
sudo systemctl restart sshd
```

Installation des référentiels pour WIreGuard
```
sudo dnf install elrepo-release epel-release
```

Installation du serveur
```
sudo dnf install kmod-wireguard wireguard-tools
```

Création de la clé privé
```
wg genkey | sudo tee /etc/wireguard/private.key

sudo chmod go= /etc/wireguard/private.key
```

Création de la clé publique à partir de la clé privé
```
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key

Output
vf/zsG4fcGF6kyR91Mn67Py796XSq1FnietL9dkvwQA=
```

Configurer le serveur
```
sudo vi /etc/wireguard/wg0.conf

[Interface]
PrivateKey = base64_encoded_private_key_goes_here
Address = 172.22.2.12/24
ListenPort = 51820
SaveConfig = true
```

Configurer la connexion internet du serveur
```
sudo vi /etc/sysctl.conf

net.ipv4.ip_forward=1

sudo sysctl -p

Output
net.ipv4.ip_forward = 1
```

Installation firewall
```
sudo dnf install firewalld

sudo systemctl start firewalld

sudo firewall-cmd --add-port=22/tcp --permanent
Success

sudo firewall-cmd --reload
Success

sudo systemctl enable firewalld

sudo systemctl status firewalld
```

Configuration du firewall
```
sudo firewall-cmd --zone=public --add-port=51820/udp --permanent
success

sudo firewall-cmd --zone=internal --add-interface=wg0 --permanent
success

sudo firewall-cmd --zone=public --add-rich-rule='rule family=ipv4 source address=172.22.2.0/24 masquerade' --permanent
success

sudo firewall-cmd --reload
success

sudo firewall-cmd --zone=public --list-all

Output
[matheo@vps-35fb43a0 ~]$ sudo firewall-cmd --zone=public --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 22/tcp 51820/udp
  protocols: 
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
    rule family="ipv4" source address="172.22.2.0/24" masquerade

sudo firewall-cmd --zone=internal --list-interfaces

Output
wg0
```

Démarrer le serveur
```
sudo systemctl enable wg-quick@wg0.service

Output
Created symlink /etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service → /usr/lib/systemd/system/wg-quick@.service.

sudo systemctl start wg-quick@wg0.service

sudo systemctl status wg-quick@wg0.service

Output
● wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0
   Loaded: loaded (/usr/lib/systemd/system/wg-quick@.service; enabled; vendor preset: disabled)
   Active: active (exited) since Mon 2022-11-07 15:48:03 UTC; 6s ago
     Docs: man:wg-quick(8)
           man:wg(8)
           https://www.wireguard.com/
           https://www.wireguard.com/quickstart/
           https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
           https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8
  Process: 47358 ExecStart=/usr/bin/wg-quick up wg0 (code=exited, status=0/SUCCESS)
 Main PID: 47358 (code=exited, status=0/SUCCESS)

nov. 07 15:48:02 vps-35fb43a0.vps.ovh.net systemd[1]: Starting WireGuard via wg-quick(8) for wg0...
nov. 07 15:48:02 vps-35fb43a0.vps.ovh.net wg-quick[47358]: [#] ip link add wg0 type wireguard
nov. 07 15:48:02 vps-35fb43a0.vps.ovh.net wg-quick[47358]: [#] wg setconf wg0 /dev/fd/63
nov. 07 15:48:02 vps-35fb43a0.vps.ovh.net wg-quick[47358]: [#] ip -4 address add 172.22.2.12/24 dev wg0
nov. 07 15:48:03 vps-35fb43a0.vps.ovh.net wg-quick[47358]: [#] ip link set mtu 1420 up dev wg0
nov. 07 15:48:03 vps-35fb43a0.vps.ovh.net systemd[1]: Started WireGuard via wg-quick(8) for wg0.
```

Configurer le client
```
sudo dnf install elrepo-release epel-release

sudo dnf install kmod-wireguard wireguard-tools
```

Création clé privé client
```
wg genkey | sudo tee /etc/wireguard/private.key

sudo chmod go= /etc/wireguard/private.key
```

Création clé public à partir de la clé privé client
```
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key

TkyK8cASfC8hElyn2fi2nW4Lj+7+aRgglJaq0bURzSM=
```

Configuration de la connexion
```
sudo vi /etc/wireguard/wg0.conf

[Interface]
PrivateKey = (hidden)
Address = 172.22.2.2/24

[Peer]
PublicKey = vf/zsG4fcGF6kyR91Mn67Py796XSq1FnietL9dkvwQA=
AllowedIPs = 172.22.2.0/24
Endpoint = 162.19.205.174:51820
```

Ajout de la clé public client dans le serveur
```
sudo wg set wg0 peer TkyK8cASfC8hElyn2fi2nW4Lj+7+aRgglJaq0bURzSM= allowed-ips 172.22.2.2

sudo wg

Output
interface: wg0
  public key: vf/zsG4fcGF6kyR91Mn67Py796XSq1FnietL9dkvwQA=
  private key: (hidden)
  listening port: 51820

peer: TkyK8cASfC8hElyn2fi2nW4Lj+7+aRgglJaq0bURzSM=
  allowed ips: 172.22.2.2/32
```

Connexion client/serveur
```
sudo wg-quick up wg0

Output
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 172.22.2.2/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

Ping du serveur
```
ping -c 2 172.22.2.12

PING 172.22.2.12 (172.22.2.12) 56(84) octets de données.
64 octets de 172.22.2.12 : icmp_seq=1 ttl=64 temps=35.5 ms
64 octets de 172.22.2.12 : icmp_seq=2 ttl=64 temps=32.4 ms

--- statistiques ping 172.22.2.12 ---
2 paquets transmis, 2 reçus, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 32.381/33.928/35.475/1.547 ms
```

Ping 1.1.1.1
```
ping -c 4 1.1.1.1

PING 1.1.1.1 (1.1.1.1) 56(84) octets de données.
64 octets de 1.1.1.1 : icmp_seq=1 ttl=55 temps=22.6 ms
64 octets de 1.1.1.1 : icmp_seq=2 ttl=55 temps=23.5 ms
64 octets de 1.1.1.1 : icmp_seq=3 ttl=55 temps=24.1 ms
64 octets de 1.1.1.1 : icmp_seq=4 ttl=55 temps=21.6 ms

--- statistiques ping 1.1.1.1 ---
4 paquets transmis, 4 reçus, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 21.624/22.930/24.074/0.923 ms
```

Statut du tunnel
```
sudo wg

Output
interface: wg0
  public key: TkyK8cASfC8hElyn2fi2nW4Lj+7+aRgglJaq0bURzSM=
  private key: (hidden)
  listening port: 45315

peer: vf/zsG4fcGF6kyR91Mn67Py796XSq1FnietL9dkvwQA=
  endpoint: 162.19.205.174:51820
  allowed ips: 172.22.2.0/24
  latest handshake: 3 minutes, 41 seconds ago
  transfer: 696 B received, 904 B sent
```

Vérification si client utilise le VPN
```
sudo wg-quick down wg0

Output
[#] ip link delete dev wg0
```