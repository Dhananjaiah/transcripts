# Module 6: The Internal Registry
## Lecture Transcript (~45 minutes)

---

### Opening (3 mins)

[SLIDE - Title: "The Internal Registry"]

"Alright everyone, welcome back! Today we're covering something that will make your life SO much easier - OpenShift's built-in container registry.

Let me ask you a question first..."

[CHECK]

"When you build a Docker image on your laptop, where do you push it? Docker Hub? Some company registry? 

And then when you deploy to Kubernetes, it has to pull that image back down. Round trip through the internet. Every. Single. Time."

[PAUSE - let them think]

"What if the registry was INSIDE your cluster? That's what we're learning today.

By the end of this module, you will:
- Understand why OpenShift has its own registry
- Know what ImageStreams are (this is OpenShift-specific!)
- Actually push your own image to the cluster
- Deploy a pod using that image

Let's go!"

---

### Section 1: Why Does OpenShift Have Its Own Registry? (6 mins)

[SLIDE - "The Problem with External Registries"]

"Before I explain the solution, let me explain the problem..."

[SLIDE - Show external registry diagram]

"Here's what happens with external registries:

1. You build image on laptop
2. Push to Docker Hub (goes over internet)
3. Deploy to cluster
4. Every node pulls from Docker Hub (over internet again!)
5. Docker Hub has rate limits
6. Docker Hub goes down? Your deployments fail.

Now imagine you have 50 nodes. Each one pulling images. Every deployment."

[PAUSE]

"It's slow. It's risky. It costs money if you need more pulls."

[SLIDE - "The OpenShift Solution"]

"OpenShift says: 'Why not put a registry INSIDE the cluster?'

Here are the benefits - and I want you to really understand each one:

**Benefit 1: Speed**
Image is stored in-cluster. When a node needs it, it's a short hop on the internal network. Not across the internet.

**Benefit 2: Security**
Your proprietary code never leaves your infrastructure. It's not sitting on some public registry.

**Benefit 3: No Internet Required**
Air-gapped clusters (like banks, military, government) - they work perfectly.

**Benefit 4: No Rate Limits**
Docker Hub limits how many pulls you can do. Your own registry? Unlimited.

