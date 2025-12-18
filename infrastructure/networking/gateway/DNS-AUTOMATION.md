# Automated DNS Management with ExternalDNS

This cluster uses ExternalDNS to automatically create Cloudflare DNS records for both external and internal services.

## How It Works

ExternalDNS watches HTTPRoute resources and creates DNS records in Cloudflare based on labels and annotations.

### External Services (Proxied via Cloudflare)
- DNS Record Type: CNAME → `jrb.nz`
- Cloudflare Proxy: **Enabled** (orange cloud)
- Benefits: CDN, DDoS protection, caching
- Use for: Public-facing services accessible from internet

### Internal Services (Private IPs)
- DNS Record Type: A → `192.168.86.50`
- Cloudflare Proxy: **Disabled** (gray cloud)
- Benefits: Direct connection, works on LAN
- Use for: Internal-only services (ArgoCD, Grafana, Longhorn, etc.)

## Configuration

### Gateway Setup

**External Gateway** (`gw-external.yaml`):
```yaml
metadata:
  labels:
    external-dns: "true"
  annotations:
    external-dns.alpha.kubernetes.io/target: "jrb.nz"
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
```

**Internal Gateway** (`gw-internal.yaml`):
```yaml
metadata:
  labels:
    external-dns: "true"
  annotations:
    external-dns.alpha.kubernetes.io/target: "192.168.86.50"
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"
```

### HTTPRoute Pattern

To enable DNS automation for an HTTPRoute, add the `external-dns: "true"` label:

**Example - Internal Service** (ArgoCD):
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd
  namespace: argocd
  labels:
    external-dns: "true"  # ← ADD THIS
spec:
  parentRefs:
    - name: gateway-internal
      namespace: gateway
  hostnames:
    - "argocd.jrb.nz"
  rules:
    - backendRefs:
        - name: argocd-server
          port: 80
```

**Result in Cloudflare:**
```
Type: A
Name: argocd.jrb.nz
Value: 192.168.86.50
Proxied: No (gray cloud)
TTL: Auto
```

**Example - External Service** (hypothetical public app):
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: public-app
  namespace: apps
  labels:
    external-dns: "true"  # ← ADD THIS
spec:
  parentRefs:
    - name: gateway-external
      namespace: gateway
  hostnames:
    - "public-app.jrb.nz"
  rules:
    - backendRefs:
        - name: public-app
          port: 80
```

**Result in Cloudflare:**
```
Type: CNAME
Name: public-app.jrb.nz
Value: jrb.nz
Proxied: Yes (orange cloud)
TTL: Auto
```

## What Gets Created

### For Each HTTPRoute with `external-dns: "true"` Label

1. **DNS Record** - Hostname from HTTPRoute → Target from Gateway
2. **TXT Record** - Ownership tracking (prefix: `external-dns-`)

### DNS Record Behavior

- **Internal Services**: A record → `192.168.86.50` (unproxied)
  - Works from LAN: ✅ Yes
  - Works from Internet: ❌ No (private IP)

- **External Services**: CNAME → `jrb.nz` (proxied)
  - Works from LAN: ✅ Yes (via Cloudflare Tunnel)
  - Works from Internet: ✅ Yes (via Cloudflare CDN)

## Enabling DNS for Existing Services

To enable automatic DNS for any service:

1. **Add label to HTTPRoute**:
   ```yaml
   labels:
     external-dns: "true"
   ```

2. **Commit and push** (GitOps)

3. **Wait ~1 minute** - ExternalDNS syncs every 60 seconds

4. **Verify in Cloudflare**:
   ```bash
   # Check DNS record
   dig argocd.jrb.nz

   # Check Cloudflare directly
   nslookup argocd.jrb.nz 1.1.1.1
   ```

5. **Verify in cluster**:
   ```bash
   # Check ExternalDNS logs
   kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns

   # Check what records are managed
   kubectl get dnsendpoint -A
   ```

## Current Services to Annotate

Add `external-dns: "true"` label to these HTTPRoutes for automatic DNS:

### Internal Services (gateway-internal)
- ✅ `infrastructure/controllers/argocd/http-route.yaml` - argocd.jrb.nz
- ⬜ `infrastructure/storage/longhorn/httproute.yaml` - longhorn.jrb.nz
- ⬜ `infrastructure/database/redis/redis-commander/httproute.yaml` - redis.jrb.nz
- ⬜ `monitoring/prometheus/httproute.yaml` - prometheus.jrb.nz
- ⬜ `monitoring/grafana/httproute.yaml` - grafana.jrb.nz
- ⬜ `monitoring/alertmanager/httproute.yaml` - alertmanager.jrb.nz
- ⬜ `my-apps/media/homepage-dashboard/httproute.yaml` - homepage.jrb.nz
- ⬜ `my-apps/media/garage-webui/http-route.yaml` - garage.jrb.nz
- ⬜ `my-apps/media/echo-server/httproute.yaml` - echo.jrb.nz

### External Services (gateway-external)
- (Add as needed for public-facing services)

## Monitoring

### ExternalDNS Logs
```bash
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns -f
```

### Check DNS Records in Cloudflare
Login to Cloudflare → DNS → jrb.nz zone

Look for:
- A records pointing to `192.168.86.50` (internal)
- CNAME records pointing to `jrb.nz` (external)
- TXT records with `external-dns-` prefix (ownership)

### Troubleshooting

**HTTPRoute not creating DNS record:**
1. Check label: `kubectl get httproute <name> -n <namespace> -o yaml | grep external-dns`
2. Check logs: `kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns`
3. Verify gateway has `external-dns: "true"` label
4. Check domain filter: Only `jrb.nz` domain is managed

**DNS record points to wrong IP:**
- Check gateway annotation: `external-dns.alpha.kubernetes.io/target`
- Internal should be: `192.168.86.50`
- External should be: `jrb.nz`

**Cloudflare proxy setting wrong:**
- Internal services should have `proxied: false` (gray cloud)
- External services should have `proxied: true` (orange cloud)
- Set via gateway annotation: `external-dns.alpha.kubernetes.io/cloudflare-proxied`

## Security Considerations

### Publishing Private IPs to Public DNS

Publishing private IPs (192.168.86.x) in public DNS is **safe** because:

1. **RFC1918 addresses are non-routable** on the internet
2. External clients can't reach them (firewall blocks)
3. Internal clients resolve same DNS and **can** reach them
4. No secrets or service discovery info leaked

### Alternative: Split-Horizon DNS

If you prefer to keep internal services private, you can:
1. Remove `external-dns: "true"` label from internal HTTPRoutes
2. Configure local DNS (router, Pi-hole, etc.) with wildcard:
   ```
   *.jrb.nz → 192.168.86.50
   ```

## Migration Notes

This setup replaces the manual Firewalla DNS configuration. All previous manual entries are now automated:

**Before (Manual):**
```
# Firewalla DNS entries (31 manual entries)
argocd.jrb.nz → 192.168.86.50
grafana.jrb.nz → 192.168.86.50
prometheus.jrb.nz → 192.168.86.50
... (28 more)
```

**After (Automated):**
```
# Just add label to HTTPRoute
labels:
  external-dns: "true"
```

ExternalDNS automatically creates the Cloudflare DNS record.
