# Branch CUPS Multi-Location Deployment

**Spectro Cloud Palette Edge - Proof of Value**

A production-ready Helm-based deployment for CUPS printing servers across distributed edge locations (k3s + Palette Edge). This PoV demonstrates multi-location management with environment-specific configurations.

## 🎯 PoV Objectives

- ✅ Deploy CUPS servers to 2+ Palette Edge clusters (Redhill, Rocky Mount)
- ✅ Manage location-specific configurations (network access, permissions)
- ✅ Use MetalLB for LoadBalancer service exposure
- ✅ Demonstrate scalability to 10 locations
- ✅ Show GitOps workflow (config-as-code)
- ✅ Enable remote management via Palette console

## 📊 Current Status

| Location | Status | Network | Cluster | Deployed |
|----------|--------|---------|---------|----------|
| Redhill | 🔵 Ready | 192.168.100.0/24, 10.100.0.0/16 | k3s + Palette | ⏳ Pending |
| Rocky Mount | 🔵 Ready | 192.168.200.0/24, 10.200.0.0/16 | k3s + Palette | ⏳ Pending |

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│         Spectro Cloud Palette Console                   │
│   (Central management across all locations)            │
└────────────────┬────────────────────────────────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
    ┌───────┐┌───────┐┌───────┐
    │Redhill││Rocky  ││Future │
    │Edge   ││Mount  ││Sites  │
    │Cluster││Cluster│└───────┘
    └───────┘└───────┘
        │        │
    ┌───▼─┐  ┌───▼─┐
    │CUPS │  │CUPS │
    │Pod  │  │Pod  │
    └─────┘  └─────┘
        │        │
    MetalLB LoadBalancer (assigns external IPs)
        │        │
  192.168.100.X 192.168.200.X (floating IPs per location)
```

## 🚀 Quick Start

### Prerequisites
- Spectro Cloud Palette account with 2+ edge clusters
- MetalLB configured on each cluster
- kubectl or Palette UI access
- Docker Hub account (`chmeehan/branch-cups-server:v1.1`)

### Step 1: Add Docker Hub Registry to Palette
1. Settings → Integrations → Add OCI Registry
2. **Name:** `docker-hub`
3. **URL:** `https://index.docker.io`
4. **Credentials:** Docker Hub username + PAT token

### Step 2: Create Cluster Profile
1. Clusters → Cluster Profiles → New Profile
2. Add Pack → Helm
3. Configure:
   - **Helm Chart URL:** `https://github.com/YOUR-USERNAME/branch-cups-helm`
   - **Chart:** `charts/branch-cups`
   - **Version:** `1.0.0`

### Step 3: Deploy to Redhill
1. Select cluster: `redhill-palette-edge`
2. Override values with:
   ```yaml
   -f values/redhill-values.yaml
   ```
3. Deploy

### Step 4: Deploy to Rocky Mount
1. Select cluster: `rockymount-palette-edge`
2. Override values with:
   ```yaml
   -f values/rockymount-values.yaml
   ```
3. Deploy

### Verify Deployment
```bash
kubectl get pods -n default | grep branch-cups
kubectl get svc | grep branch-cups

# Access CUPS UI
# Redhill: http://<METALLB-IP-1>:631
# Rocky Mount: http://<METALLB-IP-2>:631
```

## 📁 Repository Structure

## 📁 Repository Structure

```
branch-cups-helm/
├── README.md                          # This file - PoV overview
├── Chart.yaml                         # Root chart metadata (optional)
├── charts/
│   └── branch-cups/                   # Main Helm chart
│       ├── Chart.yaml                 # Chart version/metadata
│       ├── values.yaml                # Default values
│       └── templates/
│           ├── _helpers.tpl           # Reusable template functions
│           ├── deployment.yaml        # Pod definition
│           ├── service.yaml           # LoadBalancer service
│           └── configmap.yaml         # CUPS config injection
└── values/
    ├── redhill-values.yaml            # Redhill location config
    └── rockymount-values.yaml         # Rocky Mount location config
```

## 🔧 Configuration by Location

### Redhill
**File:** [values/redhill-values.yaml](values/redhill-values.yaml)
- **Local Network:** `192.168.100.0/24` (branch LAN)
- **Regional Network:** `10.100.0.0/16` (wide area)
- **Printers:** [Document your printer list here]
- **Admin Users:** [Document here]
- **Print Quota:** [TBD]

### Rocky Mount
**File:** [values/rockymount-values.yaml](values/rockymount-values.yaml)
- **Local Network:** `192.168.200.0/24` (branch LAN)
- **Regional Network:** `10.200.0.0/16` (wide area)
- **Printers:** [Document your printer list here]
- **Admin Users:** [Document here]
- **Print Quota:** [TBD]

## 📝 Configuration as Code

All CUPS settings are in `cupsConfig` blocks:

```yaml
# From values/redhill-values.yaml
cupsConfig: |
  LogLevel warn
  Listen localhost:631
  <Location />
    Order allow,deny
    Allow localhost
    Allow 192.168.100.0/24   # ← Redhill network
    Allow 10.100.0.0/16
  </Location>
```

**To modify:**
1. Edit the location values file
2. Commit to git
3. `git push`
4. Palette auto-detects changes (or manually sync)
5. Pod restarts with new config

## 🛠️ Management Tasks

### Deploy New Location (e.g., Atlanta)

