# Stage 2: Server & Kubernetes Configuration Management (Vanilla kubeadm v1.31)

Production-grade, modular, and idempotent Ansible automation that bootstraps a self-managed Vanilla Kubernetes (`kubeadm` v1.31) cluster on AWS Private Subnet EC2 instances provisioned by **Stage 1 (Terraform)**.

All node communication is performed strictly keyless via **AWS Systems Manager (`community.aws.aws_ssm`)** and an S3 staging bucket. No public IPs, no bastion hosts, and no SSH port 22 are required.

---

## 🏗️ Architecture & Topology

```
                                  Internet
                                     │
                                     ▼
                ┌─────────────────────────────────────────┐
                │   AWS Application Load Balancer (ALB)   │
                │        Target Group Port: 30080         │
                │    Health Check: HTTP GET /healthz      │
                └────────────────────┬────────────────────┘
                                     │
                   Private Subnet Application Layer
      ───────────────────────────────┼───────────────────────────────
                                     │
           ┌─────────────────────────┴─────────────────────────┐
           ▼                                                   ▼
┌──────────────────────┐                           ┌──────────────────────┐
│  role_k8s_worker-1   │                           │  role_k8s_worker-2   │
│       t3.small       │                           │       t3.small       │
│  Private Subnet 1    │                           │  Private Subnet 2    │
│  NodePort :30080     │                           │  NodePort :30080     │
└──────────┬───────────┘                           └───────────┬──────────┘
           │                                                   │
           └─────────────────────────┬─────────────────────────┘
                                     │ (Internal Flannel Overlay: 10.244.0.0/16)
                                     ▼
                          ┌──────────────────────┐
                          │   role_k8s_master    │
                          │    c7i-flex.large    │
                          │   Private Subnet 1   │
                          │ Control Plane v1.31  │
                          └──────────────────────┘
                                     ▲
                                     │ AWS SSM Agent + S3 Artifactory
                                     │ Bucket: e-commerce-artifactory/ansible-ssm
                        ┌────────────┴────────────┐
                        │  GitHub Actions Runner  │
                        │   (OIDC Authenticated)  │
                        └─────────────────────────┘
```

### Cluster Specifications
| Node Name / Role | Instance Type | Quantity | OS | Function |
| :--- | :--- | :--- | :--- | :--- |
| `role_k8s_master` | `c7i-flex.large` | 1 | Ubuntu 24.04 LTS (`amd64`) | Kubernetes Control Plane, Addons, ECR Token Refresher |
| `role_k8s_worker` | `t3.small` | 2 | Ubuntu 24.04 LTS (`amd64`) | Ingress-NGINX DaemonSet, Application Pods (Stage 3) |

---

## 🔒 Security & Connection Contract

1. **Zero Public Exposure:** EC2 instances reside strictly in private application subnets without public IPs and have ingress port 22 disabled.
2. **SSM Dynamic Transport:** The dynamic inventory invokes the `community.aws.aws_ssm` connection plugin with:
   - S3 Staging Bucket: `e-commerce-artifactory`
   - S3 Key Prefix: `ansible-ssm`
   - S3 Addressing Style: `virtual`
3. **Strict Tag Isolation:** The AWS account hosts other instances (such as a separate Ansible control node). Dynamic inventory strictly isolates Stage 2 nodes using:
   - `tag:Project: task-stats-microservices`
   - `tag:Environment: prod`
   - `tag:Role: [k8s-master, k8s-worker]`
   - `instance-state-name: running`

---

## 📁 Repository Directory Structure

