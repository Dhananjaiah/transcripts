# Module 8: Ingress & Egress
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Pod networking is great, but... how does the OUTSIDE world access your apps? And how do your apps reach external services?

That's Ingress and Egress. Let's dive in!"

---

### Section 1: Routes - OpenShift's Secret Weapon (8 mins)

[SLIDE - Routes concept]

"Kubernetes has Ingress resources. OpenShift has something better: Routes.

Routes came BEFORE Kubernetes Ingress existed. And honestly? They're still more powerful."

[SLIDE - Route flow]

"Here's the flow:
1. User requests `myapp.apps.cluster.example.com`
2. DNS points to the OpenShift Router
3. Router looks up which Service matches that hostname
4. Router forwards to Service
5. Service forwards to Pod
6. Pod responds"

[SLIDE - Simple Route YAML]

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-route
spec:
  host: myapp.apps.cluster.example.com
  to:
    kind: Service
    name: my-service
  port:
    targetPort: 8080
```

"That's it! DNS name → Service → Pods."

---

### Section 2: TLS Termination Types (8 mins)

[SLIDE - TLS types]

"HTTPS traffic needs TLS. Where do you terminate it?"

[SLIDE - Three options]

"**Edge Termination**
TLS ends at the Router. Traffic inside cluster is plain HTTP.
Use for: Most applications."

```yaml
spec:
  tls:
    termination: edge
```

"**Passthrough Termination**
Router doesn't decrypt. It passes encrypted traffic straight to pods.
Use for: Apps that handle their own TLS."

```yaml
spec:
  tls:
    termination: passthrough
```

"**Re-encrypt Termination**
Router decrypts, then re-encrypts to the pod.
Use for: End-to-end encryption requirements."

```yaml
spec:
  tls:
    termination: reencrypt
```

[CHECK]

"Which would you use for a typical web app? ...Edge. Let the router handle TLS."

---

### Section 3: LiveDemo - Create a Route (10 mins)

[TERMINAL]

[DEMO]

```bash
# Create app
oc new-project routes-demo
oc run web --image=nginx --port=80
oc expose pod web --port=80

# Create basic route
oc expose svc web

# Get the URL
oc get route web
```

"There's our URL! Let's test it..."

```bash
curl http://$(oc get route web -o jsonpath='{.spec.host}')
```

"Now let's add TLS..."

```bash
# Create edge-terminated TLS route
oc create route edge secure-web --service=web

# Get new URL
oc get route secure-web
```

```bash
curl https://$(oc get route secure-web -o jsonpath='{.spec.host}') -k
```

"HTTPS working! The `-k` is because CRC uses self-signed certs."

---

### Section 4: Egress Basics (5 mins)

[SLIDE - Egress concept]

"Ingress is traffic IN. Egress is traffic OUT.

By default, pods can reach any external IP. But enterprises often need:
- Fixed source IPs for firewall rules
- Controlled egress through proxies"

[SLIDE - Egress IP]

"With Egress IPs, all traffic from a namespace appears to come from a specific IP.

Your database team says 'allow 10.0.0.50 through the firewall.' You configure that as an Egress IP for your namespace."

[SLIDE - Theory note]

"Egress IPs require multiple nodes to work properly - we can't fully demo on CRC. But the concept is essential for enterprise environments."

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 8 done!

1. **Routes expose services externally** - Hostname → Service → Pods

2. **Three TLS types** - Edge (common), Passthrough, Re-encrypt

3. **`oc expose`** - Quick way to create routes

4. **Egress IPs** - Fixed source IP for firewall rules

5. **Routes > Ingress** - More features, OpenShift native"

[SLIDE - Next Module]

"Next: Network Security with Network Policies!"

---

## 📝 Presenter Notes

- TLS termination is a common interview question
- If curl fails, check if the service selector matches the pod labels
- Edge TLS with OpenShift-provided certs is production-ready
