# Branch CUPS Helm Chart

This Helm chart deploys CUPS (Common Unix Printing System) servers across multiple branch locations with location-specific configurations.

## Structure

```
branch-cups-helm/
├── Chart.yaml                      # Helm chart metadata
├── values.yaml                     # Default values
├── README.md                       # This file
├── charts/
│   └── branch-cups/               # Main chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── _helpers.tpl       # Template helpers
│           ├── deployment.yaml    # Kubernetes Deployment
│           ├── service.yaml       # Kubernetes Service (LoadBalancer)
│           └── configmap.yaml     # CUPS configuration
└── values/
    ├── redhill-values.yaml        # Redhill location config
    └── rockymount-values.yaml     # Rocky Mount location config
```

## Container Image

- **Repository:** `chmeehan/branch-cups-server`
- **Tag:** `v1.1`
- **Architecture:** `amd64` (cross-compiled for k3s and Palette Edge)

## Deployment

### Option 1: Using Helm CLI (Local/Direct)

```bash
# Install for Redhill location
helm install branch-cups-redhill ./charts/branch-cups \
  -f values/redhill-values.yaml

# Install for Rocky Mount location
helm install branch-cups-rockymount ./charts/branch-cups \
  -f values/rockymount-values.yaml
```

### Option 2: Using Spectro Cloud Palette

In Palette Console:
1. Create a new **Cluster Profile**
2. Add a **Pack** with type **Helm**
3. Configure the Helm source:
   - **Repository URL:** `https://github.com/yourusername/branch-cups-helm-repo/raw/main/charts`
   - **Chart Name:** `branch-cups`
   - **Chart Version:** `1.0.0`
4. Override values with location-specific config:
   ```yaml
   # For Redhill
   -f values/redhill-values.yaml
   ```

### Option 3: Using Kubernetes Manifests

Generate manifests and apply directly:

```bash
# Generate Redhill manifests
helm template branch-cups-redhill ./charts/branch-cups \
  -f values/redhill-values.yaml > redhill-manifests.yaml

# Apply to cluster
kubectl apply -f redhill-manifests.yaml
```

## Configuration

### Location-Specific Network Access

Each location has unique network requirements configured in `/values/*.yaml`:

#### Redhill
- **Local Subnet:** `192.168.100.0/24`
- **Regional Network:** `10.100.0.0/16`

#### Rocky Mount
- **Local Subnet:** `192.168.200.0/24`
- **Regional Network:** `10.200.0.0/16`

### Modifying CUPS Configuration

Edit the `cupsConfig` section in location values files to:
- Add/remove allowed networks
- Adjust log levels
- Configure printer access permissions
- Modify listening ports

Example:
```yaml
cupsConfig: |
  LogLevel warn
  Listen localhost:631
  <Location />
    Order allow,deny
    Allow localhost
    Allow 192.168.100.0/24
  </Location>
```

## Service Access

The service is exposed as a **LoadBalancer** type, which integrates with:
- **Metallb** (on k3s)
- **Spectro Cloud Palette** (on Edge clusters)

Once deployed, access the CUPS web interface:
- **URL:** `http://<LoadBalancer-IP>:631`
- **SSH:** Port 22 (credentials in Dockerfile: `root:root`)

## Upgrading

To update configuration for a location:

```bash
# Redhill configuration update
helm upgrade branch-cups-redhill ./charts/branch-cups \
  -f values/redhill-values.yaml

# Or update and deploy new image version
helm upgrade branch-cups-redhill ./charts/branch-cups \
  -f values/redhill-values.yaml \
  --set image.tag=v1.2
```

## Uninstalling

```bash
# Remove Redhill deployment
helm uninstall branch-cups-redhill

# Remove Rocky Mount deployment
helm uninstall branch-cups-rockymount
```

## Adding New Locations

To add a 3rd location (e.g., "atlanta"):

1. Create new values file: `values/atlanta-values.yaml`
2. Copy from existing location, adjust networks:
   ```yaml
   cupsConfig: |
     # Atlanta network: 192.168.50.0/24, 10.50.0.0/16
     Allow 192.168.50.0/24
     Allow 10.50.0.0/16
   ```
3. Deploy:
   ```bash
   helm install branch-cups-atlanta ./charts/branch-cups \
     -f values/atlanta-values.yaml
   ```

## Resources

- **CPU Limit:** 500m
- **Memory Limit:** 512Mi
- **Requests:** 250m CPU, 256Mi Memory

Adjust in location values files if needed for high-volume printing environments.

## Troubleshooting

Check pod logs:
```bash
kubectl logs deployment/branch-cups-redhill
```

Verify service:
```bash
kubectl get svc branch-cups-redhill
```

Access CUPS configuration inside pod:
```bash
kubectl exec -it deployment/branch-cups-redhill -- bash
cat /etc/cups/cupsd.conf
```

## Notes

- Default SSH credentials: `root:root` (change in production)
- CUPS runs in foreground mode (`cupsd -f`) for proper container operation
- LoadBalancer IP assignment depends on MetalLB or cloud provider
