VPS Configuration (WireGuard + Port Forwarding)
1. Install WireGuard
On Ubuntu 22.04/24.04:
```
sudo apt update
sudo apt install wireguard ufw
```


2. Create WireGuard keys
```
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
```

3. Create /etc/wireguard/wg0.conf

Replace:

- <SERVER_PUBLIC_IP> with your VPS public IP
- <SERVER_PRIVATE_KEY> with contents of /etc/wireguard/server.key
- <CLIENT_PUBLIC_KEY> with the key Gluetun will generate
```
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

# Allow forwarding
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Home Gluetun client
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32
```

4. Enable IP forwarding

Edit /etc/sysctl.conf:
```
net.ipv4.ip_forward=1
```

Apply:
```
sudo sysctl -p
```

5. Configure UFW firewall
Allow SSH, WireGuard, and your torrent port (example: 50000):
```
sudo ufw default allow routed
sudo ufw allow 22/tcp
sudo ufw allow 51820/udp
sudo ufw allow 50000/tcp
sudo ufw allow 50000/udp
sudo ufw allow 53
sudo ufw enable
```


6. Forward the torrent port through WireGuard
Add to /etc/ufw/before.rules above the *filter line:
```
*nat
:PREROUTING ACCEPT [0:0]
-A PREROUTING -i eth0 -p tcp --dport 50000 -j DNAT --to-destination 10.8.0.2:50000
-A PREROUTING -i eth0 -p udp --dport 50000 -j DNAT --to-destination 10.8.0.2:50000
COMMIT
```

Reload:
```
sudo ufw reload
```


7. Start WireGuard
```
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```


Your VPS is now a full VPN server with a stable inbound port.
