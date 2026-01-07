# Module 7: OVN-Kubernetes Networking (Pod Networking)

---

## 📚 DEFINITIONS & KEY CONCEPTS

Before we start the lecture, here are the key terms you MUST know:

### What is Pod Networking?

**Pod Networking** is how containers (pods) communicate with each other across a Kubernetes cluster. Since pods are ephemeral and can be on any node, we need a virtual network that makes them all appear to be on the same flat network.

### Key Terms

| Term | Definition |
|------|------------|
| **SDN (Software Defined Network)** | A virtual network created by software, overlaid on top of the physical network. Allows pods to communicate as if they're on the same LAN, even when they're on different physical servers. |
| **OVN-Kubernetes** | OpenShift's network plugin (since 4.12+, ONLY option in 4.14+). Based on Open Virtual Network and Open vSwitch. Handles all pod-to-pod traffic. |
| **Pod CIDR** | The IP address range for pods. Example: `10.128.0.0/14`. Each node gets a slice of this for its pods. |
| **Service CIDR** | A separate IP range for Kubernetes Services. Example: `172.30.0.0/16`. Services provide stable endpoints for pods. |
| **CNI (Container Network Interface)** | The standard API that Kubernetes uses to configure pod networking. OVN-Kubernetes is a CNI plugin. |
| **Overlay Network** | A network built on top of another network. Pod traffic is encapsulated (wrapped) and sent through the physical network. |
| **Encapsulation** | Wrapping one packet inside another. Pod packets get wrapped in a physical network packet for transport. |
| **ClusterIP** | The internal IP assigned to a Service. Only reachable from inside the cluster. |
| **CoreDNS** | The DNS server inside OpenShift that resolves service names to ClusterIPs. |
| **FQDN** | Fully Qualified Domain Name. For services: `<service>.<namespace>.svc.cluster.local` |

### The One Rule to Remember

> **Every pod can talk to every other pod directly, without NAT, regardless of which node they're on.**

This is the "flat network" model. The SDN makes this possible.

---

## 🎬 LECTURE TRANSCRIPT (~50 minutes)

---

### Opening (4 mins)

[SLIDE - Title: "Phase 3: Networking - The Connectivity Layer"]

"Welcome to Phase 3! We're diving into networking - the topic that confuses more people than anything else in Kubernetes.

But here's the good news: once you understand the basics, it all makes sense. And by the end of today, you'll actually UNDERSTAND how pods talk to each other."

[CHECK]

"Let me ask you something. In a traditional network - VMs on vSphere, EC2 instances, whatever - how do machines talk to each other?"

[PAUSE for answers]

"Right! They have IPs on the same network, or routers connect different networks. Simple.

Now here's the Kubernetes twist..."

[SLIDE - The Problem]

"In Kubernetes:
- Nodes have 'real' IPs like 192.168.1.10, 192.168.1.11
- But pods get IPs like 10.128.0.5, 10.129.2.3
- These pod IPs don't exist on your physical network!

So if I'm a packet traveling from Pod A on Node 1 to Pod B on Node 2... how do I get there?

The physical network doesn't know what 10.128.0.5 is. It's not in any routing table."

[PAUSE - let the problem sink in]

"THIS is what Kubernetes networking solves. And today, we're going to understand exactly how."

---

### Section 1: The Flat Network Model (8 mins)

[SLIDE - "The Kubernetes Networking Rules"]

"Kubernetes has ONE fundamental networking rule. Memorize this:

**Every pod can communicate with every other pod, on any node, WITHOUT NAT.**

This is called the 'flat network' model."

[SLIDE - Diagram of flat network]

"Let me draw this out:

```
Node 1 (192.168.1.10)          Node 2 (192.168.1.11)
┌─────────────────────┐        ┌─────────────────────┐
│                     │        │                     │
│  Pod A              │        │  Pod C              │
│  IP: 10.128.0.5     │◄──────►│  IP: 10.129.0.8     │
│                     │        │                     │
│  Pod B              │        │  Pod D              │
│  IP: 10.128.0.6     │        │  IP: 10.129.0.9     │
│                     │        │                     │
└─────────────────────┘        └─────────────────────┘
```

Pod A (10.128.0.5) wants to reach Pod C (10.129.0.8).

From Pod A's perspective, it just sends a packet to 10.129.0.8. It doesn't know or care that Pod C is on a different node. It just works."

[SLIDE - How the SDN Makes This Work]

"Here's what happens under the hood:

1. Pod A sends packet to 10.129.0.8
2. OVN on Node 1 intercepts it
3. OVN wraps (encapsulates) the packet inside another packet
4. The outer packet is addressed to Node 2's REAL IP (192.168.1.11)
5. The physical network delivers it to Node 2
6. OVN on Node 2 unwraps it
7. The inner packet gets delivered to Pod C

