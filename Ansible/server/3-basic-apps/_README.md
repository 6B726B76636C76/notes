KUBERNETES CLUSTER DEPLOYMENT PLAYBOOK
=======================================

Ansible playbook for deploying a production-ready single-node Kubernetes
cluster on Debian/Ubuntu with a full stack of networking, DNS and TLS
components.

-------------------------------------------------
TABLE OF CONTENTS
-------------------------------------------------

  1. What gets deployed
  2. Requirements
  3. Variables
  4. Usage
  5. Playbook structure
  6. After installation
  7. Health checks
  8. Maintenance
  9. Troubleshooting
 10. Security
 11. Useful commands


=================================================
1. WHAT GETS DEPLOYED
=================================================

BASE COMPONENTS
  - Docker CE + Docker Compose plugin (for building images and running
    containers in CI)
  - containerd as Kubernetes CRI runtime (with SystemdCgroup = true)
  - kubeadm, kubelet, kubectl version 1.31 (packages held via apt-mark hold)

NETWORKING COMPONENTS
  - Cilium 1.16.5 as CNI with full kube-proxy replacement and built-in
    ingress controller in shared mode
  - MetalLB 0.14.9 as L2 load balancer for bare metal (FRR disabled)

TLS AND DNS
  - cert-manager with a ClusterIssuer for Let's Encrypt via Cloudflare DNS-01
  - external-dns for automatic DNS record management in Cloudflare

TOOLS
  - Helm 3 for package management

USERS AND SECURITY
  - System user gitlab-ci (for CI/CD)
  - Users vaclav, gitlab-ci, ansible in the docker group
  - A ~/.kube/config is generated for each user

KERNEL AND SWAP SETTINGS
  - /swapfile is activated with vm.swappiness=10
  - kubelet is configured to work with swap (failSwapOn: false, LimitedSwap)
  - Kernel modules overlay and br_netfilter are loaded
  - ip_forward and bridge netfilter are enabled


=================================================
2. REQUIREMENTS
=================================================

HOSTS
  - OS: Debian 12 (Bookworm) or Ubuntu 22.04/24.04
  - Architecture: x86_64, aarch64 or armv7l
  - Minimum: 2 CPU, 2 GB RAM, 20 GB disk
  - Recommended: 4 CPU, 8 GB RAM, 50 GB SSD
  - Network: static IP, internet access

ANSIBLE
  - Ansible 2.15+
  - Collections:
      ansible-galaxy collection install ansible.posix community.general

EXTERNAL SERVICES
  - Cloudflare account with an API Token that has permissions:
      Zone:DNS:Edit  - for external-dns
      Zone:Zone:Read - for cert-manager
  - A domain managed in Cloudflare
  - An email address for Let's Encrypt

ACCESS
  - SSH access with passwordless sudo (or run with --ask-become-pass)


=================================================
3. VARIABLES
=================================================

REQUIRED VARIABLES
(Pass via --extra-vars or Ansible Vault)

  Variable               Description                    Example
  ---------------------  -----------------------------  ----------------------
  cloudflare_api_token   Cloudflare API Token           v1.0-abc...
  cloudflare_domain      Domain for external-dns        example.com
  letsencrypt_email      Email for Let's Encrypt        admin@example.com

PLAYBOOK VARIABLES (vars)

  Variable              Default                    Description
  --------------------  -------------------------  ----------------------------
  kubernetes_version    1.31                       Kubernetes version (major.minor)
  cilium_version        1.16.5                     Cilium Helm chart version
  pod_cidr              10.244.0.0/16              CIDR for pods
  k8s_users             [vaclav, gitlab-ci,        Users that receive kubeconfig
                         ansible]
  kubeconfig            /etc/kubernetes/admin.conf Path to admin kubeconfig

OPTIONAL VARIABLES

  Variable         Default   Description
  ---------------  --------  ---------------------------------------------
  reset_cluster    false     Full cluster reset (kubeadm reset -f)

  WARNING: reset_cluster=true will destroy the entire cluster and its data.
  Use only for reinstallation.


=================================================
4. USAGE
=================================================

STEP 1 - PREPARE INVENTORY

Create inventory.
Create the encrypted variables by ansible-vault:

  *cloudflare_api_token: "v1.0-xxxxxxxxxxxxxxxxxxxx"*
  *cloudflare_domain: "example.com"*
  *letsencrypt_email: "admin@example.com"*

  *ansible-vault create /home/vaclav/secrets/k8s.yml --vault-password-file /home/vaclav/.vault_pass*

STEP 2 - RUN THE PLAYBOOK

First-time installation:
  *ansible-playbook 3-basic-apps/playbook.yml -e @/home/vaclav/secrets/k8s.yml*

Full cluster reinstall:
  *ansible-playbook 3-basic-apps/playbook.yml -e @/home/vaclav/secrets/k8s.yml -e reset_cluster=true*

Run only specific tags (if you add tags):
  *ansible-playbook 3-basic-apps/playbook.yml -e @/home/vaclav/secrets/k8s.yml --tags cilium*

