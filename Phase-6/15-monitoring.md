# Module 15: The Monitoring Stack
## Lecture Transcript (~35 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Welcome to Phase 6 - Observability! 

OpenShift includes a complete monitoring stack out of the box. Prometheus, Alertmanager, Grafana - all pre-configured. Let's explore!"

---

### Section 1: Built-in Monitoring (6 mins)

[SLIDE - Stack diagram]

"Here's what's running in your cluster right now:

**Prometheus** - Collects and stores metrics
**Alertmanager** - Sends notifications
**Grafana** - Dashboards (read-only)
**Thanos** - Long-term storage"

[TERMINAL]

```bash
# See monitoring pods
oc get pods -n openshift-monitoring
```

"All running! No installation needed."

---

### Section 2: User Workload Monitoring (8 mins)

[SLIDE - Two Prometheus]

"By default, Prometheus only monitors CLUSTER components. For YOUR apps, enable User Workload Monitoring:"

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

"Wait a minute, then:"

```bash
oc get pods -n openshift-user-workload-monitoring
```

"Now you can monitor your own applications!"

---

### Section 3: LiveDemo - Prometheus Queries (12 mins)

[BROWSER]

[DEMO]

"Open the console: Observe → Metrics"

"Let's run some queries..."

```promql
# CPU usage by pod
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))

# Memory usage
container_memory_usage_bytes{namespace="default"}

# Top 5 memory consumers
topk(5, container_memory_usage_bytes)
```

[SLIDE - Useful queries]

"Save these for troubleshooting:

**High CPU pods:**
```promql
topk(10, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))
```

**Pod restarts:**
```promql
changes(kube_pod_container_status_restarts_total[1h])
```"

---

### Section 4: Alerts (5 mins)

[SLIDE - AlertManager]

"Create alerts with PrometheusRule:"

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: high-memory
spec:
  groups:
    - name: memory
      rules:
        - alert: HighMemoryUsage
          expr: container_memory_usage_bytes > 1e9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage detected"
```

"When condition is true for 5 minutes, alert fires!"

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 15 complete!

1. **Monitoring is built-in** - Prometheus, Alertmanager included

2. **Enable User Workload** - For app metrics

3. **Learn PromQL** - rate(), sum(), topk()

4. **ServiceMonitor scrapes apps** - Define what to monitor

5. **PrometheusRule creates alerts** - Automated notifications"

[SLIDE - Next Module]

"Next: Logging - Where do all those container logs go?"

---

## 📝 Presenter Notes

- Console → Observe → Metrics is the demo star
- User workload monitoring takes 2-3 min to spin up
- `rate()` on counters, raw values on gauges
