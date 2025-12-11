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
| Infrastructure | - KubeVIP (LoadBalancer)<br>- Traefik (Ingress Controller)<br>- Longhorn (Distributed Block Storage)<br>- CloudNativePG (Central PostgreSQL)<br>- Dragonfly (Central Redis Cache)<br>- Velero (Backup & Restore) |
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

### Network Architecture

- **KubeVIP**: Provides a virtual IP for cluster API access
- **Traefik**: Ingress controller with automatic TLS via cert-manager
- **Tailscale**: Secure VPN access to cluster services
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
- [Tailscale account](https://login.tailscale.com/admin/settings/keys)- 

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
   ```
   ansible-playbook playbooks/main.yml --tags infra
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

#### Tailscale
   - If you enabled Tailscale, in machine list select rke2-cluster > Edit Route settings > Approve All

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
