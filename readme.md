# GitOps Ansible RKE2 

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
| Infrastructure | - KubeVIP (LoadBalancer)<br>- Traefik (Ingress Controller)<br>- Longhorn (Distributed Block Storage) |
| GitOps & Management | - ArgoCD (GitOps CD)<br>- Rancher (Kubernetes Management)<br>- Gitea (Source Control)<br>- Renovate (Dependency Automation)|
| Security | - Authentik (SSO/IAM) |
| Monitoring Stack | - Prometheus (Metrics)<br>- Grafana (Visualization)<br>- Loki (Log Aggregation) |
| Other | - Homepage (Dashboard)<br>- Jellyfin (Media server)<br>- Donetick (Task management) |

### Features

- 🚀 Single command cluster deployment
- 🔄 GitOps-based application deployment and lifecycle management
- 📊 Comprehensive monitoring and logging stack
- 🔐 Integrated SSO with Authentik
- 💾 Distributed storage with Longhorn
- 📈 Pre-configured Grafana dashboards for:
  - Application logs and metrics
  - Error monitoring
  - System performance
  - Storage metrics
- 🔄 High availability cluster configuration
- 🔒 TLS encryption for all services with automatic certificate management
- 🔒 Secured Access remotely via Tailscale 
- 🔄 Automated dependency updates with Renovate

## Prerequisites

### Hardware Requirements
- **Control Node:**
  - Linux system/VM or WSL2 on Windows
  - With Git, Ansible and kubectl installed

- **Cluster Nodes:**
  - Ubuntu Server 24.04
  - Minimum 16GB RAM per node
  - Recommended: 3 nodes for HA
  - [Low-cost mini PC options](https://www.lowcostminipcs.com/)

- **My Setup:**
  - HP ProDesk 600 G3 mini - i3 7300t/16GB RAM/500GB SSD cost 65 €
  - GenMachine Mini PC - AMD 5300U/16GB RAM/500GB SSD cost 150 €

### Network Requirements
- SSH access to all nodes (root SSH keys)
- Doall name (Recommended: [Porkbun](https://porkbun.com/) or [DuckDNS](https://www.duckdns.org/))
- [Tailscale account](https://login.tailscale.com/admin/settings/keys)

### Optional
- Cloudflare account and API token (for TLS certificates)
- NFS share for media storage and backups

## Quick Start

1. **Clone and prepare configuration**
   ```bash
   git clone https://github.com/rtomik/k3s_ansible.git && cd k3s_ansible\
   mv inventory.yml_ex inventory.yml && \
   mv group_vars/all.yml_ex group_vars/all.yml
   ```

2. **Configure nodes**
   - Update `inventory.yml` with node IPs
   - Modify `group_vars/all.yml` with your configurations
   - Set required variables in `Required variables` section

3. **Initial Deployment**
   NOTE: If you enabled cloudflare it can take some time to propagate new TLS certificate
   First deploy k3s and cert-manager, 
   ```
   ansible-playbook playbooks/all.yml --tags local,config,k3s,infra:init
   ```
   Kubeconfig will be stored in playbook dir
   Verify certificate (wait 5-10 minutes):
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
   Delete certificate from kubernetes and retry

   After the certificate is created rerun the all playbook.

4. **Complete Deployment**
   ```bash
   ansible-playbook playbooks/all.yml
   ```

5. **To deploy specific components, use tags**
   ```
   ansible-playbook playbooks/all.yml --tags "k3s"
   ```

6. **To destroy the cluster and remove everything**
   ```
   ansible-playbook playbooks/destroy.yml
   ```

## Post Deployment

### Access Services

| Service | URL | Description |
|---------|-----|-------------|
| Rancher | `https://rancher.<doall>` | Kubernetes Management UI |
| ArgoCD | `https://argocd.<doall>` | GitOps Control Panel |
| Grafana | `https://grafana.<doall>` | Metrics & Logs Visualization |
| Authentik | `https://authentik.<doall>` | SSO/IAM Portal |
| Prometheus | `https://prometheus.<doall>` | Metrics Storage |
| Gittea | `https://git.<doall>` | Source Control |
| Homepage | `https://home.<doall>` | System Dashboard |
| Jellyfin | `https://jellyfin.<doall>/web/#/wizardstart.html` | Media |

### Configure Authentik

#### Jellyfin

   - Follow this guide [Authentik docs](https://docs.goauthentik.io/integrations/services/jellyfin/#oidc-configuration)   
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

### Configure Apps

#### Jellyfin
   - Intial setup https://jellyfin.< Doall >/web/#/wizardstart.html

#### Radarr, Sonarr add download client
   - Add Download Client - qBittorrent https://radarr.< Doall >/settings/downloadclients
   - Host: qbittorrent.arr.svc.cluster.local
   - Port: 8080
   - User: admin
   - PW: Check logs from qbittorent pod

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
- 🐛 Report issues in the [Issue Tracker](https://github.com/rtomik/k3s_ansible/issues)
- 📝 Submit improvements via Pull Requests
