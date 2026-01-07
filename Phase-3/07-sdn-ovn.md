# Module 7: SDN vs OVN-Kubernetes
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Phase 3 - Networking! The connectivity layer.

Networking is where most Kubernetes confusion lives. But today, we'll demystify it. Starting with: How do pods actually talk to each other?"

[CHECK]

"Quick question: in a traditional network, every server has an IP on the same subnet, right? But in Kubernetes, pods get IPs from a DIFFERENT network. How do they communicate? That's what we'll learn."

---

### Section 1: The Flat Network Model (6 mins)

[SLIDE - Flat network diagram]

"Kubernetes has a simple rule: Every pod can talk to every other pod, directly, without NAT.

This is called the 'flat network' model."

[SLIDE - Pod IPs]

"Look at this cluster:
- Node 1: 192.168.1.10, runs Pod A (10.128.0.5)
- Node 2: 192.168.1.11, runs Pod B (10.128.2.3)

Pod A wants to talk to Pod B. The IP 10.128.2.3 isn't real on the physical network. So how does the packet get there?"

[PAUSE]

"That's where the Software Defined Network (SDN) comes in."

[SLIDE - SDN concept]

"The SDN creates a VIRTUAL network overlay. Packets between pods get encapsulated, sent to the right node, then decapsulated.

Think of it like a VPN tunnel between nodes."

---

### Section 2: OpenShift SDN vs OVN-Kubernetes (7 mins)

[SLIDE - SDN types]

"OpenShift has two SDN options:

**OpenShift SDN** - The original, based on Open vSwitch
**OVN-Kubernetes** - The new default, also based on OVS but more powerful"

[SLIDE - Comparison table]

"Key differences:

| Feature | OpenShift SDN | OVN-Kubernetes |
|---------|--------------|----------------|
| IPv6 | No | Yes |
| Egress IP | Basic | Advanced |
| Network Policy | Basic | Full |
| Hybrid mode | No | Yes |
| Default in 4.12+ | No | Yes |"

[PAUSE]

"If you're on 4.12 or newer, you're probably using OVN-Kubernetes."

[TERMINAL]

[DEMO]

```bash
# Check which SDN you're using
oc get network cluster -o jsonpath='{.status.networkType}'
```

---

### Section 3: LiveDemo - Explore Pod Networking (12 mins)

[SLIDE - Demo time]

"Let's see how pod networking works in practice!"

[TERMINAL]

[DEMO]

```bash
# Create a namespace
oc new-project network-demo

# Deploy two pods
oc run pod-a --image=nginx
oc run pod-b --image=nginx

# Wait for running
oc get pods -w
```

[Wait for pods]

```bash
# Get pod IPs - notice they're from the pod CIDR, not node IPs
oc get pods -o wide
```

"See those IPs? 10.128.x.x - that's the pod network, not the physical network."

```bash
# Let's test connectivity
# Get pod-b's IP
POD_B_IP=$(oc get pod pod-b -o jsonpath='{.status.podIP}')
echo $POD_B_IP

# From pod-a, ping pod-b
oc exec pod-a -- curl -s $POD_B_IP:80
```

"It works! Pod A talked to Pod B using the virtual network."

[SLIDE - DNS magic]

"But we don't usually use IPs. We use DNS!"

```bash
# Services have DNS names
oc expose pod pod-b --port=80

# Now pod-a can use the service name
oc exec pod-a -- curl -s pod-b.network-demo.svc.cluster.local
```

"See? `pod-b.network-demo.svc.cluster.local` - that's the full DNS name."

---

### Section 4: Debugging Network Issues (6 mins)

[SLIDE - Troubleshooting]

"Networks break. Here's how to debug on OpenShift."

[TERMINAL]

[DEMO]

```bash
# Check DNS resolution
oc exec pod-a -- nslookup kubernetes.default

# Check if pod can reach external internet
oc exec pod-a -- curl -s -o /dev/null -w "%{http_code}" https://google.com

# Get pod events for network issues
oc describe pod pod-a | grep -A5 Events
```

[SLIDE - Common issues]

"Common problems:

1. **DNS not working** - Check CoreDNS pods in openshift-dns
2. **Can't reach external** - Check egress rules, network policies
3. **Pod to pod fails** - Check if NetworkPolicy is blocking
4. **No IP assigned** - Node network plugin issue"

```bash
# Check CoreDNS is healthy
oc get pods -n openshift-dns
```

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 7 done!

1. **Flat network model** - Every pod can reach every pod

2. **SDN creates overlay** - Virtual network on top of physical

3. **OVN-Kubernetes is default** - Full feature set, IPv6, advanced Egress

4. **Services provide DNS** - `<svc>.<namespace>.svc.cluster.local`

5. **Debug with exec** - curl, nslookup from inside pods"

[SLIDE - Next Module]

"Next: Ingress and Egress - How does the OUTSIDE world reach your pods?"

---

## 📝 Presenter Notes

- Use `curl` or `wget` in demo pods, nginx has both
- If DNS fails, check CoreDNS pods first
- Students often confuse pod CIDR with service CIDR - clarify
- OVN is more complex to debug than OpenShiftSDN - stick to basics
