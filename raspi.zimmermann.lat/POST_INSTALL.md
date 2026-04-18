# Post-Install Steps

## Fallback DNS on the Node

The node's primary DNS (`192.168.0.91`) points to AdGuard running inside the cluster via Traefik's LoadBalancer. This creates a circular dependency on boot: Traefik needs DNS to download the cloudflare plugin, but DNS goes through Traefik.

Without a fallback, Traefik's plugin download times out, disabling the `cloudflare` middleware and breaking all HTTPS routes (404 on every service).

**After installing the OS, configure a fallback DNS:**

```bash
sudo sed -i 's/^#FallbackDNS=$/FallbackDNS=8.8.8.8 1.1.1.1/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

Verify with:

```bash
resolvectl status | head -5
# Should show: Fallback DNS Servers: 8.8.8.8 1.1.1.1
```