**Benefit 5: Automatic Integration**
When you build with S2I (we'll learn that later), the image goes straight to this registry. No extra steps."

[CHECK]

"Does this make sense? This registry is especially important in enterprise environments. Any questions?"

---

### Section 2: What is an ImageStream? (10 mins)

[SLIDE - "ImageStreams - OpenShift's Secret Weapon"]

"Now before we touch the registry, I need to teach you something that confuses EVERYONE at first. But once you get it, it's beautiful.

It's called an ImageStream."

[PAUSE]

"Let me explain with a problem..."

[SLIDE - "The nginx:latest Problem"]

"Say you deploy nginx:latest. Simple, right? But think about it:

- WHERE is nginx:latest? Docker Hub? Quay? Your private registry?
- nginx:latest today might be nginx:1.25
- nginx:latest next week might be nginx:1.26
- Your deployment has NO IDEA when the image changes

You could be running different versions on different nodes and not even know it!"

[PAUSE - let them realize the problem]

"This is a real production problem. I've seen outages caused by this."

[SLIDE - "ImageStream to the Rescue"]

"An ImageStream is like a POINTER to an image. But a smart pointer.

Think of it like a bookmark that also remembers the exact page number.

Here's what an ImageStream does:
1. Points to an image (anywhere - Docker Hub, internal registry, wherever)
2. Remembers the EXACT digest (SHA256 hash) of that image
3. Can NOTIFY your deployments when the image changes
4. Keeps history of previous versions

So instead of saying 'use nginx:latest wherever that is', you say 'use my-imagestream:latest which I know points to digest abc123'."

[SLIDE - Show ImageStream YAML]

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: my-app
spec:
  tags:
    - name: latest
      from:
        kind: DockerImage
        name: docker.io/nginx:latest
```

"Let me break this down line by line:

- `kind: ImageStream` - We're creating an ImageStream, not an image
- `name: my-app` - This is what we'll reference in our deployments
- `tags` - Like Docker tags (latest, v1.0, etc.)
- `from` - Where does this tag ACTUALLY point to?
- `kind: DockerImage` - It's a regular Docker/OCI image
- `name: docker.io/nginx:latest` - The actual image location

Once this exists, you don't reference `docker.io/nginx:latest` in your deployments. You reference `my-app:latest`. OpenShift handles the rest."

[CHECK]

"I know this seems like extra work. But trust me - when we get to automated builds and triggers, you'll see why this is genius. Any questions so far?"

---

### Section 3: Let's Explore the Registry! (10 mins)

[SLIDE - "Hands-On Time!"]

"Enough theory. Let's actually see this registry."

[TERMINAL]

"First, let's check if the registry is running. Remember, it's managed by an Operator."

```bash
oc get clusteroperator image-registry
```

[EXPLAIN WHAT THEY'LL SEE]

"You should see something like:
```
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED
image-registry   4.14.x    True        False         False
```

True, False, False - that's what we want. Healthy."

[PAUSE - let them run it]

"Now let's see the actual pods running the registry:"

```bash
oc get pods -n openshift-image-registry
```

[EXPLAIN]

"You'll see pods named like `image-registry-xxx`. These ARE the registry. Running right inside your cluster."

"Now here's something cool. OpenShift ships with pre-loaded ImageStreams for building apps. Let's look:"

```bash
oc get imagestreams -n openshift
```

[PAUSE - let them run it]

"WHOA! Look at all those! 

You'll see: nodejs, python, php, ruby, java, dotnet, httpd...

These are your SOURCE-TO-IMAGE builders. When you want to build a Node.js app, OpenShift uses these. We'll cover that in Phase 7.

But here's the key point - all these images are tracked as ImageStreams. OpenShift knows exactly what version of each one you're using."

---

### Section 4: Expose the Registry (8 mins)

[SLIDE - "Pushing from Outside"]

"Here's a question: If the registry is inside the cluster, how do I push images from my laptop?

Two options:
1. Build inside the cluster (Source-to-Image, we'll learn later)
2. Expose the registry with a Route so we can reach it

Let's do option 2!"

[TERMINAL]

"First, let's enable the external route:"

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type merge \
  --patch '{"spec":{"defaultRoute":true}}'
```

[BREAKDOWN THE COMMAND]

"Let me explain this:
- `oc patch` - We're modifying a resource
- `configs.imageregistry.operator.openshift.io/cluster` - The registry's config (long name, I know!)
- `--type merge` - Merge our changes with existing config
- `--patch '{"spec":{"defaultRoute":true}}'` - Enable the route

This tells the Registry Operator: 'Hey, create a Route so people can reach you from outside.'"

[PAUSE - wait for it to apply]

"Now let's get that Route:"

```bash
oc get routes -n openshift-image-registry
```

[WHAT THEY'LL SEE]

"You should see:
```
NAME            HOST/PORT                                          
default-route   default-route-openshift-image-registry.apps.crc.testing
```

That long URL? That's how we'll reach the registry from our laptop.

Let's save it to a variable so we don't have to keep typing it:"

```bash
REGISTRY=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
echo $REGISTRY
```

"You should see the URL printed. We'll use this in a moment."

---

### Section 5: Push Your Own Image! (12 mins)

[SLIDE - "The Fun Part - Push an Image!"]

"Alright, this is where it gets exciting. We're going to:
1. Create a project for our images
2. Login to the registry
3. Pull an image from Docker Hub
4. Tag it for our registry
5. Push it
6. Deploy a pod using it

Let's go step by step!"

[TERMINAL]

"**Step 1: Create a project**"

```bash
oc new-project image-demo
```

"Why? In OpenShift, images in the registry are organized by project. Just like folders."

[PAUSE]

"**Step 2: Get our login token**

OpenShift uses your oc login token for registry authentication. Let's grab it:"

```bash
TOKEN=$(oc whoami -t)
echo $TOKEN
```

"You'll see a long string of characters. That's your token. Don't share it!"

[PAUSE]

"**Step 3: Login to the registry**

Now we use podman (or docker) to login:"

```bash
podman login $REGISTRY -u $(oc whoami) -p $TOKEN
```

[EXPLAIN]

"What's happening here:
- `podman login` - Authenticate with a registry
- `$REGISTRY` - Our registry URL from earlier
- `-u $(oc whoami)` - Username is our OpenShift username
- `-p $TOKEN` - Password is our token

You should see: `Login Succeeded!`"

[TROUBLESHOOTING]

"If you get a TLS error, the registry uses a self-signed certificate. Add `--tls-verify=false`:
```bash
podman login $REGISTRY -u $(oc whoami) -p $TOKEN --tls-verify=false
```"

[PAUSE - make sure everyone is logged in]

"**Step 4: Pull a small image**

Let's get busybox - it's tiny and perfect for testing:"

```bash
podman pull docker.io/library/busybox:latest
```

"This downloads busybox from Docker Hub to your laptop."

[PAUSE]

"**Step 5: Tag it for our registry**

Docker images need to be tagged with the destination registry. Think of it as addressing an envelope:"

```bash
podman tag busybox:latest $REGISTRY/image-demo/my-busybox:v1
```

[EXPLAIN]

"Breaking down the tag:
- `$REGISTRY` - Our registry URL
- `/image-demo` - The project name
- `/my-busybox` - We're naming our image
- `:v1` - Our version tag

The format is: REGISTRY/PROJECT/IMAGE:TAG"

[PAUSE]

"**Step 6: Push!**"

```bash
podman push $REGISTRY/image-demo/my-busybox:v1
```

"If TLS error, add `--tls-verify=false` here too."

[WAIT - this takes a moment]

"Watch the layers upload... Done!"

[CELEBRATE]

"🎉 You just pushed your first image to the OpenShift internal registry!"

---

### Section 6: Verify and Use the Image (8 mins)

[SLIDE - "Did it work?"]

"Let's verify everything worked."

[TERMINAL]

"**Check the ImageStream was created:**"

```bash
oc get imagestreams -n image-demo
```

[WHAT THEY'LL SEE]

"You should see:
```
NAME          IMAGE REPOSITORY                                              TAGS
my-busybox    default-route-openshift-image-registry.apps.../my-busybox    v1
```

Wait - we didn't create an ImageStream! We just pushed an image!

Here's the magic: When you push to the internal registry, OpenShift AUTOMATICALLY creates an ImageStream for you. No extra work needed."

[PAUSE - let that sink in]

"Let's see the details:"

```bash
oc describe imagestream my-busybox -n image-demo
```

"You'll see the tags, the actual image digest (SHA256), and history."

[SLIDE - "Now let's use it!"]

"**Deploy a pod using our image:**"

```bash
oc run test-pod \
  --image=image-registry.openshift-image-registry.svc:5000/image-demo/my-busybox:v1 \
  --command -- sleep 3600
```

[BREAKDOWN]

"Wait, what's that long image URL? Let me explain:
- `image-registry.openshift-image-registry.svc:5000` - This is the INTERNAL address
- We used the external route to push, but pods use the internal address
- Format: `service.namespace.svc:port/project/image:tag`"

"Check it's running:"

```bash
oc get pods
```

"You should see:
```
NAME       READY   STATUS    
test-pod   1/1     Running
```

RUNNING! 🎉 Your pod is using YOUR image from YOUR internal registry!"

---

### Section 7: The Magic of ImageStream Triggers! (15 mins)

[SLIDE - "Why ImageStreams are AMAZING"]

"Okay, now I'm going to show you the REAL power of ImageStreams. This is the 'aha!' moment.

Here's the scenario:
- You have a Deployment running version 1 of your app
- You build version 2 of your app
- You push v2 to the registry
- **The Deployment automatically updates to v2!**

No `oc set image`. No editing YAML. No manual intervention. It just... happens."

[CHECK]

"Sound magical? Let's prove it!"

[TERMINAL]

"First, let's clean up and start fresh with a proper Deployment (not just a pod):"

```bash
# Delete the test pod
oc delete pod test-pod

# Create a Deployment that uses ImageStream triggers
oc create deployment webapp --image=image-registry.openshift-image-registry.svc:5000/image-demo/my-busybox:v1 --dry-run=client -o yaml > webapp-deployment.yaml
```

[EXPLAIN]

"We're creating the YAML file first so we can add the trigger annotation."

"Now let's edit the file to add the ImageStream trigger. Open webapp-deployment.yaml:"

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    # THIS IS THE MAGIC LINE!
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"my-busybox:latest"},"fieldPath":"spec.template.spec.containers[?(@.name==\"webapp\")].image"}]'
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: my-busybox:latest   # Just reference the ImageStream!
        command: ["sleep", "3600"]
