# Module 5: Cluster Updates & Versioning
## Lecture Transcript (~30 minutes)

---

### Opening (2 mins)

[SLIDE - Title]

"Cluster updates. Three words that strike fear into every admin's heart.

But OpenShift makes it... dare I say... painless?

Today we'll learn:
- How update channels work
- What happens during an update
- How to handle air-gapped environments"

[CHECK]

"Who here has done a Kubernetes upgrade? Was it fun? ...I can see by your faces it wasn't."

---

### Section 1: Update Channels (6 mins)

[SLIDE - Channels diagram]

"OpenShift uses CHANNELS to control updates. Think of them like lanes on a highway.

**stable-4.14** - The slow lane. Battle-tested updates only.

**fast-4.14** - The middle lane. Quicker access to fixes.

**candidate-4.14** - The fast lane. Preview of what's coming.

**eus-4.14** - Extended Update Support. For enterprises that update less frequently."

[PAUSE]

"Which should YOU use?"

[SLIDE - Recommendations]

"Production? **stable**. Always.

Want that critical bug fix sooner? **fast**, but test first.

Playing with new features? **candidate** in dev only.

Bank or Telco with strict change windows? **eus**."

[TERMINAL]

[DEMO]

```bash
# Check current channel
oc get clusterversion -o jsonpath='{.items[0].spec.channel}'

# See available updates
oc adm upgrade
```

---

### Section 2: The Update Graph (6 mins)

[SLIDE - Update graph diagram]

"Here's something unique about OpenShift: the Update Graph.

Not every version can update to every other version. There's a GRAPH of safe paths."

[Show diagram with arrows]

"See this? From 4.13.5, you can go to 4.13.6 or 4.13.10. But NOT directly to 4.14.5. You'd need to go 4.13.5 → 4.13.15 → 4.14.5."

[CHECK]

"Why does Red Hat do this?"

[PAUSE]

"Safety. Some upgrade paths have known issues. Red Hat blocks them to protect you."

[DEMO]

```bash
# View upgrade graph
oc adm upgrade
```

"This command shows you exactly what's available from YOUR current version. Trust it."

---

### Section 3: What Happens During an Update (7 mins)

[SLIDE - Update flow]

"Let's trace what happens when you click 'Update'..."

[Step through animation]

"1. **CVO reads new version**
   - Cluster Version Operator downloads the release image

2. **CVO updates Cluster Operators**
   - One by one, each operator gets updated
   - Each operator knows how to update itself

3. **Machine Config Operator updates nodes**
   - New RHCOS configuration prepared
   - Nodes reboot one at a time

4. **Rolling node updates**
   - Cordon node (no new pods)
   - Drain node (move existing pods)
   - Reboot with new OS
   - Uncordon (back in service)

5. **Done!**
   - All operators happy
   - All nodes updated"

[PAUSE]

"The whole thing is automated. You just monitor."

[SLIDE - Monitoring commands]

```bash
# Watch cluster version
oc get clusterversion -w

# Watch operators
oc get clusteroperators -w

# Watch nodes
oc get nodes -w
```

---

### Section 4: Pausing Node Updates (4 mins)

[SLIDE - Business hours problem]

"Here's a real scenario: It's Monday 10 AM. You started an update. Suddenly you realize - if nodes reboot now, it could impact users!"

[SLIDE - MachineConfigPool pause]

"Solution: Pause the MachineConfigPool!"

```bash
# Pause worker nodes
oc patch mcp worker --type merge --patch '{"spec":{"paused":true}}'
```

"This stops node reboots immediately. Control plane continues, but workers wait."

[SLIDE - Update window]

"The smart approach:
1. Start update Monday morning
2. Control plane updates during the day (no reboots)
3. Pause workers before they reboot
4. Saturday night: unpause workers
5. Worker nodes reboot during maintenance window"

```bash
# Resume on Saturday night
oc patch mcp worker --type merge --patch '{"spec":{"paused":false}}'
```

---

### Section 5: Air-Gapped Updates (5 mins)

[SLIDE - Disconnected diagram]

"Many enterprises have no internet in production. How do they update?"

[SLIDE - Mirror process]

"The answer: Mirroring.

1. Download update on a connected machine
2. Transfer to air-gapped network
3. Upload to internal registry
4. Point cluster at internal registry"

[SLIDE - oc-mirror tool]

"Red Hat provides `oc-mirror` for this:"

```bash
# Mirror to disk
oc mirror --config imageset-config.yaml file://archive

# Later, mirror to internal registry  
oc mirror --from file://archive docker://registry.internal:5000
```

"It downloads everything - release images, operators, even signatures."

[PAUSE]

"We won't do this on CRC, but know it exists. Many enterprise jobs require this skill."

---

### Wrap-up (2 mins)

[SLIDE - Key Takeaways]

"Module 5 done! Remember:

1. **Channels control update availability** - stable for prod

2. **Update graph ensures safe paths** - Trust `oc adm upgrade`

3. **CVO orchestrates everything** - Operators, then nodes

4. **Pause MCP for maintenance windows** - Control when nodes reboot

5. **oc-mirror for air-gapped** - Essential for regulated environments"

[SLIDE - Next Module]

"Next up: The Internal Registry. Where do all those container images actually live?"

---

## 📝 Presenter Notes

- Don't actually upgrade CRC - it can break things
- Students often ask "how long does upgrade take?" - Depends on cluster size, usually 30 mins to 2 hours
- For air-gapped: If students want to see oc-mirror, show documentation or demo video
- Common fear: "What if it breaks?" - That's what staging environments are for!
