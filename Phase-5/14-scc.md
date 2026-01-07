# Module 14: Pod Security (SCCs)
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Here's a scenario: You try to deploy a Helm chart. It works on vanilla Kubernetes. On OpenShift? CreateContainerConfigError.

Why? Security Context Constraints. Let's understand and fix them!"

---

### Section 1: What SCCs Control (6 mins)

[SLIDE - SCC controls]

"SCCs control what a pod CAN and CANNOT do:

- Run as root? ❌ or ✅
- Access host network? ❌ or ✅
- Run privileged containers? ❌ or ✅
- Use specific volume types? ❌ or ✅

By default, OpenShift is RESTRICTIVE. That's a good thing!"

[SLIDE - Built-in SCCs]

"From least to most privileged:

**restricted** - Default. Very locked down.
**nonroot** - Can run as any non-root UID
**anyuid** - Can run as any UID including root
**privileged** - Full access (dangerous!)"

---

### Section 2: The Root Problem (8 mins)

[SLIDE - Why Helm fails]

"Most Docker images assume root:

```dockerfile
FROM ubuntu
USER root
RUN apt-get install ...
```

OpenShift says: 'Nope! Your ServiceAccount uses restricted SCC. No root for you!'

Result: `CreateContainerConfigError`"

[TERMINAL]

[DEMO]

```bash
# Check which SCC a pod uses
oc get pod <name> -o yaml | grep -A1 "openshift.io/scc"
```

---

### Section 3: Fixing SCC Issues (12 mins)

[SLIDE - Fix options]

"Three ways to fix:

**Option 1: Fix the image** (Best)
Use a non-root image:"

```bash
# Use nginx-unprivileged instead of nginx
oc run web --image=nginxinc/nginx-unprivileged
```

[TERMINAL]

"**Option 2: Grant anyuid to ServiceAccount**"

```bash
# Create ServiceAccount
oc create sa my-app-sa

# Grant anyuid
oc adm policy add-scc-to-user anyuid -z my-app-sa

# Use in deployment
oc set serviceaccount deployment/my-app my-app-sa
```

"**Option 3: Create custom SCC** (for specific needs)"

"Always prefer Option 1. Use Option 2 sparingly. Option 3 for special cases."

---

### Section 4: Troubleshooting Flow (5 mins)

[SLIDE - Debug flow]

"When a pod fails with SCC issues:

1. Check error: `oc describe pod <name>`
2. Check current SCC: `oc get pod <name> -o yaml | grep scc`
3. Check what SCC allows: `oc describe scc restricted`
4. Fix: Use different image OR grant appropriate SCC"

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 14 and Phase 5 complete!

1. **restricted is default** - And that's good for security

2. **Most images expect root** - OpenShift blocks this

3. **Grant SCC to ServiceAccount** - Not directly to pods

4. **anyuid is common fix** - But try non-root images first

5. **privileged = last resort** - Only for monitoring agents"

[SLIDE - Next Phase]

"Next: Phase 6 - Observability! Monitoring and logging."

---

## 📝 Presenter Notes

- SCC issues are THE most common OpenShift problem
- Always check `oc describe pod` first - error messages are helpful
- Many popular images have -unprivileged variants - use them!