```text
Ansible-for-microservices/
├── .github/
│   └── workflows/
│       └── stage2-ansible.yml         # CI/CD pipeline with AWS OIDC authentication
├── group_vars/
│   └── all.yml                        # Global cluster variables & configurations
├── inventory/
│   └── aws_ec2.yml                    # AWS EC2 Dynamic Inventory for SSM
├── roles/
│   ├── common/                        # Applied to ALL nodes (Master + Workers)
│   │   ├── handlers/
│   │   │   └── main.yml               # containerd & kubelet service handlers
│   │   └── tasks/
│   │       └── main.yml               # OS tuning, swapoff, containerd, v1.31 apt hold
│   ├── master/                        # Applied to role_k8s_master
│   │   └── tasks/
│   │       └── main.yml               # kubeadm init, kubeconfig, fresh join token
│   ├── worker/                        # Applied to role_k8s_worker
│   │   └── tasks/
│   │       └── main.yml               # kubeadm join via hostvars token
│   └── addons/                        # Applied to role_k8s_master
│       ├── tasks/
│       │   └── main.yml               # Orchestrates CNI, Ingress, Metrics, ECR Cron
│       └── templates/
│           ├── ingress-nginx-daemonset.yml.j2 # NGINX DaemonSet on worker nodes (Port 30080)
│           ├── nginx-healthz-ingress.yml.j2   # ALB /healthz responder (immediate 200 OK)
│           ├── metrics-server.yml.j2          # Metrics Server (--kubelet-insecure-tls)
│           └── ecr-refresh-cronjob.yml.j2     # 12-hour ECR auth token auto-refresh
├── .gitignore                         # Python, Ansible, and OS artifact exclusions
├── ansible.cfg                        # Production Ansible configuration
├── README.md                          # Repository documentation
├── requirements.txt                   # Pinned Python dependencies
├── requirements.yml                   # Pinned Ansible Galaxy collections
└── site.yml                           # Main multi-play orchestration playbook
```

---

## ⚡ Roles Breakdown & Idempotency Guarantee

### 1. `roles/common` (OS & Container Runtime Preparation)
- **SSM Connection Handshake:** Waits for SSM agent connection (`wait_for_connection` with 120s timeout).
- **Package Manager Safety:** Adds `lock_timeout: 300` on all `apt` tasks to mitigate Ubuntu 24.04 `unattended-upgrades` dpkg lock contention.
- **Swap Disabling:** Executes `swapoff -a` conditionally and comments out swap lines in `/etc/fstab`.
- **Kernel Networking:** Persists and loads `overlay` and `br_netfilter` modules.
- **Sysctl Parameters:** Applies `net.bridge.bridge-nf-call-iptables=1`, `net.bridge.bridge-nf-call-ip6tables=1`, and `net.ipv4.ip_forward=1`.
- **Containerd Runtime:** Installs `containerd`, generates default `config.toml` if absent, ensures `SystemdCgroup = true`, and enables the service.
- **Kubernetes v1.31 APT Repository:** Dearmors the official Google/Kubernetes keyring into `/etc/apt/keyrings/kubernetes-apt-keyring.gpg` and installs `kubelet`, `kubeadm`, and `kubectl` pinned via `dpkg_selections: selection: hold`.

### 2. `roles/master` (Control Plane Bootstrap)
- **Idempotent Init:** Runs `kubeadm init` with `--pod-network-cidr=10.244.0.0/16` and `--apiserver-advertise-address={{ private_ip }}` protected by `creates: /etc/kubernetes/admin.conf`.
- **Kubeconfig Deployment:** Sets up `/root/.kube/config` and `/home/ubuntu/.kube/config` with appropriate ownership (`0600`).
- **Dynamic Join Command Fact:** Always generates a fresh join command using `kubeadm token create --print-join-command` and registers it as `k8s_join_command` so workers can join reliably even on re-runs.

### 3. `roles/worker` (Worker Nodes Join)
- **Idempotent Join:** Reads `{{ hostvars[groups['role_k8s_master'][0]]['k8s_join_command'] }}` and joins protected by `creates: /etc/kubernetes/kubelet.conf`.
- **Kubelet Service:** Ensures `kubelet` is started and enabled.

### 4. `roles/addons` (Cluster Production Addons)
- **CNI (Flannel):** Applies Flannel CNI (`10.244.0.0/16`) and gates until all nodes reach `Ready` condition.
- **Ingress NGINX Controller:** Deployed as a **DaemonSet** targeted strictly at worker nodes with HTTP NodePort pinned to **`30080`**.
- **CRITICAL AWS ALB Health Check Fix:**
  - Stage 1 ALB target group monitors `HTTP GET :30080/healthz` expecting `200 OK`.
  - Deploys an internal lightweight health responder and an Ingress with `nginx.ingress.kubernetes.io/server-snippet` returning `200 OK` on `/healthz` directly on port `30080`.
  - Both worker nodes immediately register as **`Healthy`** in the AWS ALB target group!
- **Metrics Server:** Deployed with `--kubelet-insecure-tls` to allow scraping self-signed kubelet certificates, enabling `kubectl top nodes`, `kubectl top pods`, and Horizontal Pod Autoscalers (HPA).
- **Application Namespace (`prod`) & ECR Auto-Refresh:**
  - Amazon ECR authentication tokens expire every 12 hours.
  - Deploys a dedicated ServiceAccount, Role, RoleBinding, and CronJob (running every 6 hours) that invokes `aws ecr get-login-password` via the EC2 node's IAM instance profile and updates `ecr-registry-secret` in the `prod` namespace.
  - Automatically triggers an immediate initial Job during Stage 2 provisioning so Stage 3 microservice deployments never hit `ImagePullBackOff`.

