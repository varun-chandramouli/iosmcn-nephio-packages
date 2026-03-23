# iosmcn-nephio-packages

This repository contains the deployment guide for provisioning **iosmcn-core** and **iosmcn-ran** using Nephio on BYOH (Bring Your Own Host) clusters.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deploying iosmcn-core](#deploying-iosmcn-core)
  - [Step 1: Increase inotify Limits](#step-1-increase-inotify-limits)
  - [Step 2: Install Dependencies](#step-2-install-dependencies)
  - [Step 3: Install iosmcn Core](#step-3-install-iosmcn-core)
  - [Step 4: Label the Core Cluster](#step-4-label-the-core-cluster)
  - [Step 5: Prepare the Core Cluster with Package Variants](#step-5-prepare-the-core-cluster-with-package-variants)
- [Deploying iosmcn-ran](#deploying-iosmcn-ran)
  - [Step 1: Configure Hugepages](#step-1-configure-hugepages)
  - [Step 2: Set Up the RAN Cluster](#step-2-set-up-the-ran-cluster)
  - [Step 3: Deploy RAN Helm Charts](#step-3-deploy-ran-helm-charts)

---

## Prerequisites

Ensure the cluster has been provisioned using the BYOH script before proceeding with either the core or RAN deployment.

---

## Deploying iosmcn-core

Perform the following steps on the server or VM where you want to deploy the **iosmcn-core**.

### Step 1: Increase inotify Limits

```bash
# Apply limits for the current session
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.file-max=2097152

# Persist limits across reboots
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances=512"  | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=2097152"               | sudo tee -a /etc/sysctl.conf
```

### Step 2: Install Dependencies

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install make sshpass ansible -y
```

### Step 3: Install iosmcn Core

Download the iosmcn core release and run:

```bash
make 5gc-router-install
```

### Step 4: Label the Core Cluster

Replace `<core-node-name>` and `<core-cluster-context>` with your actual node name and cluster context.

```bash
kubectl label nodes <core-node-name> node-role.aetherproject.org=omec-upf \
  --context <core-cluster-context>

kubectl label nodes <core-node-name> node-role.aetherproject.org/omec-upf= \
  --context <core-cluster-context>
```

### Step 5: Prepare the Core Cluster with Package Variants

#### 5a. Register the Git Repository on the Nephio Management Cluster

Run the following on the server/VM where the Nephio management cluster is deployed:

```bash
cat << EOF | kubectl apply -f -
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: iosmcn-nephio-packages
  namespace: default
  labels:
    kpt.dev/repository-access: read-only
    kpt.dev/repository-content: external-blueprints
spec:
  content: Package
  deployment: false
  git:
    branch: main
    directory: /
    repo: https://github.com/varun-chandramouli/iosmcn-nephio-packages.git
  type: git
EOF
```

#### 5b – 5o. Deploy via Nephio Web UI

1. Open the Nephio Web UI at `http://<vm-or-server-ip>:7007`
2. Click **mgmt** in the Repositories section on the homepage.
3. Click **Add Deployment** (top right corner).
4. Under **Action**, select **Create a new deployment by cloning an external blueprint**.
5. Under **Source external blueprint repository**, select `iosmcn-nephio-packages`.
6. Under **External blueprint to clone**, select `nephio-workload-cluster`. Click **Next**.
7. In the **Metadata** section, change the name to `core`. Click **Next**.
8. In the **Namespace** section, click **Next**.
9. In the **Validate** section, click **Next**.
10. In the **Confirm** section, click **Create Deployment**.
11. Click **Edit** (top right corner).
12. Click the last **WorkloadCluster** option and select **Show YAML view** in the popup.
13. Change `masterInterface` to the VM/server's data interface on which the core cluster is deployed. Click **Save**.
14. Click **Save** (top right). On the next screen click **Propose**, then **Approve**.

> This installs the required Nephio packages for deploying workloads and creates the Gitea repository for the core cluster.

#### 5q. Increase MULTUS Pod Memory

1. On the Nephio homepage, click **mgmt-staging** in the Repositories section.
2. Select **core-multus** and click **Create New Revision**, then click **Edit**.
3. Select **DaemonSet** and update the memory value to `500Mi` in both `requests` and `limits`. Click **Save**.
4. Click **Save** (top right). Select **Propose**, then **Approve**.

#### 5r. Update the RootSync with the Correct Gitea IP

1. On the Nephio homepage, click **mgmt-staging** in the Repositories section.
2. Select **core-rootsync** and click **Create New Revision**, then click **Edit**.
3. Select the **Starlark Run** option. Replace `172.x.x.x` with the IP of the VM/server where Gitea is installed (typically the same host as the Nephio management cluster). Click **Save**.
4. Click **Save** (top right). Select **Propose**, then **Approve**.
5. Verify the RootSync was created successfully in the core cluster:

```bash
kubectl get rootsyncs.configsync.gke.io --context core-admin@core -A
```

#### 5s. Deploy the iosmcn Core Flux Helm Chart

Deploy the `iosmcncore-flux` Helm chart using the Nephio Web UI.

---

## Deploying iosmcn-ran

Perform the following steps on the server or VM where you want to deploy the **iosmcn-ran**.

### Step 1: Configure Hugepages

```bash
# Check current hugepage state
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Allocate 20 x 1Gi hugepages
echo 20 | sudo tee /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Verify allocation
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/free_hugepages

# Persist across reboots
echo 'vm.nr_hugepages=20' | sudo tee /etc/sysctl.d/99-hugepages.conf
sudo sysctl -p /etc/sysctl.d/99-hugepages.conf
```

### Step 2: Set Up the RAN Cluster

Repeat [Steps 5b through 5r](#5b--5o-deploy-via-nephio-web-ui) from the core deployment, substituting `ran` for `core` wherever the cluster name or repository name is referenced.

### Step 3: Deploy RAN Helm Charts

Deploy the following Helm charts **in order** using the Nephio Web UI:

1. `iosmcn-ran-cucp-flux`
2. `iosmcn-ran-cuup-flux`
3. `iosmcn-ran-du-flux`