The physical network only sees traffic between 192.168.1.10 and 192.168.1.11. It has no idea there's pod traffic inside!"

[SLIDE - VPN analogy]

"Think of it like a VPN. When you use a VPN, your browser thinks it's talking directly to a website. But actually, everything is wrapped in a tunnel to the VPN server first.

Same concept. Pods think they're on one big flat network. But underneath, it's tunnels between nodes."

---

### Section 2: OVN-Kubernetes - The Modern Network (8 mins)

[SLIDE - "The Evolution of OpenShift Networking"]

"Quick history lesson:

- **OpenShift 3.x and early 4.x:** Used 'OpenShift SDN'
- **OpenShift 4.12+:** OVN-Kubernetes became the default
- **OpenShift 4.14+:** OpenShift SDN is officially DEPRECATED"

[SLIDE - Deprecation Warning]

"> ⚠️ **IMPORTANT**: OpenShift SDN is deprecated as of OpenShift 4.14!
>
> - New clusters can ONLY use OVN-Kubernetes
> - Existing clusters must migrate to OVN-Kubernetes
> - Red Hat will remove OpenShift SDN support in a future release"

[SLIDE - OVN-Kubernetes Features]

"OVN-Kubernetes is the ONLY network plugin for modern OpenShift. Here's what it provides:

| Feature | Description |
|---------|-------------|
| **IPv6 Dual-Stack** | Run IPv4 and IPv6 simultaneously |
| **Egress IP** | Fixed source IPs for outbound traffic (for firewall rules) |
| **Network Policies** | Full Kubernetes-compliant traffic control |
| **IPsec Encryption** | Encrypt pod-to-pod traffic |
| **Admin Network Policy** | Cluster-wide security policies |
| **Hardware Offload** | Use SmartNICs for better performance |

OVN = Open Virtual Network. Built on Open vSwitch (OVS) - battle-tested technology used by VMware, OpenStack, and others."

[TERMINAL]

"Let's verify YOUR cluster uses OVN-Kubernetes:"

```bash
oc get network cluster -o jsonpath='{.status.networkType}'
```

