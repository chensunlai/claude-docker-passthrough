---
name: claude-docker-passthrough
description: Build or maintain a portable Docker deployment with a VLESS Reality gateway and an Ubuntu GPU desktop, including an isolated proxy exit, SSH, VNC, and a contiguous forwarded port range.
---

# Claude Docker Passthrough

Create a Docker Compose project that builds from scratch on a compatible Linux server. Use the user's chosen directory or a suitable working directory. Keep Docker's data directory unchanged. Derive deployment parameters from the target server and user input; do not depend on a previous server's files, images, accounts, paths, or addresses.

## Requirements

- Use exactly two services: `gateway` runs sing-box TUN, VLESS Reality, and inbound forwarding; `app` provides a full Ubuntu userspace with XFCE, a browser, SSH, TigerVNC, and noVNC. Let Compose manage container names unless the user specifies them.
- Give app its own network namespace with only `eth0` and `lo`. Keep TUN inside gateway. Do not share gateway's network namespace or add a sandbox container.
- Route all app outbound TCP, UDP, and DNS through the remote node. Fail closed when the proxy is unavailable. Preserve the host's and other containers' networking.
- Use English throughout: `LANG=LC_ALL=en_US.UTF-8`, `LANGUAGE=en_US:en`, and browser locale `en-US`. Remove Chinese desktop, browser, and noVNC translation resources. Set `TZ` to the exit location's IANA timezone, supplied by the user or determined from the exit location.
- Obtain the regular username and SSH password or public key from the user. Enable passwordless sudo and disable root SSH login. Configure a separate VNC password. Keep credentials out of this skill and public repositories.
- Default to no host directory mounts, Docker socket, host network/PID namespace, privileged mode, or SYS_ADMIN. For GPU hosts, use NVIDIA Container Toolkit with `gpus: all`, driver capabilities `compute,utility`, and `shm_size: 1gb`. If no GPU is available, explain the limitation and use CPU mode only if acceptable to the user.

## Deployment parameters

Manage these centrally and use them consistently across Compose, configuration, and entrypoint scripts:

- Node IP, port, UUID, Reality `public_key`, `short_id`, and `server_name`. Resolve a node hostname to an IP before configuring the connection.
- `APP_USER`, login credentials, `TZ`, and an external DNS address.
- Uplink and private subnets that do not overlap host routes or existing Docker networks; gateway private IP, app IP, and Docker IPAM's reserved gateway address.
- Host bind address, published SSH/VNC/noVNC ports, and at least 16 contiguous application ports. Check existing use and the host's ephemeral port range; choose another range on conflict.

Bind to a reachable address on the target server, or all addresses when requested. Default container SSH/VNC/noVNC ports are `22/5901/6080`; choose available host ports. Do not hardcode addresses or timezones from a previous deployment.

## Build and deploy

