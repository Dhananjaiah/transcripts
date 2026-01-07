# Module 7: SDN vs OVN-Kubernetes (Pod Networking)
## Lecture Transcript (~50 minutes)

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

[CHECK]

"But how? The physical switch between Node 1 and Node 2 has no idea what 10.129.0.8 is. That IP isn't in its routing tables."

[PAUSE]

[SLIDE - "Software Defined Networking to the Rescue"]

"This is where SDN - Software Defined Networking - comes in.

Think of it like this: We're building an INVISIBLE tunnel system between all the nodes.

When Pod A sends a packet to 10.129.0.8:
1. The SDN on Node 1 intercepts it
2. It wraps (encapsulates) the packet inside another packet
3. The outer packet is addressed to Node 2's REAL IP (192.168.1.11)
4. The physical network delivers it to Node 2
5. Node 2's SDN unwraps it
6. The inner packet gets delivered to Pod C

The physical network only sees traffic between 192.168.1.10 and 192.168.1.11. It has no idea there's pod traffic inside!"

[SLIDE - VPN analogy]

"Think of it like a VPN. When you use a VPN, your browser thinks it's talking directly to a website. But actually, everything is wrapped in a tunnel to the VPN server first.

Same concept. Pods think they're on one big flat network. But underneath, it's tunnels between nodes."

---

### Section 2: OpenShift SDN vs OVN-Kubernetes (10 mins)

[SLIDE - "Two Network Plugins"]

"OpenShift gives you two options for implementing this virtual network:

1. **OpenShift SDN** - The original. Been around since OpenShift 3.
2. **OVN-Kubernetes** - The new default since OpenShift 4.12.

Both use Open vSwitch (OVS) under the hood. But they work quite differently."

[SLIDE - Comparison Table]

"Let me break down the key differences:

| Feature | OpenShift SDN | OVN-Kubernetes |
|---------|---------------|-----------------|
| **IPv6 Support** | ❌ No | ✅ Yes |
| **Egress IP** | Basic | Advanced (multiple IPs) |
| **Network Policies** | Basic | Full Kubernetes compliance |
| **Windows Nodes** | ❌ No | ✅ Yes |
| **Hybrid Overlay** | ❌ No | ✅ Yes (mix of modes) |
| **Default since 4.12** | ❌ No | ✅ Yes |

[PAUSE]

"Bottom line: If you're on a modern OpenShift cluster (4.12+), you're using OVN-Kubernetes. It's more capable."

[CHECK]

"Question: Why would anyone use OpenShift SDN today?

Answer: Legacy. They upgraded from older versions and didn't switch. But new clusters should always use OVN."

[TERMINAL]

[SLIDE - "Let's check what YOU'RE using"]

"Time for our first command. Let's find out which network plugin your cluster uses."

```bash
oc get network cluster -o jsonpath='{.status.networkType}'
```

[BREAKDOWN]

"Let me explain this command:
- `oc get network` - Get the Network configuration resource
- `cluster` - There's only one, and it's named 'cluster'
- `-o jsonpath='{.status.networkType}'` - Extract just the networkType field

"

