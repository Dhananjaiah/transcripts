# Module 17: Builds & ImageStreams
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Phase 7! Today we learn something uniquely OpenShift:

Building container images INSIDE the cluster, directly from source code. No Dockerfile required!"

---

### Section 1: Source-to-Image (S2I) (7 mins)

[SLIDE - S2I concept]

"Traditional way: Write Dockerfile → Build locally → Push to registry → Deploy

S2I way: Point at Git repo → OpenShift builds → Deploys automatically"

[SLIDE - S2I flow]

"1. You provide: Source code (Git URL)
2. OpenShift provides: Builder image (nodejs, python, etc.)
3. S2I: Combines them into runnable image
4. Result: Your app, containerized, deployed"

---

### Section 2: LiveDemo - Build from Git (15 mins)

[TERMINAL]

[DEMO]

```bash
# Create project
oc new-project builds-demo

# Build Node.js app from GitHub
oc new-app nodejs:18~https://github.com/sclorg/nodejs-ex.git
```

"That's it! Watch the magic..."

```bash
# Watch the build
oc logs -f buildconfig/nodejs-ex

# Check created resources
oc get all
```

"See? BuildConfig, ImageStream, Deployment, Service - all created!"

```bash
# Expose it
oc expose svc/nodejs-ex

# Get URL
oc get route nodejs-ex
```

"Open that URL - your app is running!"

---

### Section 3: ImageStreams (6 mins)

[SLIDE - ImageStream]

"ImageStreams track container images:

- Abstract the registry URL
- Track image history
- Trigger redeployments on changes"

```bash
# Check ImageStream
oc get imagestream nodejs-ex

# See tags
oc describe imagestream nodejs-ex
```

"When you push a new build, the ImageStream updates, and your Deployment auto-redeploys!"

---

### Section 4: Triggering Builds (5 mins)

[SLIDE - Triggers]

"Builds can be triggered by:

- **Manual**: `oc start-build <name>`
- **Git webhook**: Push to repo
- **Image change**: Base image updates
- **Config change**: BuildConfig modified"

```bash
# Manual trigger
oc start-build nodejs-ex

# Watch it
oc logs -f build/nodejs-ex-2
```

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 17 complete!

1. **S2I = Build from source** - No Dockerfile needed

2. **oc new-app is powerful** - Creates everything

3. **ImageStreams track images** - History and triggers

4. **Webhooks for CI/CD** - Push triggers build

5. **Auto-redeploy on new image** - Zero manual work"

[SLIDE - Next Module]

"Next: Resource Constraints - Quotas and Limits!"

---

## 📝 Presenter Notes

- nodejs-ex is a simple demo app that builds quickly
- If build fails, check logs with `oc logs build/<name>`
- S2I is unique to OpenShift - highlight this!
