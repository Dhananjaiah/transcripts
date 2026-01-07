# Module 18: Resource Constraints
## Lecture Transcript (~25 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"What happens when one team uses ALL the cluster resources? Everyone else suffers.

Resource Quotas and LimitRanges prevent this. Let's set them up!"

---

### Section 1: Requests vs Limits (5 mins)

[SLIDE - Requests vs Limits]

"Every container should specify:

**Requests** - Guaranteed minimum. Used for scheduling.
**Limits** - Hard cap. Container gets OOMKilled if exceeded."

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

"Translation: 'I need at least 100m CPU, but never give me more than 500m'"

---

### Section 2: ResourceQuota (8 mins)

[SLIDE - ResourceQuota]

"ResourceQuota caps TOTAL resources in a namespace:"

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

[TERMINAL]

[DEMO]

```bash
oc new-project quota-demo

# Apply quota
oc apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
EOF

# Check it
oc describe resourcequota demo-quota
```

"Now try creating more than 5 pods - it will fail!"

---

### Section 3: LimitRange (6 mins)

[SLIDE - LimitRange]

"LimitRange sets defaults and constraints PER container:"

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
```

"Now any pod without resource specs gets these defaults automatically!"

---

### Wrap-up (4 mins)

[SLIDE - Key Takeaways]

"Module 18 and Phase 7 complete!

1. **ResourceQuota = namespace total** - Cap overall usage

2. **LimitRange = per-pod defaults** - No pod without limits

3. **Requests for scheduling** - Must fit on node

4. **Limits for enforcement** - Hard cap

5. **Use both together** - Complete governance"

[SLIDE - Next Phase]

"Final phase: Phase 8 - Modern Tech! GitOps and Virtualization."

---

## 📝 Presenter Notes

- Create quota BEFORE LimitRange to show the 'no resources' error
- T-shirt sizing (S/M/L) is a common pattern - mention it
- ResourceQuota is key for multi-tenant clusters
