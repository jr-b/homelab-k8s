# DNS + TLS Configuration Changes Summary

## What Was Done

### 1. ✅ TLS Certificate Automation

**Created Certificate Resource:**
- `infrastructure/networking/gateway/certificate.yaml`
  - Requests wildcard cert for `*.jrb.nz` and `jrb.nz`
  - Uses Let's Encrypt via Cloudflare DNS-01 challenge
  - Generates secret `cert-jrbnz` in gateway namespace
  - Auto-renews 30 days before expiration

**Updated Gateways:**
- `gw-internal.yaml` - Now references `cert-jrbnz`
- `gw-external.yaml` - Now references `cert-jrbnz`
- Both already had cert-manager annotations ✅

### 2. ✅ Automated DNS with ExternalDNS

**Modified ExternalDNS Configuration:**
- `infrastructure/controllers/external-dns/values.yaml`
  - Removed gateway filter to watch BOTH gateways
  - Disabled global Cloudflare proxy (now controlled per-route)
  - HTTPRoutes with `external-dns: "true"` label will auto-create DNS

**Updated Gateway Annotations:**
- `gw-external.yaml`
  - Target: `jrb.nz` (creates CNAME)
  - Cloudflare proxy: `true` (orange cloud)

- `gw-internal.yaml`
  - Target: `192.168.86.50` (creates A record)
  - Cloudflare proxy: `false` (gray cloud)
  - Added `external-dns: "true"` label

**Added DNS Automation to HTTPRoutes:**
All these now have `external-dns: "true"` label and will auto-create Cloudflare DNS:
- ✅ `argocd.jrb.nz` → 192.168.86.50
- ✅ `longhorn.jrb.nz` → 192.168.86.50
- ✅ `redis.jrb.nz` → 192.168.86.50
- ✅ `homepage.jrb.nz` → 192.168.86.50
- ✅ `garage.jrb.nz` → 192.168.86.50
- ✅ `echo.jrb.nz` → 192.168.86.50

### 3. ✅ Fixed IP Subnet (192.168.10 → 192.168.86)

**Updated Infrastructure:**
- Gateway internal IP: `192.168.86.50`
- Cilium IP pool: `192.168.86.32/27` (range: .32-.63)
- Gateway IPs: .49 (external), .50 (internal)
- TrueNAS server: `192.168.86.133` (NFS/SMB/MinIO)

**Updated LoadBalancer IPs:**
- Redis: `192.168.86.44`
- Redis Commander: `192.168.86.45`
- Paperless PostgreSQL: `192.168.86.42`
- Khoj PostgreSQL: `192.168.86.47`

**Updated Storage:**
- All SMB storage classes (6 total)
- NFS storage class
- Longhorn backup config (commented NFS)

**Updated Scripts:**
- Worker node IPs: `192.168.86.111-114`
- MinIO console: `192.168.86.133:9002`

**Updated Documentation:**
- README.md - All IP references
- infrastructure/networking/README.md - Network diagrams

## What Happens When You Sync

### Immediate (ArgoCD Sync)

1. **cert-manager** will issue wildcard certificate
   - DNS-01 challenge via Cloudflare
   - Creates secret `cert-jrbnz` in gateway namespace
   - Takes ~2-5 minutes

2. **Gateways** will use new certificate
   - Both internal and external gateways
   - TLS termination with Let's Encrypt cert

3. **Cilium** will announce new IPs
   - Gateway external: `192.168.86.49`
   - Gateway internal: `192.168.86.50`
   - L2 announcements on your LAN

4. **ExternalDNS** will create Cloudflare records
   - Within ~1 minute after sync
   - 6 A records pointing to `192.168.86.50`
   - TXT records for ownership tracking

### Within 5 Minutes

