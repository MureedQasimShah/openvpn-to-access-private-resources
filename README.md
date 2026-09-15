# OpenVPN on Amazon Linux 2023

**Author:** Mureed Qasim Shah  
**Organization:** GoCompliance

Use the [angristan/openvpn-install](https://github.com/angristan/openvpn-install) script. It installs OpenVPN, generates the PKI, and writes client `.ovpn` files. Do not build Easy-RSA or `server.conf` by hand.

Auth is certificate-based (`.ovpn` profile). There is no SSO.

---

## Prerequisites

- EC2 — Amazon Linux 2023 (`t3.small` or larger)
- Elastic IP on the instance
- SSH with sudo
- DNS A record: `pritunl.gocompliance.com` → Elastic IP

**Security group**

| Protocol | Port | Source | Purpose |
| --- | --- | --- | --- |
| TCP | 22 | Your IP/32 | SSH |
| UDP | 1194 | 0.0.0.0/0 | OpenVPN |
| TCP | 1194 | 0.0.0.0/0 | OpenVPN TCP fallback |

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

## Notes

- Keep `openvpn-install.sh` on the server. You will use it every time you add or revoke a client.
- Do not commit `.ovpn` files, keys, or certificates.
