# Kubernetes the Hard Way: 4-Node Cluster Build - Complete Journey Documentation

## Project Overview

Successfully deployed a production-grade Kubernetes cluster from scratch using Kelsey Hightower's "Kubernetes the Hard Way" tutorial, scaling from the original 1-worker-node design to a **4-worker-node cluster** (node-0 through node-3). This bare-metal and VM hybrid deployment on a home lab environment (Proxmox + physical hardware) demonstrates deep understanding of Kubernetes internals, networking, security, and troubleshooting.

**Infrastructure:**
- **Control Plane:** 1 server node (192.168.1.149)
- **Worker Nodes:** 4 nodes across Proxmox VMs and bare metal
  - node-0: 192.168.1.72 (Debian 12, Proxmox VM)
  - node-1: 192.168.1.153 (Debian 13, Proxmox VM)
  - node-2: 192.168.1.167 (Debian 13, bare metal)
  - node-3: 192.168.1.124 (Debian 13, bare metal)
- **Network:** Local 192.168.1.0/24 network
- **Pod Network:** 10.200.0.0/16 (with /24 per node)
- **Service Network:** 10.32.0.0/24

***

## Issues Encountered & Solutions

### 1. **Certificate Generation Failure for Scaled Nodes**

**Issue:**
When generating TLS certificates for node-2 and node-3 using OpenSSL, encountered error:
```
Error: No objects specified in config file
Can't open ""node-2".csr" for reading, No such file or directory
```

**Root Cause:**
Tmux terminal multiplexer was mangling array definitions during copy-paste, adding extra quotes around array values. The `certs` array contained `"node-2"` (with quotes) instead of `node-2`, resulting in filenames like `""node-2".csr"`.

**Solution:**
```bash
# Corrected array definition (no quotes around elements)
certs=(
  admin node-0 node-1 node-2 node-3
  kube-proxy kube-scheduler
  kube-controller-manager
  kube-api-server
  service-accounts
)

# Clean slate approach - removed old certs and regenerated
rm -f admin.* node-*.* kube-proxy.* kube-scheduler.* kube-controller-manager.* kube-api-server.* service-accounts.*

# Regenerated all certificates with corrected array
for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096
  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"
  openssl x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" -CAkey "ca.key" \
    -CAcreateserial -out "${i}.crt"
done
```

**Key Insight:** Tmux can interfere with special characters during paste operations. Best practice: type array definitions manually, paste into a text editor first for verification, or use heredocs/files for complex configurations.

***

### 2. **Binary Permission Denied Errors**

**Issue:**
Multiple "Permission denied" errors when executing Kubernetes binaries:
```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
# -bash: /usr/local/bin/kubectl: Permission denied

systemctl status kube-proxy
# status=203/EXEC (cannot execute binary)
```

**Root Cause:**
Downloaded binaries lacked execute permissions. Files had `-rw-r--r--` instead of `-rwxr-xr-x`.

**Affected Binaries:**
- kubectl
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kube-proxy
- kubelet

**Solution:**
```bash
# On control plane (server)
chmod +x /usr/local/bin/kube-apiserver
chmod +x /usr/local/bin/kube-controller-manager
chmod +x /usr/local/bin/kube-scheduler
chmod +x /usr/local/bin/kubectl

# On each worker node (node-0, node-1, node-2, node-3)
chmod +x /usr/local/bin/kube-proxy
chmod +x /usr/local/bin/kubelet

# Restart services
systemctl daemon-reload
systemctl restart kube-proxy
systemctl restart kubelet
```

**Key Insight:** Always set execute permissions immediately after downloading binaries or installing them to their final location. Use `chmod +x` or `install -m 755` during the installation workflow.

***

### 3. **Missing Container Runtime (runc)**

**Issue:**
Pods stuck in `ContainerCreating` state with error:
```
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create shim task:
OCI runtime create failed: exec: "runc": executable file not found in $PATH
```

**Root Cause:**
The `runc` OCI runtime was missing on all worker nodes. While `containerd` was installed, it depends on `runc` as the low-level runtime to actually create and run containers.

**Container Runtime Stack:**
- **containerd** (high-level): Manages images, container lifecycle, and orchestration
- **runc** (low-level): OCI-compliant runtime that creates and runs containers

**Solution:**
```bash
# On all worker nodes
sudo apt update
sudo apt install -y runc

# Verify installation
runc --version
```

