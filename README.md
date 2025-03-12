# OpenShift Network Pre-Check Guide

## 📌 Overview
Before deploying an OpenShift (OCP) cluster, it's crucial to validate network configurations. This guide provides a structured checklist to ensure proper connectivity, DNS resolution, firewall settings, and other network requirements.

---

## 🛠 Pre-Check Steps

### 1️⃣ **Validate Network Requirements**
✅ **Subnet & IP Ranges**:
- **Cluster Network (Pods)** → Default: `10.128.0.0/14`
- **Service Network (Internal Services)** → Default: `172.30.0.0/16`
- **Node Network (Infrastructure)** → Should match your on-prem or cloud subnet.

✅ **IP Addressing**:
- **Static IPs** for all control plane and worker nodes.
- **Virtual IPs (VIPs)**:
  - API (`api.<clustername>.<domain>`) → Example: `192.168.1.100`
  - Ingress (`*.apps.<clustername>.<domain>`) → Example: `192.168.1.101`

✅ **Load Balancer Configuration**:
- External traffic must be routed through a **Load Balancer**.
- API Load Balancer → Ports `6443`, `22623`
- Ingress Load Balancer → Ports `80`, `443`

✅ **DNS Configuration**:
- Forward & reverse DNS records must be correctly set.
- Validate DNS resolution:
  ```sh
  nslookup api.ocp-cluster.example.com
  nslookup apps.ocp-cluster.example.com
  nslookup <node-hostname>
  ```
- Ensure **CoreDNS** (or external DNS) is properly configured.

✅ **NTP (Time Sync)**:
- Ensure all nodes synchronize with an NTP server:
  ```sh
  chronyc tracking
  ```
- Time desynchronization can break cluster operations.

---

### 2️⃣ **Check Firewall & Port Access**
Ensure all required OpenShift ports are open between nodes and external systems.

| **Component**       | **Protocol** | **Port(s)**  | **Description**                 |
|---------------------|-------------|-------------|---------------------------------|
| **API Server**      | TCP         | `6443`      | OpenShift API Server           |
| **Bootstrap (Ignition)** | TCP   | `22623`     | Ignition Configuration         |
| **Ingress Controller** | TCP     | `80, 443`   | Ingress traffic                |
| **Node Communication** | TCP     | `10250`     | Kubelet API                     |
| **Etcd (Control Plane Only)** | TCP | `2379-2380` | etcd database                 |
| **NodePort Services** | TCP      | `30000-32767` | Exposed services              |

🔹 **Check firewall settings**:
```sh
firewall-cmd --list-all
```

🔹 **Check port connectivity between nodes**:
```sh
nc -zv <node-ip> 6443
```

---

### 3️⃣ **Validate Network Connectivity**

✅ **Verify Node-to-Node Connectivity**:
```sh
ping <node-ip>
ssh <node-ip>
```

✅ **Check Route Table & Default Gateway**:
```sh
ip route show
```

✅ **Check MTU (Maximum Transmission Unit)**:
For VLANs and SDNs, **MTU must match across nodes** (default: `1500`).
```sh
ip link show eth0
```

✅ **Test External Connectivity**:
Verify outbound access for package downloads, registry, and cluster telemetry.
```sh
curl -I https://mirror.openshift.com
```

---

### 4️⃣ **Validate Load Balancer Setup**

✅ **Check Load Balancer Configuration**:
- Ensure proper backend setup for HAProxy, Nginx, F5, or cloud-based LBs (AWS ELB, Azure LB).

✅ **Test API and Ingress Reachability**:
```sh
curl -k https://api.ocp-cluster.example.com:6443
curl -k https://apps.ocp-cluster.example.com
```
🔹 If failing, check **DNS, firewall, and Load Balancer settings**.

---

### 5️⃣ **Validate OpenShift Installation Files**

✅ **Check `install-config.yaml` (IP Ranges & Networks)**:
```yaml
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  networkType: OVNKubernetes
```
- `clusterNetwork` → Defines the Pod network range.
- `serviceNetwork` → Defines the internal services network.
- `networkType` → Ensure it's `OVNKubernetes` (recommended).

✅ **Validate Configuration File Before Deployment**:
```sh
openshift-install create cluster --dir=<install-dir> --log-level=debug
```

---

### ✅ **Summary Pre-Check Table**

| **Check** | **Command/Validation** | **Status** |
|-----------|-----------------------|------------|
| **Subnet & IPs defined** | Review `install-config.yaml` | ✅ / ❌ |
| **DNS Resolution Works** | `nslookup api.<cluster>` | ✅ / ❌ |
| **NTP Synchronized** | `chronyc tracking` | ✅ / ❌ |
| **Firewall Ports Open** | `firewall-cmd --list-ports` | ✅ / ❌ |
| **Node-to-Node Connectivity** | `ping <node-ip>` | ✅ / ❌ |
| **MTU Consistency** | `ip link show` | ✅ / ❌ |
| **API & Ingress Reachable** | `curl -k https://api.<cluster>` | ✅ / ❌ |