1. Create `values/atlanta-values.yaml`:
   ```yaml
   cupsConfig: |
     Allow 192.168.50.0/24
     Allow 10.50.0.0/16
   ```

2. Commit and push:
   ```bash
   git add values/atlanta-values.yaml
   git commit -m "Add Atlanta location config"
   git push
   ```

3. In Palette, deploy to Atlanta cluster with `-f values/atlanta-values.yaml`

### Update CUPS Version

1. Update image tag in chart values:
   ```yaml
   image:
     tag: "v1.2"
   ```

2. Commit and push:
   ```bash
   git commit -am "Upgrade CUPS to v1.2"
   git push
   ```

3. In Palette, trigger deployment sync

### Access CUPS Admin

SSH into any pod:
```bash
kubectl exec -it deployment/branch-cups-redhill -- bash

# Then check config
cat /etc/cups/cupsd.conf

# Or manage printers
lpstat -p
```

## 📊 Monitoring & Logging

### View Pod Logs
```bash
# Redhill
kubectl logs -f deployment/branch-cups-redhill

# Rocky Mount
kubectl logs -f deployment/branch-cups-rockymount
```

### Check Service Status
```bash
# Get LoadBalancer IPs
kubectl get svc

# Test connectivity
curl http://<LOADBALANCER-IP>:631/
```

## 🔐 Security Considerations

**Current Setup (Development):**
- ⚠️ SSH default credentials: `root:root`
- ⚠️ CUPS admin access: No authentication in current config
- ⚠️ Network access: Allow rules only (no deny)

**For Production:**
- [ ] Use strong SSH keys (update Dockerfile)
- [ ] Enable CUPS authentication with user directory
- [ ] Implement deny rules to restrict access
- [ ] Use network policies for pod-to-pod communication
- [ ] Enable audit logging
- [ ] Use secrets for credentials (avoid hardcoding)

See [SECURITY.md](SECURITY.md) for detailed hardening guide.

## 📈 Scaling to 10 Locations

To scale from 2 to 10 locations:

1. **Repeat the process:**
   ```
   values/location-3-values.yaml
   values/location-4-values.yaml
   ... up to location-10
   ```

2. **Organize by region (optional):**
   ```
   values/us-east/
     ├── redhill-values.yaml
     ├── atlanta-values.yaml
   values/us-west/
     ├── denver-values.yaml
   ```

3. **Use Palette UI:**
   - Create cluster profile per location
   - Reference same Helm chart
   - Override values per location

4. **Centralized management:**
   - All configurations in git
   - Single source of truth
   - Version history
   - Easy rollback

## 🐳 Container Image

**Build & Push (amd64 architecture):**
```bash
cd branch-cups-server/
docker buildx build --platform linux/amd64 -t chmeehan/branch-cups-server:v1.1 --push .
```

**Verify in Docker Hub:**
```
https://hub.docker.com/r/chmeehan/branch-cups-server
```

Image includes:
- Ubuntu 22.04 base
- CUPS server
- OpenSSH server
- Net-tools for diagnostics

## 🧪 Testing Checklist

- [ ] **Deployment:**
  - [ ] Pod starts successfully
  - [ ] ConfigMap mounted correctly
  - [ ] Service gets LoadBalancer IP (MetalLB)

- [ ] **CUPS Functionality:**
  - [ ] Web UI accessible (`http://IP:631`)
  - [ ] SSH access works
  - [ ] Printers can be added
  - [ ] Print jobs process

- [ ] **Network Access:**
  - [ ] Local network clients can connect
  - [ ] Regional network clients can connect
  - [ ] Non-allowed networks blocked

- [ ] **Config Management:**
  - [ ] Edit values file
  - [ ] Push to git
  - [ ] Palette picks up changes
  - [ ] Pod restarts with new config

## 📖 Documentation

- [Helm Chart Details](charts/branch-cups/README.md) - Template structure
- [SECURITY.md](SECURITY.md) - Production hardening
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues
- [SCALABILITY.md](SCALABILITY.md) - 10-location design

## 🤝 Contributing

To add or modify locations:
1. Create feature branch: `git checkout -b feature/location-name`
2. Add values file: `values/location-values.yaml`
3. Test locally: `helm template --values values/location-values.yaml`
4. Commit & push
5. PR review

## 📞 Support

**Issues? Check:**
1. Pod logs: `kubectl logs deployment/branch-cups-<location>`
2. Service status: `kubectl get svc branch-cups-<location>`
3. CUPS config: `kubectl exec ... -- cat /etc/cups/cupsd.conf`
4. MetalLB pool: `kubectl get ipaddresspool` (MetalLB should have IPs available)

## 📅 Roadmap

**Phase 1 (Current):**
- [x] Helm chart created
- [x] 2 locations (Redhill, Rocky Mount)
- [ ] Deploy to Palette Edge clusters
- [ ] Verify MetalLB integration
- [ ] Document network flows

**Phase 2:**
- [ ] Add 8 more locations
- [ ] Printer auto-discovery per location
- [ ] Central print queue monitoring
- [ ] Load balancing across CUPS instances

**Phase 3:**
- [ ] Print job audit logging
- [ ] Cost allocation by location
- [ ] Automated backup of printer configs
- [ ] Integration with Spectro Cloud billing

## 📄 License

Stauffer x Vesper - Internal Use Only