Dry-run to check:
  *ansible-playbook 3-basic-apps/playbook.yml -e @/home/vaclav/secrets/k8s.yml --check --diff*


=================================================
5. PLAYBOOK STRUCTURE
=================================================

The playbook is logically split into blocks:

  1. Prerequisites
     - swapfile in fstab + activation
     - vm.swappiness=10
     - kernel modules: overlay, br_netfilter
     - sysctl: bridge-nf-call-iptables, ip_forward

  2. Docker + containerd
     - repositories and GPG keys
     - install docker-ce, containerd.io
     - create gitlab-ci user
     - add users to docker group
     - containerd config: SystemdCgroup=true

  3. Kubernetes packages
     - pkgs.k8s.io repository
     - install kubelet/kubeadm/kubectl
     - apt-mark hold
     - /etc/default/kubelet: --fail-swap-on=false

  4. kubeadm init
     - (optional) cluster reset
     - kubeadm init with --skip-phases=addon/kube-proxy
     - copy admin.conf to users
     - remove control-plane taint (single-node)

  5. kubelet swap config
     - failSwapOn: false
     - memorySwap.swapBehavior: LimitedSwap

  6. Helm
     - install via the official script

  7. Cilium
     - helm repo add
     - helm upgrade --install with kubeProxyReplacement=true
     - ingressController.enabled=true
     - wait for DaemonSet rollout

  8. MetalLB
     - install (L2 mode, without FRR)
     - IPAddressPool (host IP)
     - L2Advertisement

  9. cert-manager
     - install with CRDs
     - Secret with Cloudflare API Token
     - ClusterIssuer letsencrypt-cloudflare (DNS-01)

 10. external-dns
     - external-dns namespace
     - Secret with Cloudflare API Token
     - Helm release with provider=cloudflare


=================================================
6. AFTER INSTALLATION
=================================================

1. CHECK NODE STATUS

   kubectl get nodes -o wide
   # Expected:
   # NAME       STATUS   ROLES           AGE   VERSION
   # k8s-node1  Ready    control-plane   5m    v1.31.x

2. CHECK PODS

   kubectl get pods -A

   All pods in kube-system, metallb-system, cert-manager, external-dns must
   be Running.

3. CHECK CERT-MANAGER

   kubectl get clusterissuer
   # NAME                      READY   AGE
   # letsencrypt-cloudflare    True    2m

4. CHECK METALLB

   kubectl get ipaddresspool -n metallb-system
   kubectl get l2advertisement -n metallb-system

5. ACCESS THE CLUSTER

   As any user from k8s_users:

     kubectl get nodes
     kubectl config view

   For user vaclav:

     sudo -u vaclav -i
     kubectl get nodes

6. FIRST TEST DEPLOYMENT WITH TLS

   Create test-app.yaml:

     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
     spec:
       replicas: 1
       selector:
         matchLabels: { app: nginx }
       template:
         metadata:
           labels: { app: nginx }
         spec:
           containers:
           - name: nginx
             image: nginx:alpine
             ports: [{ containerPort: 80 }]
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: nginx
       annotations:
         metallb.io/address-pool: default-pool
     spec:
       type: LoadBalancer
       selector: { app: nginx }
       ports: [{ port: 80, targetPort: 80 }]
     ---
     apiVersion: networking.k8s.io/v1
     kind: Ingress
     metadata:
       name: nginx
       annotations:
         cert-manager.io/cluster-issuer: letsencrypt-cloudflare
     spec:
       ingressClassName: cilium
       tls:
       - hosts: [nginx.example.com]
         secretName: nginx-tls
       rules:
       - host: nginx.example.com
         http:
           paths:
           - path: /
             pathType: Prefix
             backend:
               service:
                 name: nginx
                 port: { number: 80 }

   After a few minutes:
     - kubectl get certificate  -> READY=True
     - DNS record nginx.example.com appears automatically
     - curl https://nginx.example.com returns the nginx response


=================================================
7. HEALTH CHECKS
=================================================

QUICK FULL-STACK CHECK

   # Node is ready
   kubectl get nodes

   # All pods are Running
   kubectl get pods -A | grep -v Running | grep -v Completed

   # Cilium is working
   kubectl -n kube-system exec ds/cilium -- cilium status --brief

   # MetalLB is working
   kubectl -n metallb-system get pods

   # cert-manager is working
   kubectl -n cert-manager get pods
   kubectl get clusterissuer

   # external-dns is working
   kubectl -n external-dns logs deploy/external-dns --tail=20

CHECK SWAP

   swapon --show
   # NAME       TYPE  SIZE  USED  PRIO
   # /swapfile  file  2G    0B    -2

   kubectl describe node k8s-node1 | grep -i swap

CHECK CONTAINERD

   sudo systemctl status containerd
   sudo grep SystemdCgroup /etc/containerd/config.toml
   # Must be: SystemdCgroup = true


=================================================
8. MAINTENANCE
=================================================

UPGRADE KUBERNETES

   # On the control-plane node:
   sudo apt-mark unhold kubeadm kubelet kubectl
   sudo apt update
   sudo apt install -y kubeadm=1.31.x-1.1 kubelet=1.31.x-1.1 kubectl=1.31.x-1.1
   sudo apt-mark hold kubeadm kubelet kubectl

   sudo kubeadm upgrade plan
   sudo kubeadm upgrade apply v1.31.x

   sudo systemctl daemon-reload
   sudo systemctl restart kubelet

