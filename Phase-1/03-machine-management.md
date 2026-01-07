# Module 3: Machine Management
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Module 3! Today we're answering the question every sysadmin asks:

'If I can't SSH into my nodes... how do I manage them???'

The answer is the Machine API - and it's going to change how you think about infrastructure."

---

### Section 1: The Machine API Concept (7 mins)

[SLIDE - Machine API overview]

"Let me paint a picture. In traditional infrastructure, you have servers. Each one is special. You name them. You SSH into them. You run scripts.

In OpenShift 4, nodes are managed like... Pods!"

[PAUSE - let it sink in]

"Think about it:
- Deployment → Pod: 'Give me 3 replicas'
- MachineSet → Machine → Node: 'Give me 3 worker nodes'"

[SLIDE - Hierarchy diagram]

"Here's the hierarchy:

**MachineSet** = Like a Deployment template
- Says 'I want 3 workers of this type'

**Machine** = Like a Pod
- Represents one actual VM
- Gets created/deleted by MachineSet

**Node** = The Kubernetes view
- What the scheduler sees
- Where pods actually run"

[CHECK]

"Does this feel like a pattern you've seen before?"

---

### Section 2: MachineSets in Practice (8 mins)

[SLIDE - MachineSet YAML]

"Let me show you what a MachineSet looks like..."

```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: cluster-worker-us-east-1a
spec:
  replicas: 2
```

"See that `replicas: 2`? Just like a Deployment!"

[SLIDE - Provider section]

```yaml
spec:
  template:
    spec:
      providerSpec:
        value:
          instanceType: m5.xlarge
          placement:
            availabilityZone: us-east-1a
```

"The providerSpec tells the cloud provider exactly what to create. Instance type, AZ, disk size..."

[TERMINAL - show scaling]

```bash
# In a real cluster, you would:
oc get machinesets -n openshift-machine-api
```

"To scale? It's this simple:"

```bash
oc scale machineset cluster-worker-us-east-1a --replicas=5
```

[PAUSE]

"Five minutes later? Three new VMs appear, boot RHCOS, and join as worker nodes. Automatically."

[CHECK]

"How would you do this with traditional infrastructure? Write a Terraform script? Use Ansible? SSH into machines?"

---

### Section 3: MachineConfig - The SSH Replacement (10 mins)

[SLIDE - MachineConfig]

"NOW we get to the exciting part. MachineConfig is how you configure nodes WITHOUT SSH."

[PAUSE]

"Think about what you typically SSH into a server to do:
- Edit /etc/chrony.conf
- Add a kernel parameter
- Install a driver
- Add a user

MachineConfig handles ALL of this."

[SLIDE - MachineConfig example]

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-chrony
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/chrony.conf
          mode: 0644
          contents:
            source: data:text/plain;base64,<content>
```

"Let me explain this:
- **name**: Starts with priority (99 means late in boot)
- **role: worker**: Apply to worker nodes only
- **files.path**: The file I want to create/modify
- **contents**: The actual file content (base64 encoded)"

[SLIDE - What happens next]

"When you apply a MachineConfig, here's what happens:

1. Machine Config Operator sees the change
2. Creates a new 'rendered' config (combines all MachineConfigs)
3. Cordons a node (no new pods)
4. Drains the node (moves pods away)
5. Reboots the node with new config
6. Uncordons the node
7. Moves to next node"

[CHECK]

"Notice it does one node at a time? Why do you think that is?"

[PAUSE for answer]

"Exactly! To maintain availability. Your apps keep running on other nodes."

---

### Section 4: Kernel Parameters & More (5 mins)

[SLIDE - Kernel parameters]

"MachineConfig isn't just for files. Watch this..."

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-hugepages
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  kernelArguments:
    - "hugepagesz=2M"
    - "hugepages=100"
```

"Kernel parameters! No recompiling, no manual grub editing. Just YAML."

[SLIDE - Systemd services]

```yaml
spec:
  config:
    systemd:
      units:
        - name: my-monitoring-agent.service
          enabled: true
          contents: |
            [Unit]
            Description=My Agent
            [Service]
            ExecStart=/usr/bin/my-agent
            [Install]
            WantedBy=multi-user.target
```

"Systemd services! Want to run something on every node? MachineConfig."

[PAUSE]

"The pattern is: ANYTHING you'd do via SSH, there's a MachineConfig way."

---

### Section 5: Cluster Autoscaler (5 mins)

[SLIDE - Autoscaling]

"Let's talk about the cherry on top - automatic node scaling."

[SLIDE - Diagram]

"Imagine: Your app is under heavy load. Pods are Pending because there's not enough capacity."

"With Cluster Autoscaler:
1. Autoscaler sees Pending pods
2. Checks if adding a node would help
3. Increases MachineSet replicas
4. New node boots and joins
5. Pending pods get scheduled"

[SLIDE - ClusterAutoscaler YAML]

```yaml
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 20
  scaleDown:
    enabled: true
    unneededTime: 10m
```

"maxNodesTotal: 20 - Never exceed 20 nodes (cost control!)
unneededTime: 10m - If a node is idle for 10 minutes, remove it"

[CHECK]

"Why would you set a maximum? What could go wrong without one?"

[PAUSE]

"Cloud bill shock! Imagine a bug creates infinite pods. Without a max, you get infinite nodes and infinite bills."

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Let's recap Module 3:

1. **MachineSets are like Deployments** - For nodes instead of pods

2. **MachineConfig replaces SSH** - Files, kernel params, services - all in YAML

3. **Updates are rolling** - One node at a time, safe and controlled

4. **Autoscaler adds intelligence** - Scale up on demand, down when idle

5. **No more snowflakes** - Every node is identical, reproducible"

[SLIDE - CRC Note]

"Quick note: CRC is a single-node cluster, so we can't demo MachineSets. But the concepts apply to any production cluster."

[SLIDE - Next Phase]

"Congratulations! You've completed Phase 1 - the infrastructure layer. Next up: Phase 2, Core Platform Operations. We'll learn about Operators, Updates, and the Registry!"

---

## 📝 Presenter Notes

- CRC limitations: No MachineSets/Machines to show, only MachineConfigs
- Common question: "What if a MachineConfig breaks the node?" - MCO has some safety checks, but test in dev first!
- Emphasize the cultural shift from "fix the server" to "replace the server"
- Some students worry about data loss during node drains - explain PVCs persist
