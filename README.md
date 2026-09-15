# OpenVPN on Amazon Linux 2023

**Author:** Mureed Qasim Shah  
**Organization:** GoCompliance  
**Plugin:** [openvpn-auth-oauth2](https://github.com/jkroepke/openvpn-auth-oauth2) v1.27.4+  
**Identity provider:** Google OAuth2 / OIDC

Use the [angristan/openvpn-install](https://github.com/angristan/openvpn-install) script. It installs OpenVPN, generates the PKI, and writes client `.ovpn` files. Do not build Easy-RSA or `server.conf` by hand.

---

## Security advisory — CVE-2026-41070

**CVE-2026-41070** is **CRITICAL (CVSS 9.5)**. It affects `openvpn-auth-oauth2` **via-env** script mode (`auth-user-pass-verify via-env` with `script-security 3`).

| Detail | Value |
| --- | --- |
| CVE ID | CVE-2026-41070 |
| CVSS | 9.5 / 10.0 — CRITICAL |
| Affected mode | `via-env` (`auth-user-pass-verify via-env`) |
| Safe mode | Management interface (`tcp://127.0.0.1:1195`) |
| Fixed version | v1.27.3 and above (this guide uses **v1.27.4**) |

Never use `auth-user-pass-verify /path/binary via-env` with `script-security 3`. This setup uses the management interface only, which is not affected.

---

## How it works

When a client connects, OpenVPN talks to `openvpn-auth-oauth2` on a local TCP socket. The user signs in with Google in a browser. The plugin tells OpenVPN ALLOW or DENY. Credentials never go through environment variables.

| Step | Component | Action |
| --- | --- | --- |
| 1 | VPN client | Connects to OpenVPN on UDP 1194 |
| 2 | OpenVPN server | Signals auth-oauth2 on TCP 1195 (management interface) |
| 3 | auth-oauth2 | Sends a browser login URL to the client |
| 4 | User browser | User logs in with Google |
| 5 | Google OAuth2 | Returns a token to the callback on port 9000 |
| 6 | auth-oauth2 | Validates the token, reports ALLOW or DENY |
| 7 | OpenVPN server | Grants or rejects the tunnel |

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
| TCP | 443 | Your IP/32 | HTTPS admin |
| TCP | 9000 | 0.0.0.0/0 | OAuth2 callback |

You also need a Google account that can use Google Cloud Console (Workspace or personal).

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

Install [OpenVPN Connect](https://openvpn.net/client/) (Windows, macOS, Android, iOS) or use `openvpn --config alice.ovpn` on Linux. Import `alice.ovpn` and connect.

On Linux:

```bash
sudo openvpn --config alice.ovpn
```

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
- If you enable `openvpn-auth-oauth2`, stay on **v1.27.4+** and the **management interface** (`tcp://127.0.0.1:1195`). Do not use `via-env`.
