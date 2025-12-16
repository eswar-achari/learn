Below is a **practical, production-ready guide** focused on **Red Hat OpenShift**, **Java workloads**, and **Jenkins-driven deployments**.
I’ll separate **what Jenkins controls**, **what OpenShift controls**, and **what Java needs**, because CPU issues usually come from a mismatch across these layers.

---

## 1️⃣ First: Understand what “Improve CPU” really means

In OpenShift, you **cannot magically increase CPU** from Jenkins alone.
Jenkins can only **deploy with the right configuration**.

CPU improvement usually means:

* Giving **more CPU cores**
* Avoiding **CPU throttling**
* Making **JVM CPU-aware**
* Ensuring **proper pod autoscaling**
* Using **correct build & runtime images**

---

## 2️⃣ Set Correct CPU Requests & Limits (MOST IMPORTANT)

If CPU **limits** are too low, the JVM will be **throttled**, causing slow performance.

### ✅ Recommended practice for Java on OpenShift

* **Set CPU request**
* **Avoid strict CPU limits** (or set higher than request)

### Example Deployment YAML (controlled via Jenkins)

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1024Mi"
  limits:
    cpu: "2"
    memory: "2048Mi"
```

📌 **Rule of thumb**

* `requests` = guaranteed CPU
* `limits` = max CPU (throttling happens here)

❗ If limits = requests → JVM gets throttled under load

---

## 3️⃣ Make JVM Container-Aware (CRITICAL for Java)

Older JVMs ignore container CPU limits.

### Use these JVM options in Jenkins pipeline or Dockerfile

```bash
JAVA_OPTS="
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75
-XX:InitialRAMPercentage=50
-XX:ActiveProcessorCount=$(nproc)
"
```

### For Java 17 / 21 (recommended)

Container support is **ON by default**, but **ActiveProcessorCount** is still useful.

```bash
-XX:ActiveProcessorCount=2
```

📌 Match this value to your **CPU limit**

---

## 4️⃣ Configure Jenkins Pipeline Correctly

### Jenkinsfile – Deployment Stage Example

```groovy
stage('Deploy to OpenShift') {
  steps {
    sh """
    oc set resources deployment my-app \
      --requests=cpu=500m,memory=1Gi \
      --limits=cpu=2,memory=2Gi
    """
  }
}
```

Or apply YAML:

```groovy
sh "oc apply -f deployment.yaml"
```

✔ Jenkins **enforces CPU via manifests**, not dynamically

---

## 5️⃣ Use Horizontal Pod Autoscaler (HPA)

CPU improvement often means **scaling horizontally**, not vertically.

### Create HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

📌 OpenShift automatically adds pods when CPU crosses 70%

---

## 6️⃣ Ensure You Are Using the Right Java Base Image

### ❌ Avoid

* `openjdk:8-jre`
* Old non-container-aware images

### ✅ Use Red Hat UBI images

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21-runtime
```

✔ Optimized for OpenShift
✔ Proper cgroup support
✔ Security-patched

---

## 7️⃣ Enable CPU Performance Tuning (Advanced)

### Use `cpuManagerPolicy: static` (cluster level)

Best for **low-latency Java services**.

```yaml
spec:
  cpuManagerPolicy: static
```

Requires:

* Guaranteed QoS (requests == limits)
* Dedicated cores

📌 Used in **high-performance workloads**

---

## 8️⃣ Monitor CPU Throttling (Very Important)

### Check throttling

```bash
oc adm top pod
```

or inside pod:

```bash
cat /sys/fs/cgroup/cpu.stat
```

If you see:

```
nr_throttled > 0
```

→ CPU limits are too low.

---

## 9️⃣ Summary – What Actually Improves CPU

| Layer      | Action                                    |
| ---------- | ----------------------------------------- |
| Jenkins    | Apply correct resource YAML               |
| OpenShift  | Increase CPU requests / reduce throttling |
| JVM        | Enable container awareness                |
| Scaling    | Enable HPA                                |
| Image      | Use UBI OpenJDK                           |
| Monitoring | Detect CPU throttling                     |

---

## 10️⃣ Recommended Baseline for Java on OpenShift

```yaml
requests:
  cpu: 500m
limits:
  cpu: 2
JAVA_OPTS:
  -XX:MaxRAMPercentage=75
  -XX:ActiveProcessorCount=2
HPA:
  min: 2
  max: 10
```

---

### Next step?

If you want, I can:

* Review your **Jenkinsfile**
* Review your **Deployment YAML**
* Suggest **exact JVM flags for Java 21**
* Diagnose **CPU throttling logs**

Just tell me which one you want.
