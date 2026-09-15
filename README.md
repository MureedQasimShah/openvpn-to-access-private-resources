# OpenVPN on Amazon Linux

Step-by-step guide to install an **OpenVPN** server on **Amazon Linux 2023** (EC2) and add a new client.

This setup uses Easy-RSA for certificates, UDP port **1194**, and a VPN subnet of `10.8.0.0/24`.

> Do not commit CA keys, server keys, or `.ovpn` files. Keep them only on the server and copy client profiles over a private channel (SCP).

---

## Prerequisites

On AWS, launch an Amazon Linux 2023 instance (a `t3.micro` is enough for a small team).

1. Put the instance in a **public subnet** with a public IPv4 address.
2. Attach an **Elastic IP** so the VPN hostname/IP does not change.
3. Security group inbound rules:
   - SSH: TCP `22` from your IP
   - OpenVPN: **UDP `1194`** from `0.0.0.0/0` (or lock it down to known client IPs)
4. Disable source/destination check (required so the instance can NAT VPN traffic):

```bash
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --no-source-dest-check
```

SSH in as `ec2-user` before continuing.

---

## Part 1 — Install OpenVPN on Amazon Linux

### 1. Install packages

**Amazon Linux 2023:**

```bash
sudo dnf update -y
sudo dnf install -y openvpn iptables-services curl openssl tar wget
```

**Amazon Linux 2** (older AMI; prefer AL2023):

```bash
sudo yum update -y
sudo amazon-linux-extras install epel -y
sudo yum install -y openvpn iptables-services curl openssl tar wget
```

### 2. Install Easy-RSA

```bash
cd /tmp
curl -LO https://github.com/OpenVPN/easy-rsa/releases/download/v3.2.6/EasyRSA-3.2.6.tgz
sudo mkdir -p /etc/openvpn/easy-rsa
sudo tar xzf EasyRSA-3.2.6.tgz -C /etc/openvpn/easy-rsa --strip-components=1
sudo chown -R root:root /etc/openvpn/easy-rsa
```

### 3. Create the Certificate Authority and server certs

```bash
cd /etc/openvpn/easy-rsa

sudo ./easyrsa init-pki
sudo ./easyrsa --batch build-ca nopass
sudo ./easyrsa --batch build-server-full server nopass
sudo ./easyrsa gen-dh
sudo ./easyrsa gen-crl
sudo openvpn --genkey secret /etc/openvpn/easy-rsa/pki/tc.key
```

Copy the files OpenVPN will actually load (do not point the service at `/home`; systemd blocks that):

```bash
sudo mkdir -p /etc/openvpn/server /var/log/openvpn

sudo cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/issued/server.crt /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/private/server.key /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/tc.key /etc/openvpn/server/
sudo cp /etc/openvpn/easy-rsa/pki/crl.pem /etc/openvpn/server/

sudo chmod 600 /etc/openvpn/server/server.key /etc/openvpn/server/tc.key
sudo chmod 644 /etc/openvpn/server/crl.pem
```

### 4. Write the server config

```bash
sudo tee /etc/openvpn/server/server.conf > /dev/null << 'EOF'
port 1194
proto udp
dev tun

ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem
tls-crypt /etc/openvpn/server/tc.key
crl-verify /etc/openvpn/server/crl.pem

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /etc/openvpn/server/ipp.txt

# Send all client internet traffic through the VPN. Remove this line
# if you only want access to AWS/VPC, not a full tunnel.
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 8.8.8.8"

# Push your VPC CIDR so clients can reach private AWS resources.
# Change this to match your VPC (example: 10.0.0.0/16).
push "route 10.0.0.0 255.255.0.0"

keepalive 10 120
data-ciphers AES-256-GCM
auth SHA256
user nobody
group nobody
persist-key
persist-tun
topology subnet

status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3
explicit-exit-notify 1
EOF
```

If `group nobody` fails on your AMI, check groups with `getent group nobody openvpn` and change `group` to the one that exists.

### 5. Enable IP forwarding and NAT

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-openvpn.conf
sudo sysctl --system

PRIMARY_IF=$(ip route show default | awk '{print $5; exit}')
sudo iptables -t nat -C POSTROUTING -s 10.8.0.0/24 -o "$PRIMARY_IF" -j MASQUERADE 2>/dev/null \
  || sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o "$PRIMARY_IF" -j MASQUERADE

