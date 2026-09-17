# OpenVPN on Amazon Linux 2023

**Author:** Mureed Qasim Shah

Use the [angristan/openvpn-install](https://github.com/angristan/openvpn-install) script. It installs OpenVPN, generates the PKI, and writes client `.ovpn` files. Do not build Easy-RSA or `server.conf` by hand.

Auth is certificate-based (`.ovpn` profile). There is no SSO.

---

## Prerequisites

- EC2 — Amazon Linux 2023 (`t3.small` or larger) in a **public subnet** of the VPC
- Elastic IP on that instance
- SSH with sudo
- The servers you want to reach stay in a **private subnet** (no public IP)

VPN server security group (`openvpn-sg`) — inbound only:

| Protocol | Port | Source | Purpose |
| --- | --- | --- | --- |
| TCP | 22 | Your home/office IP `/32` | SSH to the VPN box itself |
| UDP | 1194 | `0.0.0.0/0` | OpenVPN clients |
| TCP | 1194 | `0.0.0.0/0` | OpenVPN TCP fallback |

Leave outbound on `openvpn-sg` as **All traffic**. The VPN box must be able to reach private instances.

Do **not** open SSH (`22`) to the world on private EC2. After you connect to the VPN, you SSH to the **private IP**. Rules for that are in [section 3](#3-reach-private-ec2-over-the-vpn).

---

## 1. Install OpenVPN

SSH into the instance as `ec2-user`.

### 1.1 Fix curl on Amazon Linux 2023

AL2023 ships `curl-minimal`, which conflicts with full `curl`. Swap it first:

```bash
sudo dnf swap curl-minimal curl --allowerasing -y
curl --version
```

You should see curl 8.x with OpenSSL.

### 1.2 Download and run the installer

```bash
curl -O https://raw.githubusercontent.com/angristan/openvpn-install/master/openvpn-install.sh
chmod +x openvpn-install.sh
sudo ./openvpn-install.sh install
```

That one command installs and starts OpenVPN with these defaults:

| Setting | Value |
| --- | --- |
| Endpoint | Your Elastic / public IP |
| Protocol | UDP |
| Port | 1194 |
| DNS | Cloudflare (`1.1.1.1`) |
| Auth | PKI (certificate) |
| VPN subnet | `10.8.0.0/24` |

### 1.3 Verify it is running

```bash
sudo systemctl status openvpn-server@server
sudo ss -ulnp | grep 1194
```

Status should be `active (running)`, and UDP 1194 should be listening.

---

## 2. Add a new client

Run the **same installer** again. Replace `alice` with the person’s name (letters, numbers, hyphens).

### Step 1 — Create the client

```bash
sudo ./openvpn-install.sh client add alice
```

The profile is written to:

```text
/home/ec2-user/alice.ovpn
```

### Step 2 — Copy it to their machine

From your laptop:

```bash
scp -i /path/to/your-key.pem \
  ec2-user@YOUR.ELASTIC.IP.HERE:alice.ovpn \
  ~/Downloads/alice.ovpn
```

Send that file over a private channel. Do not email it in the clear and do not commit it to git.

### Step 3 — Import and connect

**Windows**

1. Download and install [OpenVPN Connect](https://openvpn.net/client/) for Windows (the official OpenVPN client).
2. Open **OpenVPN Connect**.
3. Click the **+** button.
4. Choose **Upload File** / **Import** and select `alice.ovpn`.
5. Click the profile, then click **Connect**.

The tray icon should show connected. The client gets an address in `10.8.0.0/24`.

**Linux**

```bash
sudo openvpn --config alice.ovpn
```

**macOS / Android / iOS**

Install [OpenVPN Connect](https://openvpn.net/client/), import `alice.ovpn`, and connect.

### Step 4 — Confirm on the server

```bash
sudo ./openvpn-install.sh client list
sudo cat /var/log/openvpn/openvpn-status.log
```

---

## More clients, revoke, list

```bash
# another person
sudo ./openvpn-install.sh client add bob

# see everyone
sudo ./openvpn-install.sh client list

# remove access (lost laptop / offboarding)
sudo ./openvpn-install.sh client revoke alice
```

One `.ovpn` per person. Do not share the same file.

Optional: password-protect the client key:

```bash
sudo ./openvpn-install.sh client add alice --password
```

---

## 3. Reach private EC2 over the VPN

This is the reason the VPN exists: your laptop has no path into a private subnet until the tunnel is up. Traffic then goes:

```text
Your laptop  →  OpenVPN (public subnet)  →  private EC2 (private subnet)
                 UDP 1194                    SSH 22 to the private IP
```

The installer NATs VPN clients. The private instance does **not** see `10.8.0.x`. It sees the **OpenVPN instance private IP** (for example `10.0.1.20`). So on private EC2 you allow SSH from the VPN security group (or that private IP), not from `10.8.0.0/24`.

### Step 1 — Private EC2 security group (`private-ec2-sg`)

On each instance in the private subnet (or one SG attached to all of them):

| Protocol | Port | Source | Purpose |
| --- | --- | --- | --- |
| TCP | 22 | Security group `openvpn-sg` | SSH from people who are on the VPN |

Using the **security group id** of the VPN instance is better than a hardcoded IP. If the VPN box is replaced, the rule still works.

If you prefer an IP, allow TCP `22` from the OpenVPN instance **private** IPv4 (`/32`). Find it in the EC2 console or on the VPN server:

```bash
hostname -I
```

Add more rows the same way for other private services (for example TCP `443` for an internal HTTPS app, TCP `5432` for Postgres). Same source: `openvpn-sg`.

Remove any inbound SSH from `0.0.0.0/0` on these instances.

### Step 2 — Same VPC, no extra route for NAT

- OpenVPN instance and private EC2 must be in the **same VPC**.
- Default network ACLs (allow all) are enough.
- You do **not** need a route table entry for `10.8.0.0/24` while NAT is on. Return traffic goes back to the VPN instance private IP, which already has a local VPC route.

### Step 2b — Allow your VPC CIDR on the VPN box (required)

The installer NATs internet traffic, then **REJECT**s RFC1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) so the VPN does not “leak” into random private LANs. Your VPC is also `10.x`, so SSH to a private instance (for example `10.20.3.105`) is rejected **on the VPN server** before it ever reaches that EC2.

That shows up on your laptop as **connection refused**, not a timeout. Security groups are not the cause.

On the VPN instance, allow your VPC CIDR **before** those REJECT rules (this example is `10.20.0.0/16` — use your real VPC CIDR):

```bash
sudo iptables -I OPENVPN_INSTALL_FORWARD 1 -s 10.8.0.0/24 -d 10.20.0.0/16 -j ACCEPT
sudo iptables -t nat -I POSTROUTING 1 -s 10.8.0.0/24 -d 10.20.0.0/16 -j MASQUERADE
```

Confirm line 1 is ACCEPT to your VPC, and REJECT `10.0.0.0/8` sits **below** it:

```bash
sudo iptables -L OPENVPN_INSTALL_FORWARD -n -v --line-numbers
```

Add the same two `iptables` lines to `/etc/iptables/add-openvpn-rules.sh` or they disappear when OpenVPN restarts.

### Step 3 — Make sure the VPC CIDR is pushed to clients

The installer usually enables full tunnel (`redirect-gateway`) and NATs everything out the VPN ENI, which is enough.

If you can browse the internet through the VPN but **cannot** hit a private IP, push your VPC CIDR (change `10.0.0.0 255.255.0.0` to match your VPC, for example `10.0.0.0/16`):

```bash
sudo grep -E 'redirect-gateway|push "route' /etc/openvpn/server/server.conf
```

If your private subnet is missing, add it and restart:

```bash
echo 'push "route 10.0.0.0 255.255.0.0"' | sudo tee -a /etc/openvpn/server/server.conf
sudo systemctl restart openvpn-server@server
```

Reconnect the client after that change.

### Step 4 — Connect to VPN, then SSH to the private IP

1. Connect in OpenVPN Connect (section 2, step 3) and wait until it says connected.
2. In AWS, copy the target instance **Private IPv4** (not the public IP — private instances have none).
3. SSH to that address.

**Linux / macOS / Windows PowerShell** (after the VPN is up):

```bash
ssh -i /path/to/key.pem ec2-user@10.0.2.15
```

Use `ubuntu@` if that AMI is Ubuntu. Replace `10.0.2.15` with the real private IP.

On Windows you can also use PuTTY: Host Name = the private IP, port `22`, auth with your `.ppk` key.

You should **not** be able to SSH to that private IP when the VPN is disconnected. That is expected.

### Step 5 — Debug: `connection refused` vs timeout

| Symptom | Typical cause |
| --- | --- |
| Hang / timeout | Security group, NACL, or wrong IP — packet never arrives |
| `Connection refused` | Packet hit the VPN box (or host) and got **REJECT**. With this installer that is almost always the RFC1918 `REJECT` on `OPENVPN_INSTALL_FORWARD` |

Internet through the VPN can work (`curl ifconfig.me` shows the VPN Elastic IP) while SSH to `10.20.x.x` fails. Public internet is ACCEPTed; `10.0.0.0/8` is REJECTed.

**On your laptop** (VPN connected):

```bash
nc -vz 10.20.3.105 22
ssh -i servers.pem ec2-user@10.20.3.105
```

**On the VPN server**, watch whether the SYN arrives on `tun0` and whether it is forwarded:

```bash
sudo tcpdump -ni any host 10.20.3.105 and port 22
```

Then run `nc -vz 10.20.3.105 22` on the laptop again.

- SYN on `tun0` only (from `10.8.0.2` → `10.20.3.105:22`) and nothing out the ENI → iptables REJECT. That is this bug.
- SYN on `tun0` **and** on `eth0`/`ens5` → forwarding works; check the private instance SG and `sshd`.

List the chain (you should see REJECT `10.0.0.0/8` with packet counts when you retry SSH):

```bash
sudo iptables -L OPENVPN_INSTALL_FORWARD -n -v --line-numbers
```

Allow your VPC CIDR, then test again from the laptop:

```bash
sudo iptables -I OPENVPN_INSTALL_FORWARD 1 -s 10.8.0.0/24 -d 10.20.0.0/16 -j ACCEPT
sudo iptables -t nat -I POSTROUTING 1 -s 10.8.0.0/24 -d 10.20.0.0/16 -j MASQUERADE

sudo iptables -L OPENVPN_INSTALL_FORWARD -n -v --line-numbers
```

From the laptop:

```bash
nc -vz 10.20.3.105 22
```

`succeeded` means the path is open. Then SSH as usual.

Quick extra check: SSH **from the VPN instance itself** to `ec2-user@10.20.3.105`. If that works but the laptop still gets refused, it is definitely the `tun0` FORWARD rules, not `sshd`.

---

## Notes

- Keep `openvpn-install.sh` on the server. You will use it every time you add or revoke a client.
- Do not commit `.ovpn` files, keys, or certificates.
- Private instances stay private. The only inbound SSH they need is from the VPN security group.
