# Module 16: The Logging Stack
## Lecture Transcript (~25 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Containers are ephemeral. When a pod dies, its logs go with it... unless you have centralized logging!

OpenShift's logging stack collects logs from everywhere and puts them in one place."

---

### Section 1: Logging Architecture (6 mins)

[SLIDE - Architecture]

"The logging stack has three parts:

**Collector** (Vector/Fluentd) - Runs on every node, collects logs
**Storage** (Loki/Elasticsearch) - Stores and indexes logs
**Visualization** (Console/Kibana) - Search and view"

[SLIDE - Log types]

"Three types of logs:

- **Application** - Your containers' stdout/stderr
- **Infrastructure** - System components, nodes
- **Audit** - API server security events"

---

### Section 2: Viewing Logs (8 mins)

[TERMINAL]

[DEMO]

"Without the logging stack, use oc logs:"

```bash
# View pod logs
oc logs <pod-name>

# Follow logs
oc logs -f <pod-name>

# Previous container (after crash)
oc logs <pod-name> --previous

# All pods with label
oc logs -l app=myapp --all-containers
```

[BROWSER]

"With the console: Observe → Logs
Filter by namespace, pod, severity, search text..."

---

### Section 3: Log Forwarding (6 mins)

[SLIDE - ClusterLogForwarder]

"For enterprise environments, forward logs to external systems:"

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
spec:
  outputs:
    - name: splunk
      type: splunk
      url: https://splunk.company.com:8088
  pipelines:
    - name: app-to-splunk
      inputRefs:
        - application
      outputRefs:
        - splunk
```

"Supports: Splunk, Elasticsearch, Kafka, CloudWatch, Syslog..."

---

### Wrap-up (3 mins)

[SLIDE - Key Takeaways]

"Module 16 and Phase 6 complete!

1. **oc logs for quick access** - Always available

2. **Logging Operator for centralization** - Install from OperatorHub

3. **Loki is lightweight** - Good for most cases

4. **Forward to external SIEM** - ClusterLogForwarder

5. **Use JSON logs** - Better searchability"

[SLIDE - Next Phase]

"Next: Phase 7 - Application Management! Builds and resource quotas."

---

## 📝 Presenter Notes

- Logging Operator is optional and resource-heavy
- `oc logs -f --since=5m` is great for recent logs
- JSON logs are industry best practice
