# Module 11: Storage Operations
## Lecture Transcript (~25 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Storage isn't just 'set and forget.' You need to:
- Expand volumes when they fill up
- Take snapshots for backups
- Restore when disaster strikes

Let's cover these operations!"

---

### Section 1: Expanding PVCs (6 mins)

[SLIDE - Expansion]

"Your database is running out of space. How do you grow the PVC?"

```bash
# Current size
oc get pvc my-data -o jsonpath='{.spec.resources.requests.storage}'

# Patch to new size
oc patch pvc my-data --type merge --patch '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
```

"That's it! The CSI driver expands the underlying volume, the filesystem grows on next pod mount."

[SLIDE - Requirements]

"Requirements:
1. StorageClass has `allowVolumeExpansion: true`
2. Only INCREASE is allowed, never shrink
3. Some drivers require pod restart"

---

### Section 2: VolumeSnapshots (8 mins)

[SLIDE - Snapshots]

"Snapshots are point-in-time copies. Perfect for:
- Pre-upgrade backups
- Development clones
- Disaster recovery"

[SLIDE - Snapshot YAML]

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-db-snapshot
spec:
  volumeSnapshotClassName: csi-aws-snapclass
  source:
    persistentVolumeClaimName: my-database-pvc
```

"Create a snapshot, and you have a recovery point!"

[SLIDE - Restore]

"To restore, create a new PVC from the snapshot:"

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-db
spec:
  dataSource:
    name: my-db-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

### Section 3: Backup Best Practices (6 mins)

[SLIDE - OADP]

"For enterprise backups, use OADP - OpenShift API for Data Protection.

It backs up:
- Kubernetes resources (Deployments, ConfigMaps)
- Persistent volumes
- Cluster configuration

All in one operation."

[SLIDE - Strategy]

"Recommended backup strategy:

**Daily:** VolumeSnapshots of databases
**Weekly:** Full OADP backup of critical namespaces  
**Monthly:** Full cluster backup

Store backups off-cluster! S3, Azure Blob, etc."

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 11 and Phase 4 complete!

1. **Expand with patch** - Just increase the size in PVC spec

2. **Snapshots for backups** - Point-in-time recovery

3. **Restore creates new PVC** - From snapshot reference

4. **OADP for enterprise** - Full namespace backup/restore

5. **Test restores regularly** - Backups are useless if you can't restore"

[SLIDE - Next Phase]

"Next: Phase 5 - Security! Authentication, RBAC, and SCCs."

---

## 📝 Presenter Notes

- VolumeSnapshots may not work on CRC - explain conceptually
- OADP is an Operator - show in OperatorHub if time permits
- Emphasize: "A backup you haven't tested is not a backup"
