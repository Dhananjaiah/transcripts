# Module 9: Network Security (Network Policies)
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"By default, every pod can talk to every pod. That's convenient... and terrifying.

What if someone compromises one pod? They can reach EVERYTHING. Today we fix that with Network Policies."

---

### Section 1: Zero Trust Networking (5 mins)

[SLIDE - Zero Trust]

"Zero Trust means: Trust nothing. Verify everything.

In network terms: Block everything by default.Only allow what's explicitly needed."

[SLIDE - Without policies]

"Without Network Policies:
- Frontend pod can talk to database directly (bad!)
- Compromised pod can scan all internal services
- No segmentation"

[SLIDE - With policies]

"With Network Policies:
- Frontend → Backend → Database (only allowed path)
- Compromised pod can't reach other namespaces
- Micro-segmentation"

---

### Section 2: Network Policy Basics (8 mins)

[SLIDE - NetworkPolicy structure]

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}      # Which pods this applies to
  policyTypes:
    - Ingress          # Control incoming
    - Egress           # Control outgoing
```

"Empty `podSelector: {}` means ALL pods in the namespace."

[SLIDE - Default Deny]

"Step 1 in any secure namespace: Default Deny All."

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

"Apply this, and NOTHING can talk to anything. Now we add specific allows."

---

### Section 3: LiveDemo - Implement Network Policies (12 mins)

[TERMINAL]

[DEMO]

```bash
# Setup
oc new-project netpol-demo

# Deploy frontend and backend
oc run frontend --image=nginx --labels="tier=frontend"
oc run backend --image=nginx --labels="tier=backend"

# Create services
oc expose pod frontend --port=80
oc expose pod backend --port=80
```

"Let's test connectivity BEFORE policies..."

```bash
# Frontend can reach backend
oc exec frontend -- curl -s backend.netpol-demo.svc:80

# Works!
```

"Now apply default deny..."

```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF
```

"Try again..."

```bash
# This will timeout now!
oc exec frontend -- curl -s --max-time 3 backend.netpol-demo.svc:80
```

"Blocked! Now let's allow frontend → backend..."

```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 80
EOF
```

"And allow DNS egress (or nothing works)..."

```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-dns
      ports:
        - protocol: UDP
          port: 53
EOF
```

"Test again..."

```bash
oc exec frontend -- curl -s --max-time 3 backend.netpol-demo.svc:80
# Works!
```

---

### Section 4: Common Patterns (5 mins)

[SLIDE - Patterns]

"**Pattern 1: Allow from same namespace**"

```yaml
spec:
  ingress:
    - from:
        - podSelector: {}
```

"**Pattern 2: Allow from OpenShift Ingress (Routes)**"

```yaml
spec:
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
```

"**Pattern 3: Allow specific port only**"

```yaml
spec:
  ingress:
    - ports:
        - port: 8080
          protocol: TCP
```

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 9 complete!

1. **Default allow is dangerous** - Start with deny all

2. **podSelector targets pods** - By labels

3. **Don't forget DNS** - Allow egress to openshift-dns

4. **Routes need ingress namespace** - Allow from ingress namespace

5. **Test after each policy** - Easy to break things"

[SLIDE - Phase 3 Complete]

"Phase 3 done! You now understand OpenShift networking inside and out. Next: Storage!"

---

## 📝 Presenter Notes

- DNS egress is the most common forgotten policy - always include it
- If curl times out, policy is working; if curl works, policy isn't applied correctly
- Labels are KEY - double check they match