```

[BREAKDOWN THE TRIGGER ANNOTATION]

"Let me explain that scary-looking annotation:

```json
{
  \"from\": {
    \"kind\": \"ImageStreamTag\",     // Watch an ImageStream
    \"name\": \"my-busybox:latest\"   // Specifically the 'latest' tag
  },
  \"fieldPath\": \"spec.template.spec.containers[?(@.name==\\\"webapp\\\")].image\"
  // When ImageStream changes, update THIS field in the Deployment
}
```

In plain English: 'When my-busybox:latest changes, update the webapp container's image automatically.'"

[TERMINAL]

"But wait - we don't have a 'latest' tag yet. Let's create one by tagging our v1:"

```bash
# Tag v1 as latest
oc tag my-busybox:v1 my-busybox:latest
```

[WHAT THEY'LL SEE]

"You should see:
```
Tag my-busybox:latest set to my-busybox@sha256:abc123...
```"

"Now let's apply the deployment:"

```bash
oc apply -f webapp-deployment.yaml
```

"Check it's running:"

```bash
oc get pods -l app=webapp
```

"You should see 2 pods running. Let's check which image they're using:"

```bash
oc get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].image}'
```

[WHAT THEY'LL SEE]

"You'll see something like: `image-registry.openshift-image-registry.svc:5000/image-demo/my-busybox@sha256:abc123...`

Notice it's using the SHA digest, not just 'latest'. OpenShift resolved the tag to the exact image!"

[PAUSE - important concept]

"This is important: Even though we said 'latest', OpenShift pins it to the exact digest. This means you ALWAYS know what you're running."

---

[SLIDE - "Now Let's Update!"]

"Here comes the magic. We're going to:
1. Pull a DIFFERENT version of busybox (1.36 instead of 1.35)
2. Push it as v2
3. Update the ImageStream 'latest' tag to point to v2
4. Watch the Deployment automatically update!"

[TERMINAL]

"**Step 1: Pull a different version**"

```bash
# Pull a specific newer version
podman pull docker.io/library/busybox:1.36
```

"**Step 2: Tag and push as v2**"

```bash
# Tag it for our registry as v2
podman tag busybox:1.36 $REGISTRY/image-demo/my-busybox:v2

