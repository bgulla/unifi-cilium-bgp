# Quick Start Guide - Cilium BGP with UDM Pro

## Your Current Setup

- **Kubernetes Nodes**:
  - tpi1: 10.0.1.117
  - tpi2: 10.0.1.142
  - tpi3: 10.0.1.103
- **Cilium Version**: 1.18.0 ✓
- **Target VLAN**: VLAN 3 (10.0.3.0/24, gateway 10.0.3.1)
- **LoadBalancer IP Range**: 10.0.3.100-10.0.3.111

## Network Connectivity Options

Your nodes are on 10.0.1.x but need to peer with 10.0.3.1. You have two options:

### Option A: Inter-VLAN Routing (Recommended - Easier)

Since your UDM Pro already routes between VLANs, BGP peering should work across VLANs:

1. **Update** `udm-pro-bgp-config.conf` to use your actual node IPs:
   ```
   neighbor 10.0.1.117 peer-group K8S
   neighbor 10.0.1.142 peer-group K8S
   neighbor 10.0.1.103 peer-group K8S
   ```

2. **Update** `cilium-bgp-peering-policy.yaml`:
   ```yaml
   neighbors:
   - peerAddress: '10.0.3.1/32'
     peerASN: 64500
     peerPort: 179
   ```

3. **Ensure** UDM Pro firewall allows:
   - Source: 10.0.1.0/24 (your node network)
   - Destination: 10.0.3.1
   - Port: TCP 179 (BGP)

### Option B: Add Nodes to VLAN 3 (More Complex)

Configure a secondary interface on each node with VLAN 3 tagging. This is more complex but provides better network isolation.

## Step-by-Step Setup (Using Option A)

### 1. Update UDM Pro BGP Config

**IMPORTANT**: The UDM Pro BGP configuration requires you to **explicitly list each Kubernetes node IP** that will peer with it. You cannot use wildcards or CIDR ranges.

First, get your actual node IPs:
```bash
kubectl get nodes -o wide
```

Then edit [udm-pro-bgp-config.conf](./udm-pro-bgp-config.conf) and add each node IP individually:

```conf
 ! Add each Kubernetes node IP that will peer with UDM Pro
 ! You MUST list each node explicitly - wildcards are not supported
 neighbor 10.0.1.169 peer-group K8S  # Replace with your actual node IP
 neighbor 10.0.1.XXX peer-group K8S  # Add additional nodes if you have them
```

### 2. Enable Cilium BGP Control Plane

Check if enabled:
```bash
kubectl -n kube-system get configmap cilium-config -o yaml | grep bgp-control-plane
```

If not enabled:
```bash
# Using Helm (adjust values as needed)
helm upgrade cilium cilium/cilium -n kube-system \
  --reuse-values \
  --set bgpControlPlane.enabled=true
```

Or edit the Cilium ConfigMap directly and restart Cilium pods.

### 3. Configure UDM Pro

1. Log into UniFi Network UI
2. Go to **Settings → Routing → BGP**
3. Upload the modified `udm-pro-bgp-config.conf`
4. Verify under **Settings → Firewall** that BGP traffic is allowed

### 4. Apply Cilium BGP Resources

```bash
cd /Users/brandon/src/day0/cilium/bgp

kubectl apply -f loadbalancer-ip-pool.yaml
kubectl apply -f bgp-advertisement.yaml
kubectl apply -f cilium-bgp-peering-policy.yaml
```

### 5. Verify

**Check Cilium status:**
```bash
kubectl get ciliumbgppeeringpolicies
kubectl get ciliumloadbalancerippools
kubectl -n kube-system logs -l k8s-app=cilium | grep -i bgp | tail -20
```

**Check UDM Pro (SSH):**
```bash
vtysh
show ip bgp summary
show ip bgp neighbors
```

You should see 3 BGP sessions (one per node) in "Established" state.

### 6. Test

Create a test service:
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl get svc nginx

# Should show EXTERNAL-IP from 10.0.3.100-10.0.3.111 range
```

Test connectivity:
```bash
curl http://10.0.3.XXX  # Use the assigned IP
```

## Troubleshooting

### BGP Sessions Not Establishing

**Test connectivity from nodes to UDM Pro:**
```bash
# On each node
ping 10.0.3.1
nc -zv 10.0.3.1 179
```

If ping works but port 179 is blocked, check UDM Pro firewall rules.

**Check Cilium BGP logs:**
```bash
kubectl -n kube-system logs -l k8s-app=cilium | grep -i "bgp\|peer"
```

### LoadBalancer IP Not Assigned

**Check if IP pool is available:**
```bash
kubectl get ciliumloadbalancerippools k8s-loadbalancer-pool -o yaml
```

**Check service events:**
```bash
kubectl describe svc <service-name>
```

### Routes Not Propagating

**On UDM Pro, check received routes:**
```bash
vtysh -c "show ip bgp neighbors 10.0.1.117 routes"
```

**Check BGP advertisement config:**
```bash
kubectl get ciliumbgpadvertisements -o yaml
```

## Network Diagram

```
┌─────────────────────────────────────────┐
│ UDM Pro (10.0.3.1)                      │
│ ASN: 64500                              │
│ Role: BGP Router                        │
└────────────┬────────────────────────────┘
             │ BGP Peering (TCP 179)
             │ Advertises: Routes to 10.0.3.100/28
             │
    ┌────────┴────────┬───────────────────┐
    │                 │                   │
┌───▼────┐      ┌─────▼───┐      ┌───────▼──┐
│ tpi1   │      │ tpi2    │      │ tpi3     │
│ 10.0.1 │      │ 10.0.1  │      │ 10.0.1   │
│ .117   │      │ .142    │      │ .103     │
└────────┘      └─────────┘      └──────────┘
  Cilium BGP Speaker (ASN 64501)

  Advertises LoadBalancer IPs:
  10.0.3.100 - 10.0.3.111
```

## Next Steps After Setup

1. **Remove MetalLB** (if installed) to avoid conflicts
2. **Migrate existing LoadBalancer services** to use Cilium IP pools
3. **Configure service-specific pools** for internal vs external services
4. **Add BGP communities** for traffic engineering (optional)
5. **Enable Pod CIDR advertisement** for direct pod routing (optional)

## Files in This Directory

- `cilium-bgp-peering-policy.yaml` - Defines BGP peering with UDM Pro
- `loadbalancer-ip-pool.yaml` - Defines IP range for LoadBalancers
- `bgp-advertisement.yaml` - Controls what routes are advertised
- `udm-pro-bgp-config.conf` - FRR config to upload to UDM Pro
- `README.md` - Detailed documentation
- `QUICKSTART.md` - This file

## Support Resources

- Cilium BGP Docs: https://docs.cilium.io/en/stable/network/bgp-control-plane/
- Cilium Slack: #cilium-users
- Your setup: 3 nodes, Cilium 1.18.0, UDM Pro on 10.0.3.1
