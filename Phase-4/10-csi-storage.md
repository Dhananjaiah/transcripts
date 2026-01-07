# Module 10: CSI & Dynamic Provisioning
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Phase 4 - Storage! The persistence layer.

Containers are ephemeral. They die, they get replaced, data is lost. Unless... you use persistent storage. Let's learn how!"

---

### Section 1: The Storage Stack (7 mins)

[SLIDE - Stack diagram]

"Here's how storage works in OpenShift, from top to bottom:

**Your App** → needs to store data
**PVC (Claim)** → 'I need 10Gi of fast storage'
**PV (Volume)** → The actual disk resource  
**StorageClass** → 'Use this type of storage'
**CSI Driver** → Talks to the actual storage system"

[CHECK]

"Think of it like ordering a pizza:
- PVC = Your order ('I want a large pepperoni')
- StorageClass = The pizza style ('New York thin crust')
- CSI Driver = The pizza oven
- PV = The actual pizza delivered"

---

### Section 2: StorageClasses (6 mins)

[SLIDE - StorageClass]

"StorageClasses define TYPES of storage available."

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
```

"Key fields:
- **provisioner** - Which CSI driver to use
- **parameters** - Driver-specific settings (SSD type, IOPS)
- **reclaimPolicy** - Delete or Retain when PVC is deleted
- **allowVolumeExpansion** - Can we resize?"

[TERMINAL]

```bash
# See available storage classes
oc get storageclasses
```

---

### Section 3: LiveDemo - Create and Use PVCs (12 mins)

[TERMINAL]

[DEMO]

```bash
# Create project
oc new-project storage-demo

# See what storage classes exist
oc get sc
```

"Now let's create a PVC..."

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

```bash
# Check status
oc get pvc
```

"See the 'Bound' status? Storage was provisioned!"

"Now let's use it in a pod..."

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-data
EOF
```

"Write some data..."

```bash
oc exec storage-pod -- sh -c 'echo "Hello Persistent Storage!" > /data/test.txt'
oc exec storage-pod -- cat /data/test.txt
```

"Delete and recreate the pod..."

```bash
oc delete pod storage-pod
# Recreate (same command as before)
```

```bash
oc exec storage-pod -- cat /data/test.txt
```

"Data survived! That's persistence."

---

### Section 4: Access Modes (5 mins)

[SLIDE - Access Modes]

"**ReadWriteOnce (RWO)** - One node can mount read/write. Used for databases.

**ReadOnlyMany (ROX)** - Many nodes can mount read-only. Used for shared configs.

**ReadWriteMany (RWX)** - Many nodes can mount read/write. Used for shared file storage."

[SLIDE - Provider support]

"Important: Block storage (EBS, Azure Disk) only supports RWO. For RWX, you need file storage (EFS, Azure Files, NFS)."

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 10 complete!

1. **PVC is your request** - StorageClass fulfills it

2. **CSI is the standard** - All modern storage uses CSI

3. **ReclaimPolicy matters** - Delete for dev, Retain for prod

4. **Access modes vary** - RWO for block, RWX for file

5. **Dynamic provisioning is automatic** - Just create a PVC"

[SLIDE - Next Module]

"Next: Storage Operations - expanding, snapshots, and backups!"

---

## 📝 Presenter Notes

- CRC uses hostpath storage which is simple but limited
- If PVC stays Pending, check events with `oc describe pvc`
- Common error: wrong accessMode for the StorageClass