**Key Insight:** Containerd is not sufficient alone - it requires an OCI runtime like runc. The tutorial included `runc.arm64` in the scp commands but installation steps were missed. Always verify the complete container runtime stack is in place.

***

### 4. **Pod CIDR Not Allocated to Nodes**

**Issue:**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
# Returns empty pod CIDRs
node-0
node-1
node-2
node-3
```

No routes existed between nodes to reach pod networks on other nodes.

**Root Cause:**
The kube-controller-manager was missing the `--allocate-node-cidrs=true` flag, preventing automatic pod CIDR allocation from the cluster CIDR range.

**Solution:**
```bash
# On control plane - edit kube-controller-manager service
sudo vi /etc/systemd/system/kube-controller-manager.service

# Added flag:
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --allocate-node-cidrs=true \    # <- Added this line
  --cluster-name=kubernetes \
  ...

# Restart controller-manager
sudo systemctl daemon-reload
sudo systemctl restart kube-controller-manager

# Verify pod CIDRs allocated
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
# Now shows:
# node-0    10.200.0.0/24
# node-1    10.200.1.0/24
# node-2    10.200.2.0/24
# node-3    10.200.3.0/24
```

**Key Insight:** The `--allocate-node-cidrs=true` flag is critical for automatic pod network allocation. Without it, the controller-manager won't assign pod subnets to nodes, breaking pod-to-pod networking.

***

### 5. **Missing Inter-Node Static Routes**

**Issue:**
Pods on one node couldn't communicate with pods on other nodes. NodePort service only worked on node-2 (where the pod was running), not on other nodes.

**Root Cause:**
Each worker node had no routes to reach pod networks on other nodes. Node-0 didn't know how to reach 10.200.2.0/24 (node-2's pod network).

**Verification:**
```bash
# On node-0
ip route | grep 10.200
# Returned nothing - no routes to other pod networks
```

**Solution - Static Routes:**
```bash
# Node-0 (10.200.0.0/24) - add routes to other nodes
sudo ip route add 10.200.1.0/24 via 192.168.1.153  # to node-1
sudo ip route add 10.200.2.0/24 via 192.168.1.167  # to node-2
sudo ip route add 10.200.3.0/24 via 192.168.1.124  # to node-3

# Node-1 (10.200.1.0/24) - add routes to other nodes
sudo ip route add 10.200.0.0/24 via 192.168.1.72   # to node-0
sudo ip route add 10.200.2.0/24 via 192.168.1.167  # to node-2
sudo ip route add 10.200.3.0/24 via 192.168.1.124  # to node-3

# Node-2 (10.200.2.0/24) - add routes to other nodes
sudo ip route add 10.200.0.0/24 via 192.168.1.72   # to node-0
sudo ip route add 10.200.1.0/24 via 192.168.1.153  # to node-1
sudo ip route add 10.200.3.0/24 via 192.168.1.124  # to node-3

# Node-3 (10.200.3.0/24) - add routes to other nodes
sudo ip route add 10.200.0.0/24 via 192.168.1.72   # to node-0
sudo ip route add 10.200.1.0/24 via 192.168.1.153  # to node-1
sudo ip route add 10.200.2.0/24 via 192.168.1.167  # to node-2
```

**CNI Configuration (per node):**
Each node had the correct CNI bridge configuration in `/etc/cni/net.d/10-bridge.conf` with its allocated subnet.

**Key Insight:** In a "hard way" setup without a CNI overlay network (like Flannel/Calico), you must manually configure static routes between nodes. Each node must know how to reach every other node's pod CIDR.

***

### 6. **IP Forwarding Disabled (Critical)**

**Issue:**
NodePort services timed out when accessed from external clients (Mac, jumpbox). Curling from localhost on the node worked, but external traffic failed silently.

**Symptoms:**
```bash
# From Mac browser: http://192.168.1.72:32506
# Result: Connection timeout

# From node-0 locally:
curl -I http://127.0.0.1:32506
# Result: HTTP 200 OK (worked!)

# Diagnostic findings:
iptables -t nat -L KUBE-NODEPORTS -n -v
# Showed packet counters incrementing (packets arriving)

conntrack -L | grep 32506
# No conntrack entries for external traffic

iptables -L FORWARD -n -v
# Chain FORWARD (policy ACCEPT 0 packets, 0 bytes) <- All zeros!
```

**Root Cause:**
IP forwarding was disabled (`net.ipv4.ip_forward=0`). Worker nodes couldn't forward packets between network interfaces (eth0 ↔ CNI bridge), preventing external traffic from reaching pods.

**Solution:**
```bash
# Check current status
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 0

