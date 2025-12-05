# PORTAL-TO-VPN

TCP SNI portal that sends selected HTTPS domains **through a WireGuard VPN** using:

* **Gluetun** (WireGuard client + firewall + DNS)
* **nginx** (TCP SNI proxy inside Gluetun's netns)
* **Traefik** (TCP passthrough entrypoint on port 443)

You point a domain's DNS to this box → it dials the real upstream **via the VPN**.

Inspired by Tymscar's [Imgur Geo-Blocked the UK, So I Geo-Unblocked My Entire Network](https://blog.tymscar.com/posts/imgurukproxy/) article.

---

## How it works (TL;DR)

1. Client resolves `example.com` to the IP of this stack.
2. Traefik accepts TCP:443 and forwards it via **TLS passthrough** to nginx.
3. nginx uses **SNI** (`$ssl_preread_server_name`) and proxies to `SNI:443`.
4. nginx runs **inside Gluetun's network namespace**, so all upstream traffic goes through WireGuard.

---

## Files

* `docker-compose.yml` – stack definition
* `gluetun/wg0.conf.example` – example WireGuard config (copy to `wg0.conf`)
* `nginx/nginx.conf` – SNI-based TCP proxy
* `traefik/traefik.yml` – static Traefik config
* `traefik/dynamic/tcp.yml` – Traefik TCP router (TLS passthrough)

---

## Requirements

* Docker + docker compose
* Host (or LXC) with **`/dev/net/tun`** available to Docker
* WireGuard config from your VPN provider
* Ability to point **DNS** for chosen domains to this stack's IP

---

## Testing with `curl`

You can test **without touching your DNS server** using `curl --resolve`.

Assuming the stack's IP is `10.10.10.10`:

```bash
# Direct (no portal)
curl -s https://ifconfig.me

# Via portal+VPN
curl -s --resolve ifconfig.me:443:10.10.10.10 https://ifconfig.me
```

The second result should show the **VPN egress IP**.

---

## Adding more domains

Just point additional DNS names to the stack's IP.

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

No changes to Traefik or nginx are required - the stack is fully **domain-agnostic**.