5. **DNS propagates** globally
   - Internal clients resolve to `192.168.86.50` ✅
   - External clients resolve to `192.168.86.50` (can't reach - private IP)

6. **HTTPS works** on all services
   - Valid Let's Encrypt certificate
   - No browser warnings
   - Auto-renews every 60 days

## Verification Steps

### 1. Check Certificate
```bash
# Wait for certificate to be issued
kubectl get certificate -n gateway
# Should show: jrb-nz-wildcard   True

# Check secret exists
kubectl get secret cert-jrbnz -n gateway
```

### 2. Check DNS Records
```bash
# Test DNS resolution
dig argocd.jrb.nz
dig longhorn.jrb.nz

# Should return: 192.168.86.50

# Check in Cloudflare Dashboard
# DNS → jrb.nz zone
# Look for A records with gray cloud (not proxied)
```

### 3. Check ExternalDNS
```bash
# View logs
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns

# Should see:
# - "Creating record" messages
# - No errors about gateway filters
```

### 4. Test HTTPS Access
```bash
# From your local network
curl -k https://argocd.jrb.nz
curl -k https://longhorn.jrb.nz

# Or visit in browser (should show valid cert)
```

### 5. Check Gateway IPs
```bash
# Gateway services should have new IPs
kubectl get gateway -n gateway

# Check L2 announcements
kubectl get ciliuml2announcementpolicy -A

# Ping from local network
ping 192.168.86.50
```

## DNS Records Created in Cloudflare

After sync, you should see these in Cloudflare DNS:

| Name | Type | Value | Proxied | TTL |
|------|------|-------|---------|-----|
| argocd.jrb.nz | A | 192.168.86.50 | No | Auto |
| longhorn.jrb.nz | A | 192.168.86.50 | No | Auto |
| redis.jrb.nz | A | 192.168.86.50 | No | Auto |
| homepage.jrb.nz | A | 192.168.86.50 | No | Auto |
| garage.jrb.nz | A | 192.168.86.50 | No | Auto |
| echo.jrb.nz | A | 192.168.86.50 | No | Auto |

Plus TXT records for ownership tracking (prefix: `external-dns-`)

## Adding New Services

For any new HTTPRoute, just add one label:

```yaml
metadata:
  labels:
    external-dns: "true"  # ← Add this
```

ExternalDNS will automatically:
1. Detect the new HTTPRoute
2. Extract hostname (e.g., `newapp.jrb.nz`)
3. Create Cloudflare A record → `192.168.86.50`
4. Update within ~1 minute

## Security Notes

**Publishing Private IPs in Public DNS is Safe:**
- RFC1918 addresses (192.168.x.x) are non-routable on internet
- External clients resolve DNS but can't reach the IP
- Internal clients resolve DNS and CAN reach the IP
- No sensitive information leaked

**Certificate Security:**
- Wildcard cert stored in Kubernetes secret
- Only gateway namespace has access
- Auto-rotated every 60 days
- Private key never leaves cluster

## Documentation Created

- `infrastructure/networking/gateway/DNS-AUTOMATION.md` - Complete DNS automation guide
- `CHANGES-SUMMARY.md` - This file

## Files Modified

**Infrastructure:**
- infrastructure/networking/gateway/certificate.yaml (NEW)
- infrastructure/networking/gateway/gw-internal.yaml
- infrastructure/networking/gateway/gw-external.yaml
- infrastructure/networking/gateway/kustomization.yaml
- infrastructure/networking/cilium/ip-pool.yaml
- infrastructure/controllers/external-dns/values.yaml
- infrastructure/storage/csi-driver-smb/storage-class.yaml (6 storage classes)
- infrastructure/storage/csi-driver-nfs/values.yaml
- infrastructure/storage/longhorn/backup-settings.yaml
- infrastructure/database/redis/redis-commander/service.yaml
- infrastructure/database/redis/redis-commander/httproute.yaml
- infrastructure/database/redis/redis-instance/service.yaml
- infrastructure/database/cloudnative-pg/paperless/cluster.yaml
- infrastructure/database/cloudnative-pg/khoj/cluster.yaml

**Apps:**
- infrastructure/controllers/argocd/http-route.yaml
- my-apps/media/homepage-dashboard/httproute.yaml
- my-apps/media/garage-webui/http-route.yaml
- my-apps/media/echo-server/httproute.yaml

**Scripts:**
- scripts/verify-worker-disks.sh
- scripts/trigger-immediate-backups.sh

**Documentation:**
- README.md
- infrastructure/networking/README.md
- infrastructure/networking/gateway/DNS-AUTOMATION.md (NEW)

## Next Steps

1. **Commit and push** all changes
2. **Wait for ArgoCD** to sync (or manually sync)
3. **Monitor certificate** issuance (~2-5 min)
4. **Verify DNS** records in Cloudflare
5. **Test HTTPS** access from local network
6. **Check ExternalDNS** logs for any errors

## Rollback Plan

If issues occur:

1. **Certificate problems:**
   ```bash
   kubectl delete certificate jrb-nz-wildcard -n gateway
   kubectl delete secret cert-jrbnz -n gateway
   ```

2. **DNS problems:**
   - Remove `external-dns: "true"` labels from HTTPRoutes
   - Manually delete DNS records in Cloudflare
   - Wait for ExternalDNS to clean up TXT records

3. **Gateway IP problems:**
   - Revert `infrastructure/networking/cilium/ip-pool.yaml`
   - Revert gateway IP in `gw-internal.yaml`
   - Restart gateway pods

## Support

- ExternalDNS docs: https://kubernetes-sigs.github.io/external-dns/
- cert-manager docs: https://cert-manager.io/docs/
- Gateway API docs: https://gateway-api.sigs.k8s.io/
- Cilium docs: https://docs.cilium.io/