# Enable IP forwarding on ALL worker nodes (and optionally control plane)
sudo sysctl -w net.ipv4.ip_forward=1

# Make persistent across reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Verify
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 1
```

**Key Insight:** IP forwarding is **critical** for Kubernetes nodes. Nodes act as routers, forwarding traffic between the host network and pod networks. Linux disables this by default for security. Without it, the kernel drops packets that need to be forwarded, breaking external→pod communication even though iptables/NAT rules are correct.

***

## Troubleshooting Tools & Commands Used

### Certificate & TLS Debugging
```bash
# List and verify certificates
ls -la *.crt *.key *.csr

# Check certificate details
openssl x509 -in admin.crt -text -noout

# Verify certificate against CA
openssl verify -CAfile ca.crt admin.crt
```

### Binary & Permission Debugging
```bash
# Check file permissions
ls -la /usr/local/bin/kube*

# Find binary location
which kubectl
which runc

# Set execute permissions
chmod +x /usr/local/bin/kubectl
chmod 755 /usr/local/bin/kube-proxy

# Check binary version
kubectl version
runc --version
```

### Service & Process Debugging
```bash
# Check service status
systemctl status kube-apiserver
systemctl status kube-proxy
systemctl status kubelet
systemctl status containerd

# View service logs
journalctl -u kube-proxy -n 50 --no-pager
journalctl -u kubelet -f

# Restart services
systemctl daemon-reload
systemctl restart kube-proxy
```

### Kubernetes Debugging
```bash
# Check cluster components
kubectl get componentstatuses
kubectl cluster-info

# Check nodes and pod CIDRs
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# Pod debugging
kubectl get pods -l app=nginx -o wide
kubectl describe pods -l app=nginx
kubectl logs <pod-name>

# Service debugging
kubectl get svc nginx
kubectl get endpoints nginx
kubectl describe svc nginx
```

### Networking Debugging
```bash
# Check routes
ip route
ip route | grep 10.200

# Check network interfaces
ip addr show

# Check IP forwarding
sysctl net.ipv4.ip_forward

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1

# Test connectivity
ping -c 2 10.200.2.1
curl -I http://10.200.2.58:80

# DNS resolution
ping server.kubernetes.local
```

### Iptables & Firewall Debugging
```bash
# View NAT rules
iptables-save -t nat | grep nginx
iptables-save -t nat | grep 32506

# View filter rules
iptables -L -n -v
iptables -L FORWARD -n -v
iptables -L INPUT -n -v

# View specific chains
iptables -t nat -L KUBE-NODEPORTS -n -v
iptables -t nat -L KUBE-SERVICES -n -v
iptables -L KUBE-FORWARD -n -v

# Connection tracking
conntrack -L | grep 32506

# Watch connections in real-time
watch -n 1 'conntrack -L | grep 32506'
```

### CNI & Container Runtime Debugging
```bash
# Check CNI configuration
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-bridge.conf

# Check containerd
systemctl status containerd
ctr version

# Check runc
which runc
runc --version

# List containers
ctr containers list
```

### API Server Testing
```bash
# Test API server connectivity
curl -k https://server.kubernetes.local:6443/healthz
curl -k https://server.kubernetes.local:6443/healthz -v

# From worker nodes
curl -k https://127.0.0.1:6443/healthz  # Should fail on workers
```

***

## Key Technical Learnings

### 1. **Kubernetes Certificate Architecture**
- Each component requires its own TLS certificate (admin, nodes, kube-proxy, scheduler, controller-manager, API server, service accounts)
- Node certificates must use `system:node:<nodeName>` as CN and `system:nodes` as Organization
- OpenSSL config sections must match the `-section` flag in certificate generation
- Clean slate regeneration ensures consistency across all certificates

### 2. **Container Runtime Stack**
```
kubelet
  ↓
containerd (high-level runtime)
  ↓
runc (OCI runtime - low-level)
  ↓
Linux kernel (cgroups, namespaces)
```
Both layers are required - containerd manages lifecycle, runc actually creates containers.

### 3. **Kubernetes Networking Model**

**Pod Networking Requirements:**
- Every pod gets a unique IP from the pod CIDR
- All pods can communicate with all other pods without NAT
- All nodes can communicate with all pods without NAT
- Nodes need routes to reach pod networks on other nodes

**Service Networking (NodePort):**
```
External Client (Mac)
  ↓ (to any node IP:NodePort)