# Push it (use --tls-verify=false if needed)
podman push $REGISTRY/image-demo/my-busybox:v2
```

[PAUSE - wait for push]

"**Step 3: Check our ImageStream now has v1 AND v2**"

```bash
oc get imagestream my-busybox
```

[WHAT THEY'LL SEE]

"You should see:
```
NAME          IMAGE REPOSITORY                         TAGS
my-busybox    .../image-demo/my-busybox               latest,v1,v2
```

Both v1 and v2 are there! But 'latest' still points to v1."

"**Step 4: Update 'latest' to point to v2**"

```bash
oc tag my-busybox:v2 my-busybox:latest
```

[WHAT HAPPENS NEXT - EXPLAIN SLOWLY]

"NOW watch your pods..."

```bash
oc get pods -l app=webapp -w
```

[WHAT THEY'LL SEE]

"You should see something like:
```
NAME                      READY   STATUS              
webapp-abc123-xxx         1/1     Running           
webapp-abc123-yyy         1/1     Running           
webapp-def456-aaa         0/1     ContainerCreating  <-- NEW POD!
webapp-def456-aaa         1/1     Running
webapp-abc123-xxx         1/1     Terminating        <-- OLD POD DYING!
...
```

THE DEPLOYMENT IS ROLLING OUT NEW PODS AUTOMATICALLY! 🎉"

[CELEBRATE]

"We didn't run `oc set image`. We didn't edit the Deployment. We didn't do ANYTHING to the Deployment.

We just updated the ImageStream tag, and OpenShift said: 'Oh, my-busybox:latest changed! Let me update the Deployment for you.'"

[PAUSE - let it sink in]

"**Verify the new image is being used:**"

```bash
oc get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].image}'
```

"You'll see a DIFFERENT sha256 digest now - it's using v2!"

---

[SLIDE - "Why This Matters"]

"Let me explain why this is HUGE for production:

**Scenario 1: Continuous Deployment**
- Your CI/CD pipeline builds a new image
- Pipeline pushes to registry with tag 'latest'
- ALL deployments referencing 'latest' auto-update
- Zero manual work!

**Scenario 2: Rollback**
- Something's broken in v2
- Just run: `oc tag my-busybox:v1 my-busybox:latest`
- Deployments roll back automatically!

**Scenario 3: Promotion**
- Dev environment uses 'dev' tag
- QA environment uses 'qa' tag  
- Prod uses 'prod' tag
- Promote by: `oc tag my-busybox:qa my-busybox:prod`
- Production updates automatically!"

[CHECK]

"Now do you see why ImageStreams exist? They're not just 'smart pointers' - they're the foundation of GitOps-style deployments in OpenShift.

Any questions before we wrap up?"

---

### Wrap-up (3 mins)

[SLIDE - "What We Learned"]

"Let's recap what we accomplished today:

1. **The registry is built-in** - No need to set up anything. It's already there.

2. **ImageStreams are smart pointers** - They track images, their versions, and can trigger updates.

3. **Expose for external push** - We patched the config to create a route.

4. **Login with your oc token** - `podman login $URL -u $(oc whoami) -p $(oc whoami -t)`

5. **Tag format matters** - `Registry/Project/Image:Tag`

6. **Push creates ImageStream** - Automatic! You don't have to create it manually.

7. **Internal URL for pods** - `image-registry.openshift-image-registry.svc:5000`

8. **🌟 ImageStream Triggers are POWERFUL** - Update the tag, Deployment auto-updates. No manual intervention. This is the foundation of CI/CD in OpenShift!"

[SLIDE - "Cheat Sheet"]

"Quick commands you'll use constantly:"

```bash
# Expose registry
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type merge --patch '{"spec":{"defaultRoute":true}}'

