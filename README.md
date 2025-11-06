# Cilium BGP Configuration for UDM Pro

This directory contains the configuration files needed to set up BGP peering between Cilium and your UDM Pro for LoadBalancer services.

## Architecture

- **UDM Pro**: Acts as BGP router (ASN 64500) with router ID 10.0.3.1
- **Cilium**: Acts as BGP speaker (ASN 64501) on each Kubernetes node
- **LoadBalancer IP Pool**: 10.0.3.100/28 (10.0.3.100 - 10.0.3.111)
- **VLAN**: 3 (10.0.3.0/24)

## Prerequisites

1. **UDM Pro Firmware**: UniFi OS 4.1.13+ (for native BGP support)
2. **Cilium Version**: 1.14+ 
3. **Network Connectivity**: Kubernetes nodes must have connectivity to VLAN 3

## Installation Steps

### Step 1: Configure UDM Pro

1. Log into UDM Pro UniFi Network UI
2. Go to **Settings → Routing → BGP**
3. Click **Create New** or **Add BGP Configuration**
4. Upload the `udm-pro-bgp-config.conf` file
5. **Important**: Edit the neighbor IPs in the config to match your actual Kubernetes node IPs. YOU MUST LIST OUT EACH AND EVERY KUBERNETES HOST IP. No Wildcards, no CIDRs. Yes I agree that it is annoying.

### Step 2: Enable Cilium BGP Control Plane

Check if BGP is enabled in your Cilium config:

```bash
kubectl -n kube-system get configmap cilium-config -o yaml | grep bgp
```

If not enabled, you need to enable it:

```bash
# Using Cilium CLI
cilium upgrade --set bgpControlPlane.enabled=true

# OR using Helm
helm upgrade cilium cilium/cilium -n kube-system \
  --reuse-values \
  --set bgpControlPlane.enabled=true
```

### Step 3: Apply Cilium BGP Resources

```bash
# Apply in order:
kubectl apply -f loadbalancer-ip-pool.yaml
kubectl apply -f bgp-advertisement.yaml
kubectl apply -f cilium-bgp-peering-policy.yaml
```

### Step 4: Verify BGP Peering

**Check BGP Session State (Established vs Idle):**

The most important check is whether the BGP session is **established** or **idle**:

```bash
# Check BGP peer session state - look for "established" in the Session column
kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1 | \
  xargs -I {} kubectl exec -n kube-system {} -c cilium-agent -- cilium bgp peers
```

Expected output when working:
```
Local AS   Peer AS   Peer Address   Session       Uptime   Family         Received   Advertised
64501      64500     10.0.1.1:179   established   5m12s    ipv4/unicast   1          2
```

**Session States:**
- **established**: BGP peering is working correctly ✓
- **idle**: BGP cannot connect to peer (check firewall, UDM Pro config, node IPs)
- **active**: Attempting to establish connection
- **connect**: TCP connection in progress

**Additional Kubernetes Checks:**

```bash
# Check Cilium BGP resources
kubectl get ciliumbgppeeringpolicies
kubectl get ciliumloadbalancerippools
kubectl get ciliumbgpadvertisements

# Check Cilium logs for BGP errors
kubectl -n kube-system logs -l k8s-app=cilium --tail=100 | grep -i bgp

# View advertised routes
kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1 | \
  xargs -I {} kubectl exec -n kube-system {} -c cilium-agent -- cilium bgp routes available ipv4 unicast
```

**On UDM Pro (SSH):**

```bash
# Enter vtysh shell
vtysh

# Check BGP session summary - look for "Established" state
show ip bgp summary

# View detailed neighbor info
show ip bgp neighbors

# Check learned BGP routes
show ip route bgp
```

## Testing

Create a test LoadBalancer service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-lb
  labels:
    io.cilium/lb-pool: main  # Use the IP pool
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

Apply and check:

```bash
kubectl apply -f test-service.yaml
kubectl get svc test-lb

# The EXTERNAL-IP should be from the 10.0.3.100/28 range
# Try accessing it from a device on your network:
curl http://10.0.3.XXX
```

## Troubleshooting

### BGP Sessions Not Establishing

**First, check the BGP session state:**

