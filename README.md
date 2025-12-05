# PORTAL-TO-VPN

TCP/HTTP proxy that sends selected domains **through a WireGuard VPN** using:

* **Gluetun** (WireGuard client + firewall + DNS)
* **nginx** (SNI + Host-based proxy inside Gluetun's netns)

You point a domain's DNS to this box -> it dials the real upstream **via the VPN**.
Works for **HTTPS and HTTP**, with no per-domain configuration.

Inspired by Tymscar's [Imgur Geo-Blocked the UK, So I Geo-Unblocked My Entire Network](https://blog.tymscar.com/posts/imgurukproxy/) article.

---

## How it works (TL;DR)

1. Client resolves `example.com` to the IP of this stack.
2. nginx accepts traffic on ports **80 or 443**.
3. If HTTPS:
   * nginx **does not decrypt TLS**
   * uses **SNI** (`$ssl_preread_server_name`)
   * proxies blindly to `<SNI>:443`
4. If HTTP:
   * nginx reads the **Host** header
   * proxies to `<Host>:80`
5. nginx runs **inside Gluetun's network namespace**, so **all upstream traffic goes through WireGuard**, including DNS.

---

## Files

* `docker-compose.yml` – stack definition
* `gluetun/wg0.conf.example` – WireGuard example (copy to `wg0.conf`)
* `nginx/nginx.conf` – TCP + HTTP transparent proxy

---

## Requirements

* Docker + docker compose
* Host (or LXC) with **`/dev/net/tun`** available to Docker
* WireGuard config from your VPN provider
* Ability to point **DNS** for chosen domains to this stack's IP

---

## Testing with `curl`

Assuming the stack's IP is `10.10.10.10`:

### Direct (no VPN)
```bash
curl -s https://ifconfig.me
````

Shows your **public IP**.

### Via portal + VPN

#### HTTPS

```bash
curl -s --resolve ifconfig.me:443:10.10.10.10 https://ifconfig.me
```

#### HTTP

```bash
curl -s --resolve ifconfig.me:80:10.10.10.10 http://ifconfig.me
```

Both should return the **VPN egress IP** .

---

## Adding more domains

Just point DNS for any domain to the portal's IP.

### `/etc/hosts` (local testing)

```bash
echo "10.10.10.10 ifconfig.me" | sudo tee -a /etc/hosts
echo "10.10.10.10 i.imgur.com" | sudo tee -a /etc/hosts
```

### Mikrotik DNS

```text
/ip dns static
add name=i.imgur.com   address=10.10.10.10 comment="portal-to-vpn"
add name=example.com   address=10.10.10.10 comment="portal-to-vpn"
```

No nginx changes are required - the stack is fully **domain-agnostic**.

---

## Logging

Both HTTP and HTTPS access logs are streamed to Docker stdout:

```bash
docker compose logs -f portal-nginx
```

No log files are created on disk.

---

## Notes

* Gluetun firewall blocks all non-VPN egress.
* nginx cannot accidentally leak traffic - its network namespace **is Gluetun**.
* Only minimum capabilities are enabled (e.g. bind port 80/443, setuid).