---

## 🚀 Execution & Verification Guide

### 1. Prerequisites
- Python 3.10+
- AWS CLI v2 with active credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION=ap-southeast-2`) or AWS SSO.
- [AWS Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) installed locally.

### 2. Environment Setup
```bash
# Clone the repository
git clone https://github.com/Abhish7899/Ansible-for-microservices.git
cd Ansible-for-microservices

# Create virtual environment and install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Install Ansible Galaxy collections
ansible-galaxy collection install -r requirements.yml --force
```

### 3. Inventory Dry-Run & Node Resolution
Verify that AWS dynamic inventory resolves the exact EC2 instances over SSM:
```bash
# Graph inventory groups
ansible-inventory -i inventory/aws_ec2.yml --graph

# Detailed host variables verification
ansible-inventory -i inventory/aws_ec2.yml --list
```
Expected output:
```text
@all:
  |--@aws_ec2:
  |  |--i-0xxxxxxxxxxxxxxxxx
  |  |--i-0yyyyyyyyyyyyyyyyy
  |  |--i-0zzzzzzzzzzzzzzzzz
  |--@role_k8s_master:
  |  |--i-0xxxxxxxxxxxxxxxxx
  |--@role_k8s_worker:
  |  |--i-0yyyyyyyyyyyyyyyyy
  |  |--i-0zzzzzzzzzzzzzzzzz
```

### 4. Playbook Syntax Check
```bash
ansible-playbook --syntax-check site.yml
```

### 5. Playbook Execution
```bash
ansible-playbook site.yml
```

---

## 🔄 CI/CD Pipelines

### 1. Declarative Jenkins Pipeline (`Jenkinsfile`)
A production-grade `Jenkinsfile` at the repository root tailored for Jenkins controllers/agents running on AWS EC2 with an attached IAM Instance Profile:
- **Portable Virtual Environment:** Uses `${HOME}/ansible-venv` (safe for the `jenkins` user), auto-creating it and installing dependencies if missing.
- **Stage 1 (Pre-Flight & Tooling Bootstrap):** Validates AWS STS identity, `session-manager-plugin`, and bootstraps Ansible with collections.
- **Stage 2 (Syntax & Inventory Verification):** Validates `site.yml` syntax and verifies dynamic inventory discovers `role_k8s_master` and `role_k8s_worker`.
- **Stage 3 (SSM Connectivity Check):** Runs `ansible all -m ping` over SSM transport before modifying cluster nodes.
- **Stage 4 (Bootstrap & Configure):** Runs `ansible-playbook site.yml`.
- **Stage 5 (Post-Configure Exit Gate):**
  - Confirms nodes are `Ready` and `ecr-registry-secret` is present in `prod` namespace.
  - Polls AWS ALB Target Group (`k8s-ingress-tg-prod`) until both worker nodes report `healthy`.
  - Runs `curl -sfI "http://${ALB_DNS}/healthz"` to confirm `200 OK`.

### 2. GitHub Actions (`.github/workflows/stage2-ansible.yml`)
1. **Trigger:** On `push` to `main` branch or manual `workflow_dispatch`.
2. **AWS OIDC Authentication:** Uses `aws-actions/configure-aws-credentials@v4` with `AWS_ROLE_ARN` secret.
3. **Automated Setup:** Installs `session-manager-plugin`, Python dependencies, and Ansible collections.
4. **Execution:** Performs syntax check, validates inventory graph, and executes `site.yml`.

---

## 🎯 Stage 3 Readiness Handshake

Upon successful execution of Stage 2:
1. `kubectl get nodes -o wide` shows 1 master and 2 worker nodes in `Ready` status.
2. Ingress NGINX is actively listening on worker node port `30080`.
3. The AWS ALB Target Group (`k8s-ingress-tg-prod`) reports both worker nodes as **`Healthy`**.
4. Kubernetes namespace `prod` contains active secret `ecr-registry-secret` continuously renewed by `ecr-token-refresh` CronJob.
5. Microservices (`task-service` and `stats-service`) can now be deployed seamlessly in **Stage 3**!