UPGRADE CILIUM

   helm upgrade cilium cilium/cilium \
     --version 1.16.6 \
     --namespace kube-system \
     --reuse-values

UPGRADE CERT-MANAGER

   helm upgrade cert-manager jetstack/cert-manager \
     --namespace cert-manager \
     --reuse-values

ROTATE CLOUDFLARE API TOKEN

   # Update cloudflare_api_token in secrets.yml
   ansible-playbook -i inventory.ini k8s.yml \
     --extra-vars "@secrets.yml" \
     --ask-vault-pass \
     --start-at-task "Create Cloudflare secret for cert-manager"

BACKUP

   Critical data:

   # etcd snapshot
   sudo ETCDCTL_API=3 etcdctl \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     snapshot save /backup/etcd-$(date +%Y%m%d).db

   # kubeconfig and certificates
   sudo tar czf /backup/k8s-pki-$(date +%Y%m%d).tar.gz \
     /etc/kubernetes/pki /etc/kubernetes/*.conf


=================================================
9. TROUBLESHOOTING
=================================================

NODE IN NotReady STATE

   # Check kubelet
   sudo journalctl -u kubelet -n 100 --no-pager

   # Check containerd
   sudo journalctl -u containerd -n 100 --no-pager

   # Check Cilium
   kubectl -n kube-system logs ds/cilium --tail=50

CILIUM PODS IN CrashLoopBackOff

   Most often the issue is with k8sServiceHost. Check:

   kubectl -n kube-system get cm cilium-config -o yaml | grep k8sService

   The control-plane IP and port 6443 must be set.

CERT-MANAGER DOES NOT ISSUE A CERTIFICATE

   # Check ClusterIssuer
   kubectl describe clusterissuer letsencrypt-cloudflare

   # Check Challenge
   kubectl get challenges -A
   kubectl describe challenge -n <ns> <name>

   # Verify API Token permissions in Cloudflare:
   # Zone:DNS:Edit and Zone:Zone:Read for the required zone

EXTERNAL-DNS DOES NOT CREATE RECORDS

   kubectl -n external-dns logs deploy/external-dns --tail=100
   # Look for Cloudflare authorization errors

METALLB DOES NOT ASSIGN AN IP

   kubectl -n metallb-system logs deploy/metallb-controller
   kubectl -n metallb-system logs ds/metallb-speaker
   kubectl get svc -A | grep LoadBalancer

   If a service is stuck at <pending>, check the IPAddressPool:

   kubectl get ipaddresspool -n metallb-system -o yaml

KUBEADM INIT FAILS

   # Check the log
   sudo journalctl -u kubelet -n 200 --no-pager
   sudo tail -100 /var/log/syslog | grep kubeadm

   # Full reset and retry
   ansible-playbook -i inventory.ini k8s.yml \
     --extra-vars "@secrets.yml" \
     --extra-vars "reset_cluster=true" \
     --ask-vault-pass

SWAP ISSUES

   If kubelet fails to start due to swap:

   # Check /etc/default/kubelet
   cat /etc/default/kubelet
   # Must be: KUBELET_EXTRA_ARGS="--fail-swap-on=false"

   # Check /var/lib/kubelet/config.yaml
   sudo grep -E "failSwapOn|swapBehavior" /var/lib/kubelet/config.yaml


=================================================
10. SECURITY
=================================================

WHAT IS ALREADY DONE
  - Kubernetes packages pinned via apt-mark hold
  - containerd uses SystemdCgroup = true (Kubernetes recommendation)
  - Cilium fully replaces kube-proxy
  - cert-manager automatically renews TLS certificates
  - Cloudflare API Token stored in a Kubernetes Secret

RECOMMENDED ADDITIONAL STEPS
  1. Configure RBAC for users vaclav, gitlab-ci, ansible - currently all
     have cluster-admin through admin.conf
  2. Restrict access to the API server via firewall (port 6443)
  3. Enable audit logging for the API server
  4. Configure Network Policies in Cilium to isolate namespaces
  5. Use Sealed Secrets or External Secrets Operator instead of plain Secrets
  6. Schedule etcd backups
  7. Monitoring: install Prometheus + Grafana + Alertmanager


=================================================
11. USEFUL COMMANDS
=================================================

   # Get all resources
   kubectl get all -A

   # Check the API server
   kubectl cluster-info

   # Show component versions
   kubectl version --short

   # Cilium diagnostics
   kubectl -n kube-system exec ds/cilium -- cilium status
   kubectl -n kube-system exec ds/cilium -- cilium service list

   # View service endpoints
   kubectl get endpoints -A

   # Cluster events
   kubectl get events -A --sort-by='.lastTimestamp'

   # Node metrics
   kubectl top nodes
   kubectl top pods -A


=================================================
Internal project. Use at your own risk - the playbook modifies system
kernel parameters, swap, and networking settings.
=================================================
