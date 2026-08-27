# WireGuard Easy Server

**English** | [Русский](README.md)

A Docker Compose deployment of [wg-easy](https://github.com/wg-easy/wg-easy) — a WireGuard VPN server with a web-based management interface — extended with [AmneziaWG](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module) traffic obfuscation and a [Caddy](https://caddyserver.com/) reverse proxy that provides automatic HTTPS for the management panel.

## Architecture

| Component | Purpose | Exposed ports |
|-----------|---------|---------------|
| `wg-easy` | WireGuard/AmneziaWG server and management UI | `443/udp` (VPN endpoint) |
| `caddy` | Reverse proxy with automatic Let's Encrypt TLS | `80/tcp`, `443/tcp` |

Key design decisions:

- The management UI is not exposed directly; it is reachable only through Caddy over HTTPS. Certificates are obtained and renewed automatically. The domain name is configured in the `Caddyfile`; throughout this document, `vpn.example.com` is used as a placeholder.
- The VPN endpoint listens on `443/udp`. On this port, obfuscated AmneziaWG traffic is indistinguishable from QUIC (HTTP/3) for simple DPI filters, and the port itself is almost never blocked. There is no conflict with Caddy, which occupies `443/tcp`.
- The port configured in the wg-easy panel changes the actual `ListenPort` of the interface inside the container, therefore the port mapping in `compose.yaml` publishes the same container port: `443:443/udp`. A mismatch between the mapping and the panel setting results in clients being unable to complete a handshake.
- `OVERRIDE_AUTO_AWG=awg` forces the AmneziaWG implementation: if the kernel module is missing, the container fails with an explicit error instead of silently falling back to plain WireGuard.

## Requirements

- A Linux host with Docker and the Docker Compose plugin.
- A DNS `A` record for the chosen domain (`vpn.example.com`) pointing to the server's public IP address (required before the first start, otherwise certificate issuance fails).
- Inbound ports open on the firewall: `80/tcp`, `443/tcp`, and `443/udp`.
- The AmneziaWG kernel module installed on the host (see below).

## Step-by-step installation

### Step 1. DNS and network

- A DNS `A` record for the chosen domain (`vpn.example.com`) is created, pointing to the server's public IP address. The record must have propagated before the first start — this is verified with `nslookup vpn.example.com`.
- Inbound ports `80/tcp`, `443/tcp`, and `443/udp` are opened on the server firewall and in the provider's panel (security group). `443/udp` is opened as a separate rule — a TCP rule for the same port does not cover it.

### Step 2. Docker

If Docker is not present, it is installed with the official script:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Verification: `docker compose version` prints the Compose plugin version.

### Step 3. AmneziaWG kernel module

The module is installed on the host as described in [Installing the AmneziaWG kernel module](#installing-the-amneziawg-kernel-module). Before proceeding, the module load is confirmed:

```bash
sudo modprobe amneziawg
lsmod | grep amneziawg
```

### Step 4. Project files

Two files are placed in a working directory on the server (e.g. `~/wg-easy`):

- `compose.yaml` — the full contents are provided in the [Files](#files) section;
- `Caddyfile` — an example is provided in the [Reverse proxy configuration](#reverse-proxy-configuration) section; the `vpn.example.com` placeholder is replaced with the real domain name.

### Step 5. Startup

```bash
docker compose up -d
```

Within a few seconds Caddy obtains a Let's Encrypt certificate, after which the management panel opens at `https://vpn.example.com`. Certificate issuance can be followed with `docker logs caddy -f` if needed.

### Step 6. Initial wg-easy configuration

On the first visit to the panel, an administrator account is created. Then, under *Admin Panel → General*, the following values are set:

- **Host**: `vpn.example.com`;
- **Port**: `443`.

These values end up in the `Endpoint` field of generated client configurations.

### Step 7. Client connection

1. A client is created in the panel; its configuration is downloaded as a file or scanned as a QR code.
2. The [AmneziaWG](https://docs.amnezia.org/documentation/amnezia-wg/) or AmneziaVPN application is installed on the device (the standard WireGuard client is incompatible — see [Troubleshooting](#troubleshooting)).
3. The configuration is imported into the application and the tunnel is enabled.

A successful connection is confirmed by a completed handshake: the panel shows the client's latest handshake time, and the same is visible on the server in the output of `docker exec wg-easy awg show` (the `latest handshake` and `transfer` lines).

## Installing the AmneziaWG kernel module

The module is installed on the host system, not inside the container. The container loads it through the mounted `/lib/modules` and the `SYS_MODULE` capability, both of which are already configured in `compose.yaml`.

### Ubuntu

`deb-src` repositories must be enabled first (the package builds the module from kernel sources): in Ubuntu 24.04+ this is done by uncommenting `deb-src` in the `Types:` line of `/etc/apt/sources.list.d/ubuntu.sources`; in older releases, by uncommenting the `deb-src` lines in `/etc/apt/sources.list`.

```bash
sudo apt install -y software-properties-common python3-launchpadlib gnupg2 linux-headers-$(uname -r)
sudo add-apt-repository ppa:amnezia/ppa
sudo apt-get update
sudo apt-get install -y amneziawg
```

The package uses DKMS, so the module is rebuilt automatically on kernel upgrades.

### Other distributions

Instructions for Debian and other distributions, as well as building from source, are covered in the original documentation: [amnezia-vpn/amneziawg-linux-kernel-module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module).

### Verification

```bash
sudo modprobe amneziawg
lsmod | grep amneziawg
```

A non-empty output of the second command confirms that the module is loaded.

## Deployment

```bash
docker compose up -d
```

On first start, Caddy obtains a Let's Encrypt certificate for the configured domain (this takes a few seconds), after which the management panel becomes available at `https://vpn.example.com`.

During the initial wg-easy setup (or later in *Admin Panel → General*), the following values should be set so that generated client configurations contain the correct endpoint:

- **Host**: `vpn.example.com` (the configured domain)
- **Port**: `443` (the external UDP port)

Changing the **Port** value also changes the interface `ListenPort` inside the container, so the mapping in `compose.yaml` has to publish that same container port. After any port change, client configurations must be downloaded again.

Client applications must support AmneziaWG (AmneziaWG or AmneziaVPN apps); the standard WireGuard client cannot complete a handshake with an obfuscated server. Configurations generated by wg-easy already contain the required obfuscation parameters (`Jc`, `Jmin`, `Jmax`, `S1`, `S2`, `H1`–`H4`).

## Reverse proxy configuration

The entire Caddy configuration consists of a single site block. An example `Caddyfile`:

```
vpn.example.com {
	reverse_proxy wg-easy:51821
}
```

The domain name is the only value that needs to be adjusted. Certificate issuance and renewal, TLS termination, and the HTTP→HTTPS redirect are performed by Caddy automatically and require no additional directives.

## Files

- [`compose.yaml`](compose.yaml) — service definitions (wg-easy, Caddy, networks, volumes).
- `Caddyfile` — reverse proxy configuration (see the example above).

<details>
<summary><code>compose.yaml</code> (full contents)</summary>

```yaml
volumes:
  etc_wireguard:
  caddy_data:
  caddy_config:

services:
  wg-easy:
    environment:
     - EXPERIMENTAL_AWG=true
     - OVERRIDE_AUTO_AWG=awg

    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "443:443/udp"
    restart: unless-stopped
    # The image's built-in healthcheck uses `wg show`, which does not see
    # amneziawg interfaces, so the container is always reported unhealthy
    healthcheck:
      test: ["CMD-SHELL", "awg show | grep -q interface || exit 1"]
      interval: 60s
      timeout: 5s
      retries: 3
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
      # - NET_RAW # ⚠️ Uncomment if using Podman
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1

  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - wg

networks:
  wg:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 10.42.42.0/24
        - subnet: fdcc:ad94:bacf:61a3::/64
```

</details>

## Troubleshooting

### No handshake

A checklist ordered by likelihood, based on real-world debugging of this setup:

1. **Port mapping vs. interface port.** The container port in the `ports:` mapping must match the port configured in the wg-easy panel (which sets the interface `ListenPort`). The state can be inspected with `docker exec wg-easy awg show` (the `listening port` line) and compared with the mapping shown by `docker ps`.
2. **Client application.** The standard WireGuard client is incompatible with an obfuscated AmneziaWG server; AmneziaWG or AmneziaVPN applications are required.
3. **Stale client configuration.** After a change of the panel port, previously downloaded client configurations keep the old `Endpoint` and must be re-downloaded.
4. **Provider firewall.** `443/udp` is opened separately from `443/tcp` in most cloud security groups. A silent `sudo tcpdump -ni any udp port 443` during a connection attempt indicates that packets do not reach the server at all.
5. **Kernel module.** `lsmod | grep amneziawg` on the host; with `OVERRIDE_AUTO_AWG=awg`, a missing module makes the container fail explicitly.

### Container reported as `unhealthy`

The image's built-in healthcheck runs the standard `wg show`, which does not see AmneziaWG interfaces, so the container is always reported as `unhealthy` while operating normally. This is a false alarm and does not affect the tunnel.

In the provided `compose.yaml` the issue is fixed by overriding the healthcheck with the `awg` utility shipped in the image:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "awg show | grep -q interface || exit 1"]
      interval: 60s
      timeout: 5s
      retries: 3
```

For the same reason, diagnostics inside the container should use `awg show` instead of `wg show`.

## Notes

- Certificates are stored in the `caddy_data` volume and survive container recreation.
- WireGuard configuration and peer data are stored in the `etc_wireguard` volume.
- Some networks (hotels, certain mobile carriers) block UDP entirely; no port choice mitigates this, and a TCP-based transport would be required in such environments.

## References

- [wg-easy documentation: AmneziaWG](https://wg-easy.github.io/wg-easy/v15.3/advanced/config/amnezia/)
- [AmneziaWG Linux kernel module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module)
- [Caddy documentation](https://caddyserver.com/docs/)
