# Ansible GitOps K8s Project 

This project aims to easily deploy RKE2 Cluster via Ansible. With various applications and services, following GitOps principles. 

Initial deployment is done via Ansible playbooks after that all apps are managed from ArgoCD.

## Overview

### What is GitOps?

GitOps is a modern approach to managing infrastructure and applications where:

- **Git is the single source of truth**: All configuration is stored in Git repositories
- **Declarative configuration**: The desired state of the system is explicitly defined
- **Continuous reconciliation**: Automated processes ensure the actual state matches the desired state
- **Pull-based deployment**: Agents in the cluster pull changes from Git repositories

In this project, we use ArgoCD as our GitOps engine to automatically deploy and manage applications from Git repositories, ensuring your Kubernetes cluster always reflects the state defined in your code.

### Apps

| Category | Applications |
|----------|-------------|
| Infrastructure | - KubeVIP (LoadBalancer)<br>- Traefik (Ingress Controller)<br>- Longhorn (Distributed Block Storage)<br>- CloudNativePG (Central PostgreSQL)<br>- Dragonfly (Central Redis Cache)<br>- Garage (Self-hosted S3 Object Store)<br>- Velero (Backup & Restore) |
| GitOps & Management | - ArgoCD (GitOps CD)<br>- Rancher (Kubernetes Management)<br>- Gitea (Source Control)<br>- Renovate (Dependency Automation)|
| Security & Access | - Authentik (SSO/IAM)<br>- Tailscale (VPN)<br>- cert-manager (TLS Certificates) |
| Monitoring & Observability | - Prometheus (Metrics)<br>- Grafana (Visualization)<br>- Loki (Log Aggregation)<br>- Alloy (Log Collection) |
| Media & Entertainment | - Jellyfin (Media Server)<br>- Jellyseerr (Media Requests)<br>- Radarr (Movie Management)<br>- Sonarr (TV Management)<br>- Prowlarr (Indexer Manager)<br>- qBittorrent (Download Client)<br>- Audiobookshelf (Audiobook Server) |
| Productivity & Organization | - Paperless-ngx (Document Management)<br>- Donetick (Task Management)<br>- Joplin (Note Taking)<br>- Mealie (Recipe Manager)<br>- Recipya (Recipe Manager)<br>- Lubelogger (Vehicle Maintenance) |
| Home & IoT | - Home Assistant (Home Automation)<br>- Homepage (Dashboard) |
| Development & AI | - n8n (Workflow Automation)<br>- Open-WebUI (AI Interface) |
| Photo & Media Management | - Immich (Photo Management)<br>- Karakeep (Media Organizer) |

### Features

- 🚀 Single command cluster deployment
- 🔄 GitOps-based application deployment and lifecycle management
- 📊 Comprehensive monitoring and logging stack
- 🔐 Integrated SSO with Authentik
- 💾 Distributed storage with Longhorn
- 🗄️ Centralized data layer with CloudNativePG (PostgreSQL) and Dragonfly (Redis caching)
- 🪣 Self-hosted S3 object storage via Garage for PostgreSQL WAL archiving and base backups
- 📁 External storage support — SFTP, NFS, and SMB mounts exposed as hostPath PVs
- 🗄️ PostgreSQL continuous backups (WAL archiving) to Garage or any cloud S3 provider
- 📈 Pre-configured Grafana dashboards for:
  - Application logs and metrics
  - Error monitoring
  - System performance
  - Storage metrics
- 🔄 High availability cluster configuration
- 🔒 TLS encryption for all services with automatic certificate management
- 🔒 Secured Access remotely via Tailscale
- 🔄 Automated dependency updates with Renovate

## Architecture

### Deployment Flow

The deployment follows a structured, multi-phase approach:

1. **Local Setup** (`01_local`) - Install Ansible collections and configure vault
2. **User Creation** (`02_create_user`) - Create ansible user on target nodes
3. **Base Configuration** (`03_base_config`) - System hardening and preparation
4. **RKE2 Cluster** (`04_rke2_server`) - Deploy Kubernetes with KubeVIP load balancer
5. **Infrastructure** (`05_infra`) - Deploy core services (Traefik, Longhorn, Authentik, PostgreSQL, Redis)
6. **ArgoCD** (`06_argocd`) - Deploy GitOps engine and application definitions
7. **Applications** (`07_apps`) - Configure specific applications (Grafana dashboards, etc.)