---

Would you like any modifications for UPenn’s specific environment? 🚀
# OpenShift Network Pre-Check Script

## Overview
This script automates the validation of network settings before deploying an OpenShift cluster. It checks connectivity, DNS resolution, firewall rules, time synchronization, MTU settings, and endpoint reachability.

## Prerequisites
- Ensure SSH access to all cluster nodes.
- Run the script as a user with sudo privileges.
- Modify the variables in the script to match your cluster environment.

## 📜 Script: `ocp_network_precheck.sh`
```bash
#!/bin/bash

# Define variables
CLUSTER_NAME="ocp-cluster"
DOMAIN="example.com"
API_IP="192.168.1.100"
INGRESS_IP="192.168.1.101"
NODES=("192.168.1.10" "192.168.1.11" "192.168.1.12" "192.168.1.20" "192.168.1.21")
DNS_ENTRIES=("api.${CLUSTER_NAME}.${DOMAIN}" "apps.${CLUSTER_NAME}.${DOMAIN}")
REQUIRED_PORTS=("6443" "22623" "2379" "2380" "10250" "80" "443")

# Check connectivity to each node
echo "🔍 Checking node connectivity..."
for NODE in "${NODES[@]}"; do
    if ping -c 2 $NODE &> /dev/null; then
        echo "✅ Node $NODE is reachable."
    else
        echo "❌ Node $NODE is NOT reachable!"
    fi
done

# Check DNS resolution
echo "🔍 Checking DNS resolution..."
for ENTRY in "${DNS_ENTRIES[@]}"; do
    if nslookup $ENTRY &> /dev/null; then
        echo "✅ DNS resolves $ENTRY correctly."
    else
        echo "❌ DNS resolution failed for $ENTRY!"
    fi
done

# Check API and Ingress VIP reachability
echo "🔍 Checking API and Ingress VIP reachability..."
for VIP in $API_IP $INGRESS_IP; do
    if ping -c 2 $VIP &> /dev/null; then
        echo "✅ VIP $VIP is reachable."
    else
        echo "❌ VIP $VIP is NOT reachable!"
    fi
done

# Check network configuration
echo "🔍 Checking IP routes..."
ip route show | grep -E "default|${NODES[0]}" && echo "✅ Routing looks good." || echo "❌ Routing issue detected!"

# Check firewall rules
echo "🔍 Checking required firewall ports..."
for PORT in "${REQUIRED_PORTS[@]}"; do
    if sudo firewall-cmd --list-ports | grep -q $PORT; then
        echo "✅ Port $PORT is open."
    else
        echo "❌ Port $PORT is NOT open!"
    fi
done

# Check time synchronization
echo "🔍 Checking NTP synchronization..."
if chronyc tracking &> /dev/null; then
    echo "✅ NTP is synchronized."
else
    echo "❌ NTP is NOT synchronized!"
fi

# Check MTU settings
echo "🔍 Checking MTU settings..."
for NODE in "${NODES[@]}"; do
    SSH_CMD="ssh -o StrictHostKeyChecking=no root@$NODE"
    MTU=$($SSH_CMD "ip link show eth0 | grep mtu | awk '{print \$5}'")
    echo "🔹 MTU on $NODE: $MTU"
done

# Check API and Ingress endpoints
echo "🔍 Checking OpenShift API and Ingress endpoints..."
curl -k --connect-timeout 5 https://api.${CLUSTER_NAME}.${DOMAIN}:6443 &> /dev/null && echo "✅ API endpoint is reachable." || echo "❌ API endpoint is NOT reachable!"
curl -k --connect-timeout 5 https://apps.${CLUSTER_NAME}.${DOMAIN} &> /dev/null && echo "✅ Ingress endpoint is reachable." || echo "❌ Ingress endpoint is NOT reachable!"

echo "✅ Pre-checks completed!"
```
```

## 📌 How to Use

### 1️⃣ Modify the Script
- Change the following variables:
  - `CLUSTER_NAME` → Your OpenShift cluster name.
  - `DOMAIN` → Your domain.
  - `API_IP` and `INGRESS_IP` → Reserved VIPs for OpenShift.
  - `NODES` → List of control plane and worker node IPs.
  - `REQUIRED_PORTS` → Ensure these ports are open for communication.

### 2️⃣ Make the Script Executable
```bash
chmod +x ocp_network_precheck.sh
```

### 3️⃣ Run the Script
```bash
./ocp_network_precheck.sh
```
