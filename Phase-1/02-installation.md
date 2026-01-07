# Module 2: Installation Scenarios
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome back! In Module 2, we're covering how OpenShift clusters actually get installed.

Now, we're using CRC - so we won't actually run an installation. But understanding this is CRUCIAL because:
1. You'll interview for jobs that ask about it
2. You need to understand install-config.yaml for Day 2 operations
3. It explains WHY certain things are configured the way they are"

[CHECK]

"Has anyone here installed Kubernetes from scratch? With kubeadm maybe? That experience will help you appreciate what OpenShift does differently."

---

### Section 1: Two Installation Paths (8 mins)

[SLIDE - IPI vs UPI]

"There are TWO ways to install OpenShift 4. Think of them like buying furniture..."

[PAUSE]

"**IPI - Installer Provisioned Infrastructure**
This is like ordering from IKEA with assembly included. You say 'I want a cluster on AWS' and the installer creates EVERYTHING:
- VPC and networks
- Load balancers
- DNS records
- VMs
- Security groups

You just provide credentials and wait."

[SLIDE - UPI section]

"**UPI - User Provisioned Infrastructure**
This is like ordering raw lumber and building your own furniture. YOU create:
- The network
- The load balancers  
- The DNS entries
- The VMs or bare metal servers

Then the installer just configures OpenShift on top."

[CHECK]

"Quick question - which one sounds easier? ...Right, IPI. So why would ANYONE choose UPI?"

---

### Section 2: Why UPI Exists (5 mins)

[SLIDE - Enterprise requirements]

"Great question. Let me tell you some real stories...

**The Bank Story:**
A major bank I worked with couldn't use IPI. Why? Their security team controls ALL network configuration. They have to approve every firewall rule. The installer can't just create things.

**The Telco Story:**  
A telecom company runs everything in air-gapped networks. No internet connection. They have to mirror all images internally and use custom DHCP servers.

**The Government Story:**
A government agency requires specific security baselines and hardware. They can't use generic cloud VMs."

[PAUSE]

"UPI exists because enterprises have CONSTRAINTS. They've been running datacenters for decades with established processes."

[SLIDE - Summary table]

"So here's the simple rule:
- **IPI** = Cloud, Dev/Test, Speed matters
- **UPI** = Regulated industries, On-prem, Control matters"

---

### Section 3: The Bootstrap Process (8 mins)

[SLIDE - Bootstrap diagram]

"Now here's the fascinating part - how does the cluster actually come to life?"

[SLIDE - Timeline animation]

"It starts with a TEMPORARY node called the Bootstrap node. Think of it as a midwife - it helps deliver the babies but doesn't stick around."

[Step through diagram]

"Step 1: Bootstrap node starts
- It runs a temporary API server
- It runs a temporary etcd
- It generates configuration for the real masters

Step 2: Master nodes boot
- They contact the Bootstrap node
- They download their configuration
- They start the REAL etcd cluster
- They start the REAL control plane

Step 3: Bootstrap complete
- Masters take over completely
- Bootstrap node can be destroyed
- You'll never need it again

Step 4: Workers join
- Workers boot and contact the API
- They get their configuration
- They join the cluster"

[CHECK]

"Here's a tricky interview question: What happens if the Bootstrap node dies AFTER the masters are running?"

[PAUSE for answer]

"Nothing! The Bootstrap node's job is done. The masters are self-sufficient now."

---

### Section 4: The install-config.yaml (10 mins)

[SLIDE - install-config.yaml]

"Let me show you the SINGLE most important file in any OpenShift installation."

[TERMINAL - show file]

```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: my-cluster
```

"This is where EVERYTHING is configured. Let's break it down..."

[Walk through sections]

"**baseDomain** - This is your DNS domain. Combined with the cluster name, it creates:
- `api.my-cluster.example.com` - API endpoint
- `*.apps.my-cluster.example.com` - Application URLs"

[SLIDE - Control plane section]

```yaml
controlPlane:
  replicas: 3
  platform:
    aws:
      type: m5.xlarge
```

"**controlPlane** - ALWAYS 3 replicas in production. Why 3? Because etcd needs a quorum. 3 nodes can survive 1 failure."

[SLIDE - Worker section]

```yaml
compute:
  - replicas: 6
    platform:
      aws:
        type: m5.2xlarge
```

"**compute** - These are your workers. Start with what you need, scale later. This is the only thing that's easy to change after install."

[SLIDE - Networking section]

```yaml
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
  serviceNetwork:
    - 172.30.0.0/16
  networkType: OVNKubernetes
```

"**networking** - Pod and Service IPs. Pick these carefully - you CANNOT change them later!

And networkType - OVNKubernetes or OpenShiftSDN. Also cannot change easily."

[CHECK]

"What if you pick the wrong network CIDR and it overlaps with your corporate network?"

[PAUSE]

"You rebuild the cluster. Seriously. Plan this carefully."

---

### Section 5: Key Decisions (5 mins)

[SLIDE - Can Change / Cannot Change table]

"Let me save you from future pain. Here's what you CAN and CANNOT change after installation..."

[Walk through table]

"**CANNOT Change:**
- baseDomain - Baked into certificates
- metadata.name - Everywhere
- clusterNetwork CIDR - In every node config
- serviceNetwork CIDR - Same
- networkType - Fundamental to cluster

**CAN Change:**
- Worker replicas - Just scale MachineSets
- Worker instance types - Update MachineSets
- StorageClasses - Add anytime
- Identity Providers - Configure later"

[PAUSE]

"The lesson? Spend time on install-config.yaml. Get it RIGHT the first time."

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Let's recap:

1. **IPI vs UPI** - Easy vs Controlled. Pick based on constraints.

2. **Bootstrap is temporary** - It delivers the cluster, then disappears.

3. **install-config.yaml is sacred** - Many settings are permanent.

4. **Network planning matters** - CIDRs can't be changed.

5. **Always 3 masters** - etcd quorum requirement."

[SLIDE - Next Module]

"Next up: Machine Management - how to manage nodes at scale without SSH. See you there!"

---

## 📝 Presenter Notes

- Students often ask "can we see a real installation?" - Show YouTube video if needed
- Common confusion: "Why can't we change network CIDRs?" - Because every node has them configured
- Have a sample install-config.yaml file ready to show
- Some students mix up Bootstrap and Master - use the "midwife" analogy