**Deployment time:** Approximately 20-30 minutes for full cluster setup.

### Data Layer Architecture

The project uses a centralized data layer approach:

**CloudNativePG (PostgreSQL):**
- Centralized PostgreSQL cluster for all applications
- High availability with automatic failover
- Applications include: Gitea, Authentik, Paperless-ngx, Immich, n8n, and more
- Automated backups and point-in-time recovery

**Dragonfly (Redis):**
- Centralized Redis-compatible cache cluster
- High-performance in-memory caching
- Used by: ArgoCD, Paperless-ngx, and other applications
- Drop-in replacement for Redis with better performance

This architecture eliminates the need for each application to run its own database, reducing resource usage and simplifying backup procedures.

### Object Storage & PostgreSQL Backup

**Garage (Self-hosted S3):**
- S3-compatible object store running as a single-node StatefulSet on the first master
- Used for continuous PostgreSQL WAL archiving and daily base backups via CloudNativePG's barman integration
- Metadata stored on Longhorn; data stored on any external mount (SFTP, NFS, etc.)
- No public Helm chart — deployed directly from the Garage git repository by Ansible

**PostgreSQL backup supports two backends:**
- `type: garage` — bootstrapped automatically (bucket + access key created, CNPG cluster patched)
- `type: s3` — any S3-compatible provider (AWS, Backblaze B2, Cloudflare R2, etc.)

Enable and configure in `group_vars/all/main.yml`:
```yaml
deploy_garage: true

garage:
  namespace: "garage"
  region: "garage"
  pg_backup_bucket: "pg-backups"
  pg_backup_key_name: "cnpg-backup"
  s3_endpoint: "http://garage.garage.svc.cluster.local:3900"
  meta:
    size: "1Gi"
    storage_class: "longhorn"
  data:
    size: "200Gi"   # backed by external storage mount

postgresql:
  backup:
    enabled: true
    schedule: "0 2 * * *"
    storage:
      type: "garage"   # or "s3" for cloud providers
```

```bash
# Deploy Garage
ansible-playbook playbooks/main.yaml --tags infra:garage

# Bootstrap backups (create bucket/key, patch CNPG cluster)
ansible-playbook playbooks/main.yaml --tags apps:garage
```

### External Storage

SFTP, NFS, and SMB/CIFS external mounts are deployed as DaemonSets and exposed as hostPath PVs:

```yaml
external_storage:
  sftp:
    host: "your-storage-host"
    port: 23
    user: "storagebox"
    identity_file: "~/.ssh/id_rsa"
  applications:
    - name: myapp
      sftp:
        remote_path: "/path/on/remote"
        node: "master-01"        # pin to single node (optional)
      mount_path: "/mnt/myapp-storage"
```

```bash
# Deploy external storage mounts
ansible-playbook playbooks/main.yaml --tags infra:externalstorage
```

### Network Architecture

- **KubeVIP**: Provides a virtual IP for cluster API access
- **Traefik**: Ingress controller with automatic TLS via cert-manager
- **Tailscale Operator**: Secure VPN access via Kubernetes Operator with subnet router (no auth key rotation needed)
- **Authentik**: Central SSO/IAM for unified authentication across all services

## Prerequisites

### Hardware Requirements
- **Control Node:**
  - Linux system/VM or WSL2 on Windows

