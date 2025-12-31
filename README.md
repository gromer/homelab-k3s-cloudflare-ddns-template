# homelab-k3s-cloudflare-ddns

A template repository for Kubernetes infrastructure as code to run a Cloudflare Dynamic DNS (DDNS) updater on a k3s cluster. This solution automatically updates your Cloudflare DNS A records with your current public IP address, perfect for homelab environments with dynamic IPs.

## About This Template

This is a **GitHub template repository**. Use this template to create your own repository with pre-configured Kubernetes manifests for deploying a Cloudflare DDNS service.

### How to Use This Template

1. Click the green **"Use this template"** button at the top of the repository
2. Choose a name for your new repository (e.g., `my-homelab-ddns`)
3. Clone your newly created repository to your local machine
4. Follow the [Deployment Guide](#deployment-guide) below to configure and deploy to your k3s cluster

For more details on using GitHub template repositories, see the [GitHub documentation](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template).

## Features

- ✅ Runs as a Kubernetes CronJob (hourly updates)
- ✅ Supports multiple domains/subdomains
- ✅ Automatically creates DNS records if they don't exist
- ✅ Lightweight curl-based implementation
- ✅ Configurable via ConfigMaps and Secrets
- ✅ Color-coded output for easy monitoring

## Prerequisites

- A k3s (or any Kubernetes) cluster running on Raspberry Pi or any other hardware
- A Cloudflare account with a domain
- Cloudflare API Token with DNS edit permissions
- `kubectl` configured to access your cluster

## Deployment Guide

### 1. Create Your Repository from the Template

If you haven't already, use the **"Use this template"** button to create your own repository from this template.

Then clone your new repository:

```bash
git clone https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
cd YOUR-REPO-NAME
```

### 2. Get Your Cloudflare Credentials

#### API Token (Recommended)
1. Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click "Create Token"
3. Use the "Edit zone DNS" template or create a custom token with:
   - Permissions: `Zone` → `DNS` → `Edit`
   - Zone Resources: `Include` → `Specific zone` → Select your zone
4. Copy the generated token

#### Zone ID
1. Go to your domain's dashboard on Cloudflare
2. Scroll down on the overview page
3. Find your Zone ID in the right sidebar

### 3. Configure Your Setup

#### Edit the ConfigMap

Edit `k8s/configmap.yaml` to configure your domains and zone ID:

```yaml
data:
  # Single domain
  DOMAINS: "vpn.example.com"
  
  # Multiple domains (comma-separated)
  # DOMAINS: "vpn.example.com,home.example.com,nas.example.com"
  
  # Your Cloudflare Zone ID
  ZONE_ID: "your-zone-id-here"
```

#### Create the Secret

Copy the secret template and add your API token:

```bash
cp k8s/secret.yaml.template k8s/secret.yaml
```

Edit `k8s/secret.yaml` and replace `your-cloudflare-api-token-here` with your actual API token:

```yaml
stringData:
  API_TOKEN: "your-actual-cloudflare-api-token"
```

**⚠️ Important:** Never commit `k8s/secret.yaml` to version control! The template file is provided for convenience.

### 4. Deploy to Kubernetes

#### Option A: Using Kustomize

```bash
# Make sure you've created k8s/secret.yaml from the template
# and added it to kustomization.yaml resources

kubectl apply -k k8s/
```

#### Option B: Using kubectl

```bash
# Apply all manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/script-configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/cronjob.yaml
```

### 5. Verify Deployment

```bash
# Check if everything is created
kubectl get all -n cloudflare-ddns

# Expected output:
# NAME                              SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# cronjob.batch/cloudflare-ddns     0 * * * *   False     0        <none>          10s
```

### 6. Test the Setup

Create a manual job to test immediately:

```bash
# Create a test job from the cronjob
kubectl create job --from=cronjob/cloudflare-ddns manual-test -n cloudflare-ddns

# Wait a few seconds, then check the logs
kubectl logs -n cloudflare-ddns job/manual-test

# Expected output should show:
# Starting Cloudflare DDNS update...
# Timestamp: ...
# Fetching current public IP address...
# Current public IP: X.X.X.X
# Processing domain: vpn.example.com
# ...
# Successfully updated DNS record for vpn.example.com to X.X.X.X
```

### 7. Clean Up Test Job

```bash
kubectl delete job manual-test -n cloudflare-ddns
```

## Configuration

### Schedule

The CronJob runs every hour by default. To change the schedule, edit the `schedule` field in `k8s/cronjob.yaml`:

```yaml
spec:
  # Run every hour (default)
  schedule: "0 * * * *"
  
  # Run every 30 minutes
  # schedule: "*/30 * * * *"
  
  # Run every 6 hours
  # schedule: "0 */6 * * *"
  
  # Run daily at 2 AM
  # schedule: "0 2 * * *"
```

### DNS Record Settings

By default, DNS records are created with:
- TTL: 120 seconds (2 minutes)
- Proxied: false (DNS-only mode)

To change these settings, edit the `update-dns.sh` script in `k8s/script-configmap.yaml`.

### Multiple Domains Example

If you want to update multiple subdomains, configure your `configmap.yaml` like this:

```yaml
data:
  DOMAINS: "vpn.example.com,home.example.com,nas.example.com,plex.example.com"
  ZONE_ID: "your-zone-id"
```

The script will process each domain sequentially and update their A records to point to your current public IP.

## Monitoring

### View Recent Jobs

```bash
kubectl get jobs -n cloudflare-ddns
```

### View Logs from Latest Job

```bash
kubectl logs -n cloudflare-ddns -l app=cloudflare-ddns --tail=50
```

### Check CronJob Status

```bash
kubectl describe cronjob cloudflare-ddns -n cloudflare-ddns
```

## Troubleshooting

### Check if the CronJob is Running

```bash
kubectl get cronjob -n cloudflare-ddns
```

### View Failed Jobs

```bash
kubectl get jobs -n cloudflare-ddns --field-selector status.successful!=1
```

### Check Pod Logs for Errors

```bash
kubectl logs -n cloudflare-ddns -l app=cloudflare-ddns --tail=100
```

### Common Issues

1. **"API_TOKEN is not set" error**: Make sure you created `secret.yaml` from the template and applied it
2. **"Failed to fetch DNS record" error**: Verify your Zone ID is correct
3. **"Failed to update DNS record" error**: Check that your API token has the correct permissions
4. **No jobs running**: The first job will run at the top of the next hour. Use the manual test to verify immediately.

## Security Best Practices

1. **Never commit secrets**: The `secret.yaml` file should never be committed to version control
2. **Use minimal API token permissions**: Only grant DNS edit permissions for specific zones
3. **Rotate tokens regularly**: Periodically regenerate your Cloudflare API tokens
4. **Use Sealed Secrets or External Secrets**: For production, consider using [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [External Secrets Operator](https://external-secrets.io/)

## Architecture

```
┌─────────────────────────────────────┐
│         k3s Cluster                 │
│                                     │
│  ┌───────────────────────────────┐  │
│  │  cloudflare-ddns namespace    │  │
│  │                               │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  CronJob                │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │  1. Get Public IP │  │  │  │
│  │  │  │  2. Query CF API  │  │  │  │
│  │  │  │  3. Update DNS    │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  │                               │  │
│  │  ConfigMap: Domain config     │  │
│  │  Secret: API credentials      │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
              │
              ▼
    ┌──────────────────┐
    │  Cloudflare API  │
    └──────────────────┘
```

## Project Structure

```
.
├── .gitignore
├── LICENSE
├── README.md
├── k8s/
│   ├── namespace.yaml          # Kubernetes namespace
│   ├── configmap.yaml          # Domain and zone configuration
│   ├── script-configmap.yaml   # Update script as ConfigMap
│   ├── secret.yaml.template    # Template for API credentials
│   ├── cronjob.yaml            # CronJob definition
│   └── kustomization.yaml      # Kustomize configuration
└── scripts/
    └── update-dns.sh           # Standalone update script (for reference)
```

## Contributing

Suggestions and improvements are welcome! Feel free to open issues or submit pull requests.

## License

See [LICENSE](LICENSE) file for details.