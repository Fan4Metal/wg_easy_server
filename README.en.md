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

On the first visit to the panel, an administrator account is created. Then, under *Admin Panel → Config*, the following values are set:

- **Host**: `vpn.example.com`;
- **Port**: `443`.

These values end up in the `Endpoint` field of generated client configurations.

In the same *Admin Panel → Config* section, the `I1` signature packet (the *AmneziaWG Obfuscation Parameters* block) is set before the first client is created, as wg-easy does not fill it in automatically. The description and a link to ready-to-use values are given in [AmneziaWG signature packets (I1–I5)](#amneziawg-signature-packets-i1i5).

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

During the initial wg-easy setup (or later in *Admin Panel → Config*), the following values should be set so that generated client configurations contain the correct endpoint:

- **Host**: `vpn.example.com` (the configured domain)
- **Port**: `443` (the external UDP port)

Changing the **Port** value also changes the interface `ListenPort` inside the container, so the mapping in `compose.yaml` has to publish that same container port. After any port change, client configurations must be downloaded again.

Client applications must support AmneziaWG (AmneziaWG or AmneziaVPN apps); the standard WireGuard client cannot complete a handshake with an obfuscated server. Configurations generated by wg-easy already contain the required obfuscation parameters (`Jc`, `Jmin`, `Jmax`, `S1`–`S4`, `H1`–`H4`). The `I1`–`I5` signature packets, however, are not filled in automatically and are configured separately — see [AmneziaWG signature packets (I1–I5)](#amneziawg-signature-packets-i1i5).

## AmneziaWG signature packets (I1–I5)

The `Jc`, `Jmin`, `Jmax`, `S1`–`S4` and `H1`–`H4` parameters are generated by wg-easy on first startup. The `I1`–`I5` parameters (signature packets, labelled *Special junk packet* in the panel) are not filled in automatically and have to be set manually.

### Purpose

Each of the `I1`–`I5` parameters describes a single UDP packet that is sent before every handshake, in the order `I1`, `I2`, … `I5`; unset parameters are skipped. The packet contents imitate the beginning of a real protocol session (for example a QUIC Initial packet or a DNS response), so that a DPI system sees an ordinary HTTP/3 session rather than a WireGuard handshake. These packets carry no payload and do not need to match between server and client: the receiving side discards them as unrelated traffic. The AmneziaWG developers recommend setting them on the side that initiates the connection, that is, in client configurations.

### Requirements

- wg-easy 15.2 or newer; the `ghcr.io/wg-easy/wg-easy:15` image in `compose.yaml` satisfies this.
- Up-to-date AmneziaWG/AmneziaVPN client applications: the `I1`–`I5` parameters were introduced with the AmneziaWG 1.5 protocol, and older clients do not support them.
- For server-side values (*Admin Panel → Interface*), a kernel module with `I1`–`I5` support: a build from sources dated October 2025 or later (tag `v1.0.20251004` and newer). The packages from the `amnezia/ppa` PPA are built from current sources and include this support. The module version is irrelevant for client-side values.

### Value syntax

A parameter value is a sequence of tags without separators:

| Tag | Packet contents |
|-----|-----------------|
| `<b 0x…>` | static bytes given as a hex string (two characters per byte, even length) |
| `<r N>` | `N` random bytes |
| `<rd N>` | `N` random decimal digits (`0`–`9`) |
| `<rc N>` | `N` random Latin letters (`a`–`z`, `A`–`Z`) |
| `<t>` | current UNIX time, 4 bytes |

The resulting packet must not exceed the network MTU including the IP and UDP headers: a larger packet is fragmented, which is itself noticeable to DPI. Real QUIC clients send Initial packets of 1200 bytes, so values imitating them have the same size.

### Configuration in the wg-easy panel

The `I1`–`I5` fields are located in the *AmneziaWG Obfuscation Parameters* block of three panel sections:

| Section | Scope |
|---------|-------|
| *Admin Panel → Config* | defaults for **new** clients; existing clients are not affected |
| client page (edit button in the *Clients* list) | values of a specific client; included in its configuration |
| *Admin Panel → Interface* | values of the server itself, i.e. packets the server sends before its own handshakes; not required for client-side circumvention |

Recommended order: the values are entered in *Admin Panel → Config* before clients are created; for existing clients, on each client's page. Each field takes the value as a single line without line breaks. After a client's parameters change, its configuration has to be downloaded and imported into the app again, as it is not updated automatically. Changes take effect on save; a container restart is not required.

### Values in use

In this deployment only the `I1` field is filled in; the `I2`–`I5` fields are left empty. Three ready-to-use `I1` values are given below; **one** of them is used. Each value is entered into the *Special junk packet 1 (I1)* field as a single line without line breaks; all variants fit within the MTU.

**Variant 1. QUIC Initial with fixed contents.** A 1200-byte packet: long header `0xc7`, QUIC version 1, an 8-byte connection ID, payload length 1182 bytes, followed by fixed contents. To a DPI system it looks like the first packet of an HTTP/3 session.

```
<b 0xc70000000108ce1bf31eec7d93360000449e227e4596ed7f75c4d35ce31880b4133107c822c6355b51f0d7c1bba96d5c210a48aca01885fed0871cfc37d59137d73b506dc013bb4a13c060ca5b04b7ae215af71e37d6e8ff1db235f9fe0c25cb8b492471054a7c8d0d6077d430d07f6e87a8699287f6e69f54263c7334a8e144a29851429bf2e350e519445172d36953e96085110ce1fb641e5efad42c0feb4711ece959b72cc4d6f3c1e83251adb572b921534f6ac4b10927167f41fe50040a75acef62f45bded67c0b45b9d655ce374589cad6f568b8475b2e8921ff98628f86ff2eb5bcce6f3ddb7dc89e37c5b5e78ddc8d93a58896e530b5f9f1448ab3b7a1d1f24a63bf981634f6183a21af310ffa52e9ddf5521561760288669de01a5f2f1a4f922e68d0592026bbe4329b654d4f5d6ace4f6a23b8560b720a5350691c0037b10acfac9726add44e7d3e880ee6f3b0d6429ff33655c297fee786bb5ac032e48d2062cd45e305e6d8d8b82bfbf0fdbc5ec09943d1ad02b0b5868ac4b24bb10255196be883562c35a713002014016b8cc5224768b3d330016cf8ed9300fe6bf39b4b19b3667cddc6e7c7ebe4437a58862606a2a66bd4184b09ab9d2cd3d3faed4d2ab71dd821422a9540c4c5fa2a9b2e6693d411a22854a8e541ed930796521f03a54254074bc4c5bca152a1723260e7d70a24d49720acc544b41359cfc252385bda7de7d05878ac0ea0343c77715e145160e6562161dfe2024846dfda3ce99068817a2418e66e4f37dea40a21251c8a034f83145071d93baadf050ca0f95dc9ce2338fb082d64fbc8faba905cec66e65c0e1f9b003c32c943381282d4ab09bef9b6813ff3ff5118623d2617867e25f0601df583c3ac51bc6303f79e68d8f8de4b8363ec9c7728b3ec5fcd5274edfca2a42f2727aa223c557afb33f5bea4f64aeb252c0150ed734d4d8eccb257824e8e090f65029a3a042a51e5cc8767408ae07d55da8507e4d009ae72c47ddb138df3cab6cc023df2532f88fb5a4c4bd917fafde0f3134be09231c389c70bc55cb95a779615e8e0a76a2b4d943aabfde0e394c985c0cb0376930f92c5b6998ef49ff4a13652b787503f55c4e3d8eebd6e1bc6db3a6d405d8405bd7a8db7cefc64d16e0d105a468f3d33d29e5744a24c4ac43ce0eb1bf6b559aed520b91108cda2de6e2c4f14bc4f4dc58712580e07d217c8cca1aaf7ac04bab3e7b1008b966f1ed4fba3fd93a0a9d3a27127e7aa587fbcc60d548300146bdc126982a58ff5342fc41a43f83a3d2722a26645bc961894e339b953e78ab395ff2fb854247ad06d446cc2944a1aefb90573115dc198f5c1efbc22bc6d7a74e41e666a643d5f85f57fde81b87ceff95353d22ae8bab11684180dd142642894d8dc34e402f802c2fd4a73508ca99124e428d67437c871dd96e506ffc39c0fc401f666b437adca41fd563cbcfd0fa22fbbf8112979c4e677fb533d981745cceed0fe96da6cc0593c430bbb71bcbf924f70b4547b0bb4d41c94a09a9ef1147935a5c75bb2f721fbd24ea6a9f5c9331187490ffa6d4e34e6bb30c2c54a0344724f01088fb2751a486f425362741664efb287bce66c4a544c96fa8b124d3c6b9eaca170c0b530799a6e878a57f402eb0016cf2689d55c76b2a91285e2273763f3afc5bc9398273f5338a06d>
```

**Variant 2. DNS response.** A 48-byte packet: a random transaction ID (`<r 2>`) followed by a static response to an `A` record query for `yabs.yandex.ru` with the address `87.250.39.209`.

```
<r 2><b 0x8580000100010000000004796162730679616e6465780272750000010001c00c000100010000026d000457fa27d1>
```

**Variant 3. QUIC Initial with random contents.** A 1200-byte packet: header `0xc3`, QUIC version 1, a random connection ID (`<r 8>`), payload length 1182 bytes (`0x449e`), random packet number and payload. Unlike variant 1, the packet contents change on every send.

```
<b 0xc30000000108><r 8><b 0x0000449e><r 4><r 1000><r 178>
```

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
6. **Outdated client and `I1`–`I5` parameters.** Client applications without AmneziaWG 1.5 support do not accept configurations containing signature packets; the application has to be updated.

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
- [amneziawg-go: parameter reference, including the I1–I5 signature packets](https://github.com/amnezia-vpn/amneziawg-go#custom-signature-packets)
- [Caddy documentation](https://caddyserver.com/docs/)