# Get registry URL
REGISTRY=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')

# Login
podman login $REGISTRY -u $(oc whoami) -p $(oc whoami -t)

# Push
podman push $REGISTRY/project/image:tag
```

[SLIDE - "Phase 2 Complete!"]

"Congratulations! You've finished Phase 2! You now understand:
- Operators and how they manage everything
- Cluster updates and versioning
- The internal registry and ImageStreams

Next up: Phase 3 - Networking! How do pods talk to each other?"

[CHECK]

"Any final questions about the registry?"

---

## 📝 Presenter Notes

### Before the session:
- Have CRC running
- Have podman/docker installed and working
- Test the registry push yourself first
- Pre-pull busybox to save time if network is slow

### Common issues and fixes:

**"x509: certificate signed by unknown authority"**
→ Add `--tls-verify=false` to podman commands

**"unauthorized: authentication required"**
→ Token expired. Run `oc login` again, then get new token

**"no route to host"**
→ Registry route not created. Check `oc get routes -n openshift-image-registry`

**"imagePullBackOff when deploying"**
→ Check the internal registry URL is correct: `image-registry.openshift-image-registry.svc:5000`

### Timing guidance:
- If running short on time, skip "Section 4: Expose the Registry" and just explain it
- The "Push an Image" section is the most important demo
- Students always love seeing their image actually work

### Extension activities:
- Push a custom app image they built
- Show how ImageStream triggers work with deployments
- Demonstrate `oc import-image` for mirroring external images
