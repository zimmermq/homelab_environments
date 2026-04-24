# Post-Install Steps

## Netplan DNS Configuration

The node's primary DNS (`192.168.0.91`) points to AdGuard running inside the cluster via Traefik's LoadBalancer. This creates a circular dependency on boot: Traefik needs DNS to download the cloudflare plugin, but DNS goes through Traefik.

Without a fallback, Traefik's plugin download times out, disabling the `cloudflare` middleware and breaking all HTTPS routes (404 on every service).

The DHCP-provided search domain (`fritz.box` from the Fritz!Box) also causes the node (hostname `ubuntu`) to generate thousands of junk DNS queries like `ubuntu.fritz.box`, `www.google.com.fritz.box`, etc. These queries also leak into pod `resolv.conf` via kubelet.

**After installing the OS**, replace `/etc/netplan/50-cloud-init.yaml` with:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      optional: true
      dhcp4: true
      dhcp4-overrides:
        use-domains: false
        use-dns: false
      nameservers:
        addresses:
          - 192.168.0.91
          - 8.8.8.8
          - 1.1.1.1
```

Then apply:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

### Why each setting

| Setting | Purpose |
|---|---|
| `use-dns: false` | Ignore DNS servers from DHCP (Fritz!Box would push only `192.168.0.91`, with no fallback). |
| `use-domains: false` | Ignore the `fritz.box` search domain to stop junk lookups. |
| `nameservers.addresses` | Static list. `192.168.0.91` (AdGuard) is queried first; `8.8.8.8` / `1.1.1.1` take over automatically when AdGuard is unreachable (bootstrap, pod restart, host reboot). |

Note: systemd-resolved's global `FallbackDNS=` is **not** sufficient here — it only engages when no per-link DNS is known. Since DHCP provides a link-level DNS, fallback has to be configured at the same (per-link) scope.

### Verify

```bash
resolvectl status eth0 | head -5
# Should show: DNS Servers: 192.168.0.91 8.8.8.8 1.1.1.1

cat /etc/resolv.conf | grep search
# Should NOT contain fritz.box
```