```bash
kubectl get pods -n kube-system -l k8s-app=cilium -o name | head -1 | \
  xargs -I {} kubectl exec -n kube-system {} -c cilium-agent -- cilium bgp peers
```

If the session shows **"idle"** instead of **"established"**, follow these steps:

1. **Verify UDM Pro has your node IP in BGP config:**
   - Each Kubernetes node IP must be explicitly listed in `udm-pro-bgp-config.conf`
   - Get your node IPs: `kubectl get nodes -o wide`
   - Check the UDM Pro BGP config includes all node IPs as neighbors
   - Re-upload the config if you added/changed nodes

2. **Check node connectivity to UDM Pro:**
   ```bash
   # From your local machine (testing BGP port 179)
   nc -zv 10.0.1.1 179
   ```

3. **Verify firewall rules on UDM Pro:**
   - Allow TCP port 179 from Kubernetes node IPs to UDM Pro
   - Check Settings → Firewall for rules blocking BGP

4. **Check Cilium BGP logs for connection errors:**
   ```bash
   kubectl -n kube-system logs -l k8s-app=cilium --tail=100 | grep -i bgp
   ```

### LoadBalancer IPs Not Being Assigned

1. **Check IP pool:**
   ```bash
   kubectl get ciliumloadbalancerippools -o yaml
   ```

2. **Verify service selector matches:**
   - If you have `serviceSelector` in the IP pool, your services need matching labels
   - Remove `serviceSelector` to make it default for all services

3. **Check Cilium status:**
   ```bash
   cilium status
   ```

### Routes Not Appearing on UDM Pro

1. **Check BGP advertisements:**
   ```bash
   # On UDM Pro
   vtysh -c "show ip bgp"
   ```

2. **Verify prefix-list matches:**
   - The LoadBalancer IPs must fall within `10.0.3.100/28`
   - Check UDM Pro BGP config prefix-list

## Configuration Notes

### ASN Numbers
- **64500**: UDM Pro (private ASN, you can change if needed)
- **64501**: Cilium (private ASN, must differ from UDM Pro)
- Both are in the private ASN range (64512-65534)

### IP Allocation
- **10.0.3.100/28**: 14 usable IPs for LoadBalancer services
- Adjust the CIDR in both `loadbalancer-ip-pool.yaml` and `udm-pro-bgp-config.conf` if you need more IPs

### Node IPs

**CRITICAL**: The UDM Pro BGP configuration does **NOT** support wildcards or CIDR ranges. You **must explicitly list each Kubernetes node IP** that will peer with the UDM Pro.

To find your node IPs:
```bash
kubectl get nodes -o wide
```

Then update `udm-pro-bgp-config.conf` with each node IP individually:
```
neighbor 10.0.1.169 peer-group K8S  # Replace with actual node IP
neighbor 10.0.1.XXX peer-group K8S  # Add each additional node
```

**Important**: If you add or remove nodes from your cluster, you must update the UDM Pro BGP config and re-upload it to include/remove the node IPs.

## Additional Configuration

### Enable Pod CIDR Advertisement (Optional)

If you want to advertise Pod CIDRs over BGP (for direct pod-to-pod routing):

Edit `cilium-bgp-peering-policy.yaml`:
```yaml
exportPodCIDR: true  # Change from false to true
```

### Add Multiple IP Pools

Edit `loadbalancer-ip-pool.yaml` to add more CIDR blocks:
```yaml
spec:
  blocks:
  - cidr: 10.0.3.100/28
  - cidr: 10.0.3.120/29
  - cidr: 10.0.3.150/29
```

### Selective Pool Assignment

Create multiple IP pools with different selectors:

```yaml
# pool-internal.yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: internal-pool
spec:
  blocks:
  - cidr: 10.0.3.100/28
  serviceSelector:
    matchLabels:
      lb-type: internal

# pool-external.yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: external-pool
spec:
  blocks:
  - cidr: 10.0.3.120/29
  serviceSelector:
    matchLabels:
      lb-type: external
```

Then label your services:
```yaml
metadata:
  labels:
    lb-type: internal  # or external
```

## References

- [Cilium BGP Control Plane Documentation](https://docs.cilium.io/en/stable/network/bgp-control-plane/)
- [UniFi BGP Configuration](https://help.ui.com/hc/en-us/articles/16271338193559)
# unifi-cilium-bgp