[WHAT THEY'LL SEE]

"You should see:
```
OVNKubernetes
```"

---

### Section 3: Hands-On - Pod Networking in Action (15 mins)

[SLIDE - "Let's PROVE It Works!"]

"Enough theory! Let's actually watch pod networking in action."

[TERMINAL]

"**Step 1: Create a new project**"

```bash
oc new-project network-demo
```

[PAUSE]

"**Step 2: Deploy two pods**"

```bash
oc run pod-a --image=nginx
oc run pod-b --image=nginx
```

"We're using nginx because it has a web server AND network tools."

"**Step 3: Wait for them to be running**"

```bash
oc get pods -w
```

[WHAT THEY'LL SEE]

"Wait until both show `1/1 Running`. Press Ctrl+C to stop watching.
```
NAME    READY   STATUS    
pod-a   1/1     Running   
pod-b   1/1     Running   
```"

[PAUSE - wait for pods]

"**Step 4: Check their IP addresses**"

```bash
oc get pods -o wide
```

[WHAT THEY'LL SEE]

"You'll see something like:
```
NAME    READY   IP           NODE
pod-a   1/1     10.128.0.85  crc
pod-b   1/1     10.128.0.86  crc
```

Those IPs are from the pod network (10.128.x.x), NOT the node's IP!"

"**Step 5: Test pod-to-pod connectivity**"

```bash
# Get pod-b's IP
POD_B_IP=$(oc get pod pod-b -o jsonpath='{.status.podIP}')
echo "Pod B's IP is: $POD_B_IP"

# From pod-a, curl pod-b
oc exec pod-a -- curl -s http://$POD_B_IP
```

[WHAT THEY'LL SEE]

"You'll see nginx's welcome page HTML:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

🎉 Pod A just talked to Pod B across the virtual network!"

---

### Section 4: DNS - The Real Way Pods Communicate (10 mins)

[SLIDE - "Services and DNS"]

"In production, you NEVER hardcode IP addresses. Pods come and go. IPs change.

Instead, we use Services which provide:
1. A stable IP (ClusterIP)
2. A DNS name
3. Load balancing across pods"

[TERMINAL]

"**Step 1: Create a Service for pod-b**"

```bash
oc expose pod pod-b --port=80
```

"**Step 2: Test DNS resolution**"

```bash
oc exec pod-a -- nslookup pod-b
```

[WHAT THEY'LL SEE]

"```
Server:    172.30.0.10
Address:   172.30.0.10#53

Name:      pod-b.network-demo.svc.cluster.local
Address:   172.30.xxx.xxx
```

Breaking this down:
- `172.30.0.10` = CoreDNS server
- `pod-b.network-demo.svc.cluster.local` = Full DNS name
- `172.30.xxx.xxx` = Service ClusterIP (NOT pod IP!)"

"**Step 3: Access using DNS name**"

```bash
oc exec pod-a -- curl -s http://pod-b
```

"Same nginx page! But now using the DNS name."

[SLIDE - "DNS Name Format"]

"The full format is:
```
<service>.<namespace>.svc.cluster.local
```

From the same namespace, you can use just:
```
<service>
```

From different namespace:
```
<service>.<namespace>
```"

---

### Section 5: Troubleshooting Network Issues (5 mins)

[SLIDE - "Debug Commands"]

"When networking breaks, here's your troubleshooting toolkit:"

```bash
# 1. Check DNS
oc exec <pod> -- nslookup kubernetes.default

# 2. Check CoreDNS pods
oc get pods -n openshift-dns

# 3. Check pod-to-pod connectivity
oc exec <pod> -- curl -s http://<target-ip>

# 4. Check external connectivity
oc exec <pod> -- curl -s https://google.com --max-time 5

# 5. Check network type
oc get network cluster -o yaml
```

[SLIDE - Common Problems]

| Problem | Check This |
|---------|------------|
| DNS not resolving | CoreDNS pods in openshift-dns |
| Can't reach external | Egress settings, NetworkPolicy |
| Pod-to-pod fails | NetworkPolicy blocking |
| Service not working | Service selector matches pod labels |

---

### Wrap-up (2 mins)

[SLIDE - "Key Takeaways"]

"Module 7 Summary:

1. **Flat network model** - Every pod can reach every pod
2. **OVN-Kubernetes** - The ONLY network plugin for OpenShift 4.14+
3. **Encapsulation** - Pod traffic wrapped in node-to-node tunnels
4. **Services provide DNS** - `<svc>.<ns>.svc.cluster.local`
5. **Always use DNS** - Never hardcode pod IPs"

---

## 🌍 REAL-WORLD SIGNIFICANCE

### Why This Matters in Production

#### Scenario 1: Microservices Communication
"In a real microservices architecture, you might have:
- 50 frontend pods
- 30 API gateway pods  
- 100 backend service pods
- 20 database pods

ALL of these need to talk to each other seamlessly. The flat network model makes this possible without complex routing configurations."

#### Scenario 2: Multi-Tenant Clusters
"When multiple teams share a cluster, they all get their own namespaces. Team A's pods can talk to Team B's pods using DNS names:
```
team-b-api.team-b-namespace.svc.cluster.local
```
No firewall rules to configure. It just works."

#### Scenario 3: Debugging Production Issues
"When a production incident happens at 3 AM:
```bash
# Quick connectivity check
oc exec frontend-pod -- curl -s http://backend-api:8080/health

# DNS check
oc exec frontend-pod -- nslookup backend-api
```
Understanding pod networking lets you diagnose issues in minutes, not hours."

#### Scenario 4: Security Compliance
"With OVN-Kubernetes, you get:
- **IPsec encryption** - Required for PCI-DSS, HIPAA
- **Network Policies** - Segment traffic between namespaces
- **Egress IP** - All traffic from a namespace exits with a known IP (for firewall rules)"

### Interview Questions You'll Face

1. **"How do pods communicate across nodes?"**
   → Encapsulation/tunneling through the SDN

2. **"What's the difference between Pod IP and Service IP?"**
   → Pod IPs are ephemeral (10.128.x.x), Service IPs are stable (172.30.x.x)

3. **"How does DNS work in Kubernetes?"**
   → CoreDNS resolves `<svc>.<ns>.svc.cluster.local` to ClusterIP

4. **"What network plugin does OpenShift use?"**
   → OVN-Kubernetes (OpenShift SDN is deprecated in 4.14+)

---

## 📝 PRESENTER NOTES

### Before the session:
- Have CRC running
- Pre-create the project if network is slow
- Test all commands yourself first

### Common student questions:

**"Why are pod IPs and service IPs different?"**
→ Different purposes. Pod IPs are per-pod and change. Service IPs are stable. Services load-balance across pods.

**"What if pods are on different nodes?"**  
→ Same result! OVN handles the encapsulation. Demo works the same regardless of node placement.

### If something fails:

**Pods stuck in ContainerCreating:**
→ Wait longer, or check events: `oc describe pod <name>`

**curl times out:**
→ Check if it's an SCC issue. Try `nginxinc/nginx-unprivileged` image.

**nslookup command not found:**
→ Use `getent hosts <name>` instead.