sudo iptables-save | sudo tee /etc/sysconfig/iptables
sudo systemctl enable --now iptables
```

If you also need return traffic from other instances in the VPC to VPN clients (`10.8.0.0/24`), add a route in the VPC route table pointing `10.8.0.0/24` at this instance.

### 6. Start OpenVPN

Amazon Linux 2023:

```bash
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server --no-pager
```

Amazon Linux 2 (if the unit above is missing):

```bash
sudo cp /etc/openvpn/server/server.conf /etc/openvpn/server.conf
sudo systemctl enable --now openvpn@server
sudo systemctl status openvpn@server --no-pager
```

Confirm it is listening:

```bash
sudo ss -ulnp | grep 1194
sudo tail -n 50 /var/log/openvpn/openvpn.log
```

You should see `Initialization Sequence Completed`.

---

## Part 2 — Add a new client (step by step)

Do this **on the OpenVPN server** for every new user or device. Each client must have its own certificate.

Replace `alice` with a short name: letters, numbers, and hyphens only (for example `alice-laptop`).

### Step 1 — Create the client certificate

```bash
cd /etc/openvpn/easy-rsa
sudo ./easyrsa --batch build-client-full alice nopass
```

That writes:

- `/etc/openvpn/easy-rsa/pki/issued/alice.crt`
- `/etc/openvpn/easy-rsa/pki/private/alice.key`

### Step 2 — Set the public IP clients will connect to

```bash
# Elastic IP or the instance public IPv4
VPN_SERVER_IP="YOUR.ELASTIC.IP.HERE"
CLIENT_NAME="alice"
```

### Step 3 — Build a single `.ovpn` profile

```bash
sudo mkdir -p /etc/openvpn/client-configs

sudo tee /etc/openvpn/client-configs/${CLIENT_NAME}.ovpn > /dev/null << EOF
client
dev tun
proto udp
remote ${VPN_SERVER_IP} 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
data-ciphers AES-256-GCM
auth SHA256
verb 3

<ca>
$(sudo cat /etc/openvpn/easy-rsa/pki/ca.crt)
</ca>

<cert>
$(sudo cat /etc/openvpn/easy-rsa/pki/issued/${CLIENT_NAME}.crt)
</cert>

<key>
$(sudo cat /etc/openvpn/easy-rsa/pki/private/${CLIENT_NAME}.key)
</key>

<tls-crypt>
$(sudo cat /etc/openvpn/easy-rsa/pki/tc.key)
</tls-crypt>
EOF

sudo chmod 600 /etc/openvpn/client-configs/${CLIENT_NAME}.ovpn
ls -l /etc/openvpn/client-configs/${CLIENT_NAME}.ovpn
```

### Step 4 — Copy the profile to the client machine

From **your laptop** (not the server):

```bash
scp -i /path/to/your-key.pem \
  ec2-user@YOUR.ELASTIC.IP.HERE:/etc/openvpn/client-configs/alice.ovpn \
  ~/Downloads/alice.ovpn
```

If `scp` is denied because the file is root-owned, copy it to the home directory first on the server:

```bash
sudo cp /etc/openvpn/client-configs/alice.ovpn /home/ec2-user/alice.ovpn
sudo chown ec2-user:ec2-user /home/ec2-user/alice.ovpn
chmod 600 /home/ec2-user/alice.ovpn
```

Then download `/home/ec2-user/alice.ovpn` and delete it from the server after transfer:

```bash
rm /home/ec2-user/alice.ovpn
```

### Step 5 — Import and connect

| Platform | Client |
| --- | --- |
| Windows / macOS | [OpenVPN Connect](https://openvpn.net/client/) |
| Linux | `sudo dnf install -y openvpn` then `sudo openvpn --config alice.ovpn` |
| Android / iOS | OpenVPN Connect from the store |

Import `alice.ovpn` and connect. The client should get an address in `10.8.0.0/24`.

### Step 6 — Verify on the server

```bash
sudo cat /var/log/openvpn/openvpn-status.log
```

Look for the client name under `CLIENT_LIST`.

---

## Add more clients later

Repeat Part 2 with a new name:

```bash
CLIENT_NAME="bob"
VPN_SERVER_IP="YOUR.ELASTIC.IP.HERE"

cd /etc/openvpn/easy-rsa
sudo ./easyrsa --batch build-client-full "$CLIENT_NAME" nopass
```

Then run the same `tee ... ${CLIENT_NAME}.ovpn` block from Step 3 and copy the new file to that person.

Never reuse one `.ovpn` file for two people. If a laptop is lost, revoke that one certificate (below) instead of rebuilding the whole server.

---

## Revoke a client

```bash
cd /etc/openvpn/easy-rsa
sudo ./easyrsa --batch revoke alice
sudo ./easyrsa gen-crl
sudo cp /etc/openvpn/easy-rsa/pki/crl.pem /etc/openvpn/server/crl.pem
sudo chmod 644 /etc/openvpn/server/crl.pem
sudo systemctl restart openvpn-server@server
```

That client can no longer connect. Other clients are unchanged.

---

## Quick troubleshooting

| Symptom | Check |
| --- | --- |
| Service will not start | `sudo journalctl -u openvpn-server@server -e --no-pager` |
| Client times out | Security group **UDP 1194**, instance public IP / Elastic IP, `ss -ulnp \| grep 1194` |
| Connects but no internet | `cat /proc/sys/net/ipv4/ip_forward` must be `1`; NAT rule on the default interface; source/destination check **disabled** |
| Connects but cannot reach VPC | `push "route ..."` CIDR matches the VPC; route table has `10.8.0.0/24` → this instance |
| Permission errors on certs | Keys live under `/etc/openvpn/server/`, mode `600` on `server.key` |

---

## Security notes

- Treat `/etc/openvpn/easy-rsa/pki/` as secret. A leaked CA key can mint new clients.
- Prefer locking UDP `1194` to known client IPs when you can.
- Use one certificate per person/device and revoke on offboarding.
- Keep Amazon Linux patched: `sudo dnf update -y`.