Node (kube-proxy iptables rules)
  ↓ (DNAT to pod IP)
Pod Network Route (static route)
  ↓ (forward to destination node)
Target Pod
```

**iptables flow for NodePort:**
1. KUBE-NODEPORTS (matches port 32506)
2. KUBE-EXT (external traffic handling)
3. KUBE-SVC (service load balancing)
4. KUBE-SEP (service endpoint)
5. DNAT to pod IP

### 4. **kube-proxy's Role**
- Runs on every node as a DaemonSet or systemd service
- Watches API server for Services and Endpoints
- Programs iptables/IPVS rules to implement Service abstraction
- Handles load balancing across multiple pod endpoints
- Requires kubeconfig to authenticate to API server

### 5. **kube-controller-manager's Role**
- Manages node lifecycle and pod CIDR allocation
- `--cluster-cidr` defines the overall pod network range
- `--allocate-node-cidrs=true` enables automatic /24 subnet assignment per node
- Without this, nodes won't get pod CIDRs and CNI can't function

### 6. **IP Forwarding Requirement**
- Kubernetes nodes act as **routers** between networks
- Must forward packets: Host Network ↔ Pod Network ↔ Other Nodes
- Disabled by default in Linux for security
- Must be enabled: `sysctl -w net.ipv4.ip_forward=1`
- Critical for external → NodePort → pod traffic flow

### 7. **CNI (Container Network Interface)**
- Responsible for assigning IPs to pods
- `host-local` IPAM plugin allocates IPs from configured subnet
- `bridge` plugin creates virtual bridge (cni0) on each node
- Each node's CNI config must use its assigned pod CIDR
- Static routes required between nodes in "hard way" setup (vs. overlay networks like Flannel/Calico)

### 8. **Scaling "The Hard Way"**
When adding nodes beyond the tutorial's default:
- Update ca.conf with new node sections
- Update certificate generation arrays
- Configure CNI per node with correct subnet
- Add static routes on ALL nodes (N→N-1 routes per node)
- Ensure kube-controller-manager allocates CIDRs
- Verify binary permissions on new nodes
- Install complete container runtime stack
- Enable IP forwarding

***

## Production Implications & Best Practices

### 1. **Automation is Critical**
Manual setup across 4 nodes revealed why automation tools exist (kubeadm, Ansible, Terraform). In production:
- Use configuration management (Ansible, Puppet, Chef)
- Script binary installations with proper permissions
- Automate network configuration
- Use CNI plugins with automatic route management (Calico, Flannel, Cilium)

### 2. **Binary Permission Security**
```bash
# Best practice during installation
wget https://...../kubectl
chmod +x kubectl
sudo install -m 755 kubectl /usr/local/bin/
```
Always set permissions explicitly rather than assuming defaults.

### 3. **Persistent Configuration**
Runtime changes (routes, sysctl settings) don't survive reboots. Make persistent:
```bash
# For routes - create systemd service or use netplan/ifupdown
# For sysctl
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

