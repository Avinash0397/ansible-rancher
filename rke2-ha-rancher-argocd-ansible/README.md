# 3-Node HA RKE2 + Rancher + Argo CD Ansible Setup

This playbook installs:

- 3-node HA RKE2 server/control-plane cluster
- kubectl and crictl symlinks
- Helm
- cert-manager
- Rancher with 3 replicas
- Argo CD with HA-oriented values
- systemd-managed port-forward services:
  - Argo CD: `https://<node1-public-ip>:8080`
  - Rancher: `https://<node1-public-ip>:8443`

## 1. Install Ansible collections on the Ansible master

```bash
ansible-galaxy collection install community.general
```

## 2. Update inventory

Edit `inventory.ini` and replace:

```ini
rke2-1 ansible_host=20.244.31.172 node_ip=10.0.0.4
rke2-2 ansible_host=98.70.38.21  node_ip=10.0.0.5
rke2-3 ansible_host=<NODE_3_PUBLIC_IP> node_ip=10.0.0.6
```

Use your real VM public IPs. `node_ip` should be the private IP of each VM if available.

## 3. Generate production token

```bash
openssl rand -hex 32
```

Put it in `group_vars/all.yml`:

```yaml
rke2_cluster_token: "<your-generated-token>"
```

Production recommendation:

```bash
ansible-vault encrypt group_vars/all.yml
```

## 4. For real production HA, configure a load balancer

Create a TCP load balancer in front of all 3 RKE2 server nodes.

Forward:

```text
TCP 6443 -> all RKE2 server nodes
TCP 9345 -> all RKE2 server nodes
```

Then update:

```yaml
rke2_api_endpoint: "rke2-api.yourcompany.com"
```

Without this, the fallback uses node 1 as the registration/API endpoint.

## 5. Fix SSH host key issue if needed

Preferred:

```bash
ssh-keyscan -H 20.244.31.172 >> ~/.ssh/known_hosts
ssh-keyscan -H 98.70.38.21 >> ~/.ssh/known_hosts
ssh-keyscan -H <NODE_3_PUBLIC_IP> >> ~/.ssh/known_hosts
```

Lab-only fallback: uncomment this in `inventory.ini`:

```ini
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

## 6. Run the playbook

```bash
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini site.yml
```

## 7. Verify cluster

On node 1:

```bash
sudo kubectl get nodes -o wide
sudo kubectl get pods -A
sudo helm list -A
```

## 8. Access URLs

From browser:

```text
Rancher: https://<node1-public-ip>:8443
Argo CD: https://<node1-public-ip>:8080
```

Because port-forward exposes HTTPS services, the browser may show a self-signed certificate warning.

## 9. Get passwords

Rancher bootstrap password is from:

```yaml
rancher_bootstrap_password
```

Argo CD initial admin password:

```bash
sudo kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Username:

```text
admin
```

## 10. Check port-forward services

```bash
sudo systemctl status argocd-port-forward
sudo systemctl status rancher-port-forward
sudo journalctl -u argocd-port-forward -f
sudo journalctl -u rancher-port-forward -f
```

## Production Notes

Port-forward is not recommended as the final production access method. For production, use DNS + Ingress/LoadBalancer + TLS certificates. Port-forward is included because it is useful in lab and POC environments.