[WHAT THEY'LL SEE]

"You should see either:
```
OVNKubernetes
```
or
```
OpenShiftSDN
```

On CRC, it's OVNKubernetes."

[PAUSE]

"Let's also check the network CIDRs - the IP ranges used for pods and services:"

```bash
oc get network cluster -o yaml | grep -A5 clusterNetwork
```

[WHAT THEY'LL SEE]

"You'll see something like:
```yaml
clusterNetwork:
- cidr: 10.128.0.0/14
  hostPrefix: 23
```

This means:
- `10.128.0.0/14` - The overall pod network (about 262,000 IPs)
- `hostPrefix: 23` - Each node gets a /23 (512 IPs for pods on that node)"

[ADDITIONAL CONTEXT]

"Let's also check the service network:"

```bash
oc get network cluster -o yaml | grep serviceNetwork
```

"You'll see something like:
```yaml
serviceNetwork:
- 172.30.0.0/16
```

This is where ClusterIP services get their IPs. Different from the pod network!"

---

### Section 3: Let's PROVE Pod Networking Works! (15 mins)

[SLIDE - "Hands-On Time!"]

"Enough theory! Let's actually watch pod networking in action."

[TERMINAL]

"**Step 1: Create a new project**"

```bash
oc new-project network-demo
```

"We're using a fresh project so we don't interfere with anything else."

[PAUSE]

"**Step 2: Deploy two pods**

We'll use nginx because it has both a web server AND network tools like curl."

```bash
oc run pod-a --image=nginx
oc run pod-b --image=nginx
```

[EXPLAIN]

"We're creating two simple pods:
- `pod-a` - Our first pod
- `pod-b` - Our second pod

They're running nginx, which listens on port 80."

"**Step 3: Wait for them to be running**"

```bash
oc get pods -w
```

[WHAT THEY'LL SEE]

"You'll see:
```
NAME    READY   STATUS              RESTARTS   AGE
pod-a   0/1     ContainerCreating   0          5s
pod-b   0/1     ContainerCreating   0          3s
pod-a   1/1     Running             0          8s
pod-b   1/1     Running             0          10s
```

Wait until both show `1/1 Running`. Press Ctrl+C to stop watching."

[PAUSE - wait for pods]

"**Step 4: Check their IP addresses**

This is where it gets interesting!"

```bash
oc get pods -o wide
```

[WHAT THEY'LL SEE]

"You'll see something like:
```
NAME    READY   STATUS    IP           NODE
pod-a   1/1     Running   10.128.0.85  crc
pod-b   1/1     Running   10.128.0.86  crc
```

Look at those IP addresses! They're from the pod network (10.128.x.x), not the node's IP."

[PAUSE]

[SLIDE - "Now Let's Test Connectivity!"]

"**Step 5: Get pod-b's IP address into a variable**"

```bash
POD_B_IP=$(oc get pod pod-b -o jsonpath='{.status.podIP}')
echo "Pod B's IP is: $POD_B_IP"
```

[WHAT THEY'LL SEE]

"Should print something like: `Pod B's IP is: 10.128.0.86`"

"**Step 6: THE MOMENT OF TRUTH - Can pod-a reach pod-b?**"

```bash
oc exec pod-a -- curl -s http://$POD_B_IP
```

[BREAKDOWN]

"Let me explain:
- `oc exec pod-a` - Run a command inside pod-a
- `--` - Everything after this goes to the container
- `curl -s http://$POD_B_IP` - Make an HTTP request to pod-b's IP
- `-s` means silent (no progress bar)"

[WHAT THEY'LL SEE]

"You'll see the nginx welcome page HTML:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

🎉 IT WORKS! Pod A just talked to Pod B using the virtual network!"

[CELEBRATE]

"Think about what just happened:
1. We ran curl inside pod-a
2. It sent a request to pod-b's virtual IP
3. The SDN delivered that packet
4. nginx in pod-b responded
5. We got the response back in pod-a

All across this invisible virtual network!"

---

### Section 4: DNS - The Real Way Pods Communicate (8 mins)

[SLIDE - "Nobody Uses IP Addresses"]

"In the real world, you don't hardcode IP addresses. Pods come and go. IPs change.

Instead, we use DNS names. And OpenShift has built-in DNS!"

[TERMINAL]

"**Step 1: Create a Service for pod-b**

Services give pods a stable DNS name."

```bash
oc expose pod pod-b --port=80
```

[EXPLAIN]

"This creates a Service named `pod-b` that points to our pod. Now pod-b has a DNS name!"

"**Step 2: Test DNS resolution**

Let's prove DNS works:"

```bash
oc exec pod-a -- nslookup pod-b
```

[WHAT THEY'LL SEE]

"You'll see:
```
Server:    172.30.0.10
Address:   172.30.0.10#53

Name:      pod-b.network-demo.svc.cluster.local
Address:   172.30.xxx.xxx
```

Breaking this down:
- `Server: 172.30.0.10` - That's the cluster's DNS server (CoreDNS)
- `Name: pod-b.network-demo.svc.cluster.local` - The full DNS name
- `Address: 172.30.xxx.xxx` - The Service's ClusterIP (not the pod IP!)"

[PAUSE - important distinction]

"Notice the Service has a DIFFERENT IP than the pod! The Service IP is from the `serviceNetwork` (172.30.x.x), not the pod network (10.128.x.x).

Why? Because the Service is an abstraction. It could have multiple pods behind it. It's a stable endpoint."

"**Step 3: Access pod-b using DNS**"

```bash
oc exec pod-a -- curl -s http://pod-b
```

[WHAT THEY'LL SEE]

"Same nginx welcome page! But this time we used the DNS name `pod-b`, not the IP."

[SLIDE - "DNS Name Formats"]

"Let me teach you the DNS naming convention:

**Full name (FQDN):**
```
pod-b.network-demo.svc.cluster.local
```

Broken down:
- `pod-b` - Service name
- `network-demo` - Namespace
- `svc` - It's a Service (not a pod)
- `cluster.local` - Default cluster domain

**From same namespace, you can use short form:**
```
pod-b
```

**From different namespace:**
```
pod-b.network-demo
```
"

[TERMINAL]

"Let's prove the short name works:"

```bash
oc exec pod-a -- curl -s http://pod-b.network-demo.svc.cluster.local
```

"Same result! Both the short and long name work."

---

### Section 5: Debugging Network Issues (8 mins)

[SLIDE - "When Things Break"]

"Networks fail. Here's how to troubleshoot pod networking issues like a pro."

[TERMINAL]

"**Debug Tool 1: Check DNS Resolution**

If pods can't find each other by name, it might be DNS:"

```bash
oc exec pod-a -- nslookup kubernetes.default
```

[WHAT THEY'LL SEE]

"You should see:
```
Server:    172.30.0.10
Address:   172.30.0.10#53

Name:      kubernetes.default.svc.cluster.local
Address:   172.30.0.1
```

If this fails, DNS is broken. Check CoreDNS pods."

"**Debug Tool 2: Check CoreDNS pods**"

```bash
oc get pods -n openshift-dns
```

[WHAT THEY'LL SEE]

"You should see healthy DNS pods:
```
NAME                  READY   STATUS
dns-default-xxxxx     2/2     Running
dns-default-yyyyy     2/2     Running
```

If they're not Running, you found your problem!"

"**Debug Tool 3: Check connectivity to external internet**"

```bash
oc exec pod-a -- curl -s -o /dev/null -w "%{http_code}\n" https://google.com --max-time 5
```

[BREAKDOWN]

"- `-s` - Silent
- `-o /dev/null` - Discard the page content
- `-w \"%{http_code}\\n\"` - Just print the HTTP status code
- `--max-time 5` - Timeout after 5 seconds"

[WHAT THEY'LL SEE]

"You should see `200` (success). If you see nothing or an error, egress is blocked."

"**Debug Tool 4: Get raw pod events**"

```bash
oc describe pod pod-a | tail -20
```

"The Events section at the bottom shows network-related issues like 'NetworkNotReady'."

[SLIDE - "Common Network Problems"]

"Here's a cheat sheet of issues I see most often:

| Problem | Symptom | Solution |
|---------|---------|----------|
| DNS not working | `nslookup` fails | Check openshift-dns pods |
| Can't reach external | `curl google.com` times out | Check Egress/NetworkPolicy |
| Pod to pod fails | `curl $POD_IP` times out | Check NetworkPolicy |
| No pod IP assigned | Pod stuck in ContainerCreating | Check node network plugin |
| Service doesn't work | DNS resolves but curl fails | Check Service selector matches pod labels |

"

---

### Wrap-up (3 mins)

[SLIDE - "What We Learned Today"]

"Let's recap Module 7:

1. **Flat network model** - Every pod can reach every pod, directly

2. **SDN creates the overlay** - Virtual network on top of physical

3. **OVN-Kubernetes is the modern default** - IPv6, full Network Policy, Windows support

4. **Pod IPs vs Service IPs** - Different networks! 10.128.x.x vs 172.30.x.x

5. **DNS for service discovery** - `<service>.<namespace>.svc.cluster.local`

6. **Debugging order** - DNS first, then connectivity, then policies"

[SLIDE - "Cheat Sheet"]

```bash
# Check network type
oc get network cluster -o jsonpath='{.status.networkType}'

# Get pod IPs
oc get pods -o wide

# Test pod-to-pod
oc exec <pod> -- curl -s http://<target-pod-ip>

# Test DNS
oc exec <pod> -- nslookup <service-name>

# Check CoreDNS
oc get pods -n openshift-dns
```

[SLIDE - "Next Up"]

"Next module: Ingress and Egress! 

We learned how pods talk to EACH OTHER. But how does the OUTSIDE world reach your pods? And how do your pods reach the internet?

That's what Routes, Ingress, and Egress IPs are for. See you in Module 8!"

[CHECK]

"Any questions about pod networking before we wrap up?"

---

## 📝 Presenter Notes

### Before the session:
- Have CRC running
- Pre-create the project if network is slow
- Test all commands yourself first

### Common student questions:

**"Why are pod IPs and service IPs different?"**
→ Different purposes. Pod IPs are per-pod and change. Service IPs are stable. Services load-balance across pods.

**"Can I SSH into the node to see the network?"**
→ In production, no (immutable). On CRC, you can with `oc debug node/crc`. But we avoid this.

**"What if pods are on different nodes?"**
→ Same result! SDN handles the encapsulation. The demo works the same regardless of node placement.

### If something fails:

**Pods stuck in ContainerCreating:**
→ Wait longer, or check events: `oc describe pod <name>`

**curl times out:**
→ Check if it's an SCC issue. nginx-unprivileged might work better.

**nslookup command not found:**
→ Some minimal images don't have it. Use `getent hosts <name>` instead.

### Extensions if you have time:
- Show `oc get endpoints` to see how Services discover pods
- Demonstrate what happens when you scale up: IPs added to endpoint
- Show the actual OVS flows (advanced, requires node access)