# For static routes, consider:
# - /etc/network/interfaces (Debian)
# - Netplan config (Ubuntu)
# - NetworkManager dispatcher scripts
```

### 4. **Monitoring & Observability**
Tools used for debugging should be production monitoring:
- `kubectl get nodes -o wide` → Node monitoring (Prometheus node-exporter)
- `iptables packet counters` → Network flow monitoring
- `conntrack -L` → Connection tracking metrics
- `journalctl -u <service>` → Centralized logging (ELK, Loki)

### 5. **Network Design Considerations**
- Pod CIDR sizing: /24 per node = 254 pods max per node (sufficient for most cases)
- Service CIDR: /24 = 254 services (may need expansion for large clusters)
- Route table growth: N nodes = O(N²) routes with static routing
  - Solution: Use overlay networks (VXLAN, BGP) in larger clusters

### 6. **Certificate Management**
- 10-year certificate expiry set in tutorial (3653 days)
- Production: Use shorter expiries (1 year) with automated rotation
- Consider using cert-manager for automatic certificate lifecycle
- Store CA private key securely (vault, HSM)

### 7. **High Availability Gaps**
Current setup has single points of failure:
- Single control plane node
- Single etcd instance
- No load balancer for API server

Production requires:
- 3+ control plane nodes (etcd quorum)
- Load balancer in front of API servers
- External etcd cluster (or stacked etcd on control plane)

### 8. **Security Hardening**
Areas for production hardening:
- RBAC policies (not tested in smoke tests)
- Pod Security Standards (PSS/PSA)
- Network Policies (restrict pod-to-pod traffic)
- Audit logging enabled on API server
- Secret encryption at rest
- Private network for pod/service CIDRs

***

## Skills Demonstrated

### Technical Skills
- **Kubernetes Architecture:** Deep understanding of control plane and data plane components
- **PKI & TLS:** Certificate generation, signing, and chain of trust
- **Linux Networking:** Routing, iptables/NAT, IP forwarding, conntrack
- **Container Runtimes:** containerd, runc, OCI standards
- **CNI:** Bridge networking, IPAM, pod network configuration
- **systemd:** Service management, unit files, troubleshooting
- **Troubleshooting Methodology:** Systematic debugging from symptoms to root cause

### Problem-Solving Approach
1. **Symptom identification** (error messages, timeouts)
2. **Hypothesis formation** (permission issue, network issue, config missing)
3. **Targeted testing** (specific commands to validate hypothesis)
4. **Root cause identification** (IP forwarding disabled, not just "network broken")
5. **Solution implementation** (fix all nodes, make persistent)
6. **Verification** (test all endpoints, confirm full functionality)

### Infrastructure Skills
- Hybrid bare-metal and VM deployment
- Home lab infrastructure management (Proxmox)
- Network architecture and subnetting
- Multi-node distributed system deployment
- Version control and documentation

***

## Smoke Test Results

All smoke tests passed successfully:

✅ **Data Encryption** - Secrets encrypted at rest in etcd
✅ **Deployments** - nginx deployment created and scaled
✅ **Pod Networking** - Direct pod-to-pod communication across nodes
✅ **Port Forwarding** - kubectl port-forward functionality
✅ **Logs** - Container log retrieval
✅ **Exec** - Shell access to running containers
✅ **Services** - NodePort service accessible from all nodes
✅ **DNS** - CoreDNS resolution working

**Final validation:**
- nginx accessible from external clients (Mac, jumpbox) via NodePort on all 4 worker nodes
- Pod running on node-2, accessible via node-0, node-1, node-2, node-3 IPs
- Full cluster networking operational

***

## Repository Structure

```
kubernetes-the-hard-way/
├── ca.conf                          # OpenSSL CA configuration with all node sections
├── ca.crt / ca.key                  # Root CA certificate and key
├── admin.crt / admin.key            # Admin client certificates
├── node-0.crt through node-3.crt    # Worker node certificates
├── kube-proxy.crt                   # kube-proxy certificate
├── kube-scheduler.crt               # Scheduler certificate
├── kube-controller-manager.crt      # Controller manager certificate
├── kube-api-server.crt              # API server certificate
├── service-accounts.crt / .key      # Service account signing keys
├── configs/
│   ├── containerd-config.toml       # containerd configuration
│   ├── kubelet-config.yaml          # kubelet configuration
│   ├── kube-proxy-config.yaml       # kube-proxy configuration
│   └── 10-bridge.conf               # CNI bridge plugin config
├── units/
│   ├── kube-apiserver.service       # systemd unit files
│   ├── kube-controller-manager.service
│   ├── kube-scheduler.service
│   ├── kubelet.service
│   └── kube-proxy.service
└── docs/
    └── TROUBLESHOOTING.md           # This document
```

***

## Conclusion

This project demonstrates comprehensive Kubernetes knowledge from the ground up - not just using Kubernetes, but **building** Kubernetes. Every component was manually configured, every certificate manually generated, and every network route manually established.

The troubleshooting journey showcased real-world debugging skills applicable to production environments:
- Reading service logs and error messages
- Understanding iptables and Linux networking
- Diagnosing permission and binary execution issues
- Tracing packet flows through complex network stacks
- Systematic root cause analysis

**This experience directly translates to:**
- Production Kubernetes cluster operations
- Platform engineering and infrastructure automation
- Site reliability engineering (SRE)
- DevOps troubleshooting and incident response
- Cloud infrastructure architecture (AWS EKS, GKE, AKS internals)

The 4-node scaled deployment exceeded the tutorial scope, requiring independent problem-solving and deep technical understanding. All issues were resolved through methodical debugging, demonstrating the ability to work independently on complex distributed systems.

***

**Built by:** Andrew Stephens
**Date Completed:** May 28, 2026
**GitHub:** https://www.github.com/astephens-cloud/
**LinkedIn:** https://www.linkedin.com/in/stepheam/
