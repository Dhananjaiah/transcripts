# Module 20: OpenShift Virtualization
## Lecture Transcript (~25 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Our final module! Here's a question: What do you do with workloads that CAN'T be containerized?

Windows Server. Legacy apps. Things that need a full VM. OpenShift Virtualization runs them ALONGSIDE containers!"

---

### Section 1: Why VMs on Kubernetes? (6 mins)

[SLIDE - The problem]

"The reality:
- Not everything can be containerized
- Windows Server with Active Directory
- Legacy apps with no source code
- Apps requiring specific kernel versions

You still need VMs. But you want ONE platform to manage."

[SLIDE - The solution]

"OpenShift Virtualization (based on KubeVirt):
- VMs run as pods
- Same networking as containers
- Same storage as containers
- Same management tools
- Gradual migration path"

---

### Section 2: How It Works (8 mins)

[SLIDE - Architecture]

"Each VM runs inside a 'virt-launcher' pod:

Pod → Container → libvirt → QEMU/KVM → Your VM

The VM gets a pod IP. It can talk to other pods. Services work. Routes work. Everything just works!"

[SLIDE - VirtualMachine resource]

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel8-vm
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 2
        memory:
          guest: 4Gi
        devices:
          disks:
            - name: rootdisk
              disk: {}
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: rhel8-disk
```

"It's YAML! Just like everything else in Kubernetes."

---

### Section 3: VM Operations (5 mins)

[SLIDE - Operations]

"Manage VMs like pods:

```bash
# List VMs
oc get vms

# Start/Stop
virtctl start rhel8-vm
virtctl stop rhel8-vm

# Console access
virtctl console rhel8-vm

# VNC access
virtctl vnc rhel8-vm
```"

[SLIDE - Migration]

"Live migration - move VMs between nodes with zero downtime. Just like VMware vMotion!"

---

### Section 4: CRC Limitation (2 mins)

[SLIDE - CRC note]

"Important: CRC CANNOT run OpenShift Virtualization. It needs:
- Bare metal or nested virtualization
- 8+ CPU cores
- 32GB+ RAM

This module is theory. But in production environments, it's incredibly powerful!"

---

### Wrap-up & Course Conclusion (5 mins)

[SLIDE - Key Takeaways]

"Module 20 complete!

1. **VMs and containers together** - One platform

2. **VMs are pods** - Same networking, storage, tools

3. **Gradual migration** - Containerize at your pace

4. **Live migration** - Enterprise-grade

5. **For when containers aren't enough** - Legacy, Windows"

[SLIDE - Course Complete]

"🎉 CONGRATULATIONS! 🎉

You've completed The Complete OpenShift Atlas!

You now understand:
- Architecture & Installation
- Operators & Updates
- Networking & Security
- Storage & Observability
- Application Management
- GitOps & Virtualization

You're ready to:
- Manage production OpenShift clusters
- Pass the EX280/EX380 exams
- Architect container platforms
- Join the OpenShift community"

[SLIDE - Next Steps]

"What now?

1. Practice on CRC daily
2. Join OpenShift Commons
3. Pursue Red Hat certification
4. Build real projects
5. Keep learning - Service Mesh, Serverless, AI/ML..."

[PAUSE]

"Thank you for learning with me. Any final questions?"

[SLIDE - Thank You]

"Happy Engineering! 🚀"

---

## 📝 Presenter Notes

- This is the conclusion - make it celebratory!
- Emphasize the journey they've completed
- Encourage certification path
- If time, take questions about the whole course
- End on an inspiring note