- **Cluster Nodes:**
  - Ubuntu Server 24.04
  - Minimum 16GB RAM per node
  - Recommended: 3 nodes for HA
  - [Low-cost mini PC options](https://www.lowcostminipcs.com/)

### Network Requirements
- SSH access to all nodes (root SSH keys)
- DOMAIN name (Recommended: [Porkbun](https://porkbun.com/) or [DuckDNS](https://www.duckdns.org/))

### Optional
- Cloudflare account and API token (for TLS certificates)
- NFS share for media storage and backups
- [Tailscale account](https://login.tailscale.com/admin/settings/oauth) (OAuth client, not auth key)

## Quick Start

1. **Clone and prepare configuration**
   ```bash
   git clone https://github.com/rtomik/ansible-gitops-k8s.git && cd ansible-gitops-k8s && \
   mv inventory.yml_ex inventory.yml && \
   mv group_vars/all/main.yml_ex group_vars/all/main.yml
   ```

2. **Configure nodes**
   - Update `inventory.yml` with node IPs
   - Modify `group_vars/all/main.yml` with your configurations
   - Set required variables in `Required variables` section

3. **Initial Deployment**
   NOTE: If you enabled cloudflare it can take some time to propagate new TLS certificate
   ```
   ansible-playbook playbooks/main.yml
   ```
   Kubeconfig will be stored in playbook dir
   ```
   k get certificate -A
   ```
   Check cert-manager pod logs for any error
   ```
   CERT_MANAGER_POD=$(kubectl get pods -n cert-manager -l app.kubernetes.io/name=cert-manager -o jsonpath='{.items[0].metadata.name}')
   k logs $CERT_MANAGER_POD -n cert-manager -f
   ```
   On error
   ```
   Error cleaning up challenge: while querying the Cloudflare API for DELETE
   ```
   Delete TXT _acme-challenge records in cloudflare
   Delete certificate from kubernetes and retry then ansible playbook

4. **To deploy specific components, use tags**
   ```bash
   ansible-playbook playbooks/main.yml --tags infra
   ansible-playbook playbooks/main.yaml --tags infra:garage      # Garage S3 storage
   ansible-playbook playbooks/main.yaml --tags apps:garage       # Bootstrap PostgreSQL backups
   ansible-playbook playbooks/main.yaml --tags infra:tailscale   # Tailscale VPN operator
   ansible-playbook playbooks/main.yaml --tags infra:externalstorage  # SFTP/NFS/SMB mounts
   ansible-playbook playbooks/main.yaml --tags infra:etcd-backup # etcd backup system
   ```

5. **To destroy the cluster and remove everything**
   ```
   ansible-playbook playbooks/destroy.yml
   ```

## Post Deployment

### Access Services

| Service | URL | Description |
|---------|-----|-------------|
| Rancher | `https://rancher.<DOMAIN>` | Kubernetes Management UI |
| ArgoCD | `https://argocd.<DOMAIN>` | GitOps Control Panel |
| Grafana | `https://grafana.<DOMAIN>` | Metrics & Logs Visualization |
| Authentik | `https://authentik.<DOMAIN>` | SSO/IAM Portal |
| Prometheus | `https://prometheus.<DOMAIN>` | Metrics Storage |
| Gitea | `https://git.<DOMAIN>` | Source Control |
| Homepage | `https://home.<DOMAIN>` | System Dashboard |
| Jellyfin | `https://jellyfin.<DOMAIN>/web/#/wizardstart.html` | Media |

### Configure Authentik

Some apps needs to be configured manually

#### Jellyfin

   - Follow this guide [Authentik docs](https://integrations.goauthentik.io/media/jellyfin/)   
   - In plugin settings add Roles: 
      ```
      user
      admin 
      authentik Admins
      ```
   - Admin Roles: 
      ```
      admin
      authentik Admins
      ```
   - Enable Role-Based Folder Access
   - Enable All Folders
   - Role Claim: groups

#### Audiobookshelf
  
   - Guide [Authentik docs](https://www.audiobookshelf.org/guides/oidc_authentication)    
  
#### Home Assistant
   - Install [HACS](https://www.hacs.xyz/docs/use/download/download/#to-download-hacs)   
   - Install [hass-openid](https://github.com/cavefire/hass-openid)   

#### Paperless-ngx
   - Create initial admin user
   - In "My Profile"
   - Under "Connected social accounts" link account to Authentik

### Configure Apps

#### Tailscale Operator
   1. Create an OAuth client at https://login.tailscale.com/admin/settings/oauth
      - Scopes: **Devices Core**, **Auth Keys**, **Services** (all write)
      - Tag the client with: `tag:k8s-operator`
   2. Add to your vault file (`group_vars/all/vault.yml`):
      ```yaml
      vault_tailscale_oauth_client_id: "<client-id>"
      vault_tailscale_oauth_client_secret: "<tskey-client-...>"
      ```
   3. Update your Tailscale ACL policy (https://login.tailscale.com/admin/acls) — JSON editor:
      ```json
      "tagOwners": {
          "tag:k8s-operator": [],
          "tag:k8s": ["tag:k8s-operator"]
      },
      "autoApprovers": {
          "routes": {
              "10.42.0.0/16": ["tag:k8s"],
              "10.43.0.0/16": ["tag:k8s"]
          }
      }
      ```
   4. Set `tailscale.enabled: true` and `tailscale.subnet` (your LAN CIDR, e.g. `192.168.0.0/24`) in `main.yml`
   5. Deploy: `ansible-playbook playbooks/main.yaml --tags infra:tailscale`
   6. The LAN subnet route requires manual approval: Tailscale Admin > Machines > k8s-subnet-router > Edit route settings > Approve

#### qBittorrent
   - If you enabled qBittorrent with VPN, check your torrent client IP [checkmytorrentipaddress](https://torguard.net/checkmytorrentipaddress.php)  

#### Jellyfin
   - Initial setup https://jellyfin.<DOMAIN>/web/#/wizardstart.html

#### Jellyseerr
   - Initial setup https://jellyseerr.<DOMAIN>/setup

#### Radarr, Sonarr add download client
   - Configure login
   - Add Download Client - qBittorrent https://radarr.<DOMAIN>/settings/downloadclients
   - Host: qbittorrent-vpn.arr.svc.cluster.local
   - Port: 8080
   - User: admin
   - PW: Check logs from qBittorrent pod

## RKE2 Cluster Upgrade

> Do not skip minor versions: `v1.32 → v1.33 → v1.34 → v1.35`

1. Set target version in `group_vars/all/main.yml`:
   ```yaml
   rke2_version: "v1.34.6+rke2r1"
   ```
2. Review [release notes](https://github.com/rancher/rke2/releases) for breaking changes
3. Run the upgrade:
   ```bash
   ansible-playbook playbooks/upgrade-rke2.yaml
   ```

The playbook upgrades nodes one at a time (primary first, then secondaries). Each node is cordoned, drained, upgraded, and uncordoned. The cluster API stays available throughout via kube-vip.

Before the first node is touched, the playbook automatically:
- Saves the cluster token to `rke2-cluster-token.txt` (required for etcd restore — store it securely)
- Takes an etcd snapshot named `pre-upgrade-<version>-<date>`

### Rollback

```bash
# Stop RKE2 on all nodes
systemctl stop rke2-server

# Restore snapshot on primary
rke2 server --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/pre-upgrade-<version>-<date>.db

# Start primary, then on each secondary clear db and restart
rm -rf /var/lib/rancher/rke2/server/db/ && systemctl start rke2-server
```

### Troubleshooting

| Problem | Fix |
|---------|-----|
| Node stuck `NotReady` | `systemctl restart rke2-server` and check `journalctl -u rke2-server -n 100` |
| Node left cordoned | `kubectl uncordon <node>` |
| Re-run after fixing issue | Safe to re-run — nodes already on target version are skipped |

## Troubleshooting

Common issues and solutions:

1. **Certificate Issues**
   - Check cert-manager logs
   - Clear Cloudflare DNS TXT records
   - Verify API token permissions

2. **Node Connectivity**
   - Verify Tailscale connectivity
   - Check firewall rules
   - Ensure correct SSH keys

3. **GitOps Sync Issues**
   - Check ArgoCD logs for sync errors
   - Verify repository access
   - Validate YAML syntax in manifests

4. **Garage / PostgreSQL Backup Issues**
   ```bash
   # Check Garage pod status
   kubectl logs -n garage garage-0
   kubectl exec -n garage garage-0 -- /garage status

   # Re-run bootstrap (recreates bucket/key, re-patches CNPG cluster)
   ansible-playbook playbooks/main.yaml --tags apps:garage

   # Check WAL archiving status
   kubectl get cluster <name> -n <ns> -o jsonpath='{.status.conditions}'
   kubectl logs -n cnpg-system -l app.kubernetes.io/name=cloudnative-pg --tail=50
   ```

5. **External Storage (SFTP stale mount)**
   ```bash
   # On the affected node — clear stale FUSE mount
   fusermount -u /mnt/<name>-storage
   umount -f /mnt/<name>-storage
   rm -rf /mnt/<name>-storage
   kubectl rollout restart daemonset/<name>-sftp -n <namespace>
   ```

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Submit a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

- 🌟 Star this repo if you find it helpful!
- 🐛 Report issues in the [Issue Tracker](https://github.com/rtomik/ansible-gitops-k8s/issues)
- 📝 Submit improvements via Pull Requests