1. **Prepare.** Check Docker Engine, Compose, `/dev/net/tun`, CPU architecture, and GPU runtime. Confirm support for `interface_name`, `gw_priority`, isolated bridge mode, and the chosen GPU configuration. Generate `compose.yaml`, two Dockerfiles, sing-box configuration, and service entrypoint scripts. Nonsecret parameters may use `.env`; store credentials separately and exclude them from version control.
2. **Build gateway.** Pin an official sing-box image version or digest. COPY configuration and entrypoint scripts into the image rather than bind mounting them. Configure TUN `address`, `auto_route`, and `strict_route`; enable route `auto_detect_interface`. Use VLESS Reality as the default outbound with uTLS chrome, `xtls-rprx-vision`, and `xudp`. Match fields to the selected version. Where supported, `dns_mode: disabled` disables automatic TUN DNS interception so DNS travels as ordinary proxied traffic. Grant gateway NET_ADMIN, the TUN device, and IPv4 forwarding.
3. **Connect networks and forward ports.** Attach gateway to an uplink bridge on `eth0` and a private bridge on `eth1`, with the uplink having the highest `gw_priority`. Attach app only to the private bridge on `eth0`. Set the private network to `internal: true` and `com.docker.network.bridge.gateway_mode_ipv4: isolated`, without assigning a private bridge IP to the host. Keep Docker IPAM's reserved gateway address distinct from the gateway container's IP; point app's actual default route to the container. Publish all ports on gateway using normal Compose `ports`, then DNAT to app inside gateway using nftables. SNAT forwarded inbound traffic to gateway's private IP so replies follow the return path. Forward the application port range for both TCP and UDP; do not add a separate host NAT service.
4. **Prevent bypass.** Load gateway firewall rules before starting sing-box. Allow only the node IP/port and established traffic on the uplink. Default forwarding to drop, allowing only app-to-TUN traffic, established connections, and DNATed inbound connections. Block new app connections to gateway itself. Exclude private routes from TUN to preserve inbound replies while blocking app access to the host/LAN through the uplink. Update firewall rules whenever the node IP changes. Give app an external DNS server directly to avoid Docker embedded DNS bypass. Default to IPv4 only; IPv6 requires equivalent proxy routing and leak prevention.
5. **Build desktop.** Start from an official Ubuntu LTS image, such as `ubuntu:24.04`. Install XFCE, a browser available for the target architecture, TigerVNC, noVNC/websockify, openssh-server, sudo, locales, tzdata, iproute2, and util-linux. Configure the user, English locale, timezone, and service entrypoint; systemd is unnecessary. Run the browser as the regular user. Use `--no-sandbox` only for a demonstrated compatibility issue and explain the reduced browser isolation; do not expand container privileges to fix it. NVIDIA runtime does not install the full CUDA Toolkit. Add application CUDA libraries, nvcc, or PyTorch as needed. Generate SSH host keys and VNC credentials on first startup, preserving them across restarts of the same instance.
6. **Start.** Temporarily grant app NET_ADMIN. Its entrypoint runs `ip route replace default via "$GATEWAY_IP" dev eth0`, sets external DNS, then starts SSH and desktop services through `setpriv --bounding-set=-net_admin --inh-caps=-net_admin --ambient-caps=-net_admin`. This prevents service descendants, including sudo sessions, from regaining NET_ADMIN. Do not enable `no-new-privileges` when it would prevent the requested sudo access. Start app after gateway is ready, use `restart: unless-stopped`, and deploy with `docker compose up -d --build`.

## Block documentation networks before TUN

Following [RFC 5737](https://www.rfc-editor.org/rfc/rfc5737.html), silently drop all traffic to `192.0.2.0/24`, `198.51.100.0/24`, and `203.0.113.0/24` at the beginning of gateway's nftables `forward` and `output` chains. Place these rules before `ct state established,related accept` and app-to-TUN acceptance. An interval address set with `ip daddr @documentation_v4 counter drop` is sufficient. Include the rules in the image entrypoint so they survive restarts and rebuilds.

Block before traffic enters TUN or the userspace proxy stack. Rejecting only at the proxy outbound may still allow an initial local TCP handshake. Use silent DROP for these ranges rather than synthetic TCP success or ICMP Echo Replies. Do not accidentally block the TUN addressing or required private networks. Add equivalent documentation-prefix filtering when enabling IPv6.

For one address in each IPv4 range, such as `.1`, verify that TCP connect fails, no ICMP Echo Reply arrives, and firewall drop counters increase. Also verify a normal HTTPS request through the exit. A successful TUN TCP handshake or low-latency ping alone does not prove remote reachability; exit checks must include an application-layer response.

## Handoff and maintenance

- Run only relevant checks: app has eth0/lo only, TCP/UDP/DNS exit correctly, SSH/VNC work, and GPU access works. If the user requests skipping checks, state what remains unverified. Container startup alone does not establish functional success.
- Provide the actual SSH command, VNC address, noVNC URL, forwarded port range, and credential retrieval instructions. Application services must bind to `0.0.0.0`. DNAT/SNAT makes inbound peers appear as gateway's private IP inside app.
- By default, data lives in app's writable layer: restart preserves it; removal or recreation loses new files. Stop app and snapshot or back up before replacement, then restore into the new container. Offer Docker named volumes for long-term persistence when consistent with the user's isolation requirements.
- Fresh deployments must build from public base images without historical snapshots or machine-specific files. Private snapshots are only for preserving an existing instance. Migrate by export/import or by rebuilding and restoring a backup. Regenerate SSH host keys for independent clones.
- Keep images containing node credentials, accounts, or user data private. Reuse build caches, avoid reinstalling unrelated components, and leave unrelated containers and networks untouched. Docker containers still share the host kernel.
