# Day 44 — Kubernetes Configuration, Secrets, Resources & Namespace Guardrails

## Overview

Day 44 focused on making Kubernetes applications more configurable, secure, predictable, and resource-controlled.

The main areas covered were:

1. ConfigMaps
2. Secrets
3. Environment variables vs mounted files
4. ConfigMap update behavior
5. `subPath` behavior
6. Rollout restarts
7. Checksum-based automatic rollouts
8. Configuration failure drills
9. Environment variable precedence
10. Resource requests
11. Resource limits
12. CPU throttling
13. Memory pressure and `OOMKilled`
14. Liveness-probe-related memory failures
15. Application/runtime memory ceilings
16. Kubernetes QoS classes
17. Namespaces
18. LimitRange
19. ResourceQuota
20. ResourceQuota failure behavior
21. Kubernetes troubleshooting patterns

---

# 1. Day 44 Mental Model

The overall goal was to move from:

```text
Application
    |
    ├── hardcoded configuration
    ├── hardcoded secrets
    └── unpredictable resource usage
```

to:

```text
                    Kubernetes
                        |
        ┌───────────────┼────────────────┐
        |               |                |
    ConfigMap         Secret         Resources
        |               |                |
    normal config    sensitive       requests/limits
        |               |                |
        └───────────────┘                |
                |                        |
               Pod                    QoS class
                                        |
                                   Namespace
                                    /      \
                              LimitRange   ResourceQuota
```

---

# 2. ConfigMap

## What is a ConfigMap?

A ConfigMap stores **non-sensitive configuration**.

Examples:

```text
DB_HOST
REDIS_HOST
APP_PORT
LOG_LEVEL
FEATURE_X
GREETING
```

Our ConfigMap was:

```text
hello-api-config
```

Its values included:

```text
DB_HOST=postgres
REDIS_HOST=redis
APP_PORT=8080
LOG_LEVEL=info
FEATURE_X=true
GREETING=Hello from Kubernetes
```

The basic idea:

```text
ConfigMap
    |
    v
Application configuration
```

This allows the same application image to run with different configuration in different environments.

For example:

```text
Development
DB_HOST=dev-db

Production
DB_HOST=prod-db
```

without rebuilding the image.

---

# 3. Consuming ConfigMap Through Environment Variables

We used:

```yaml
envFrom:
  - configMapRef:
      name: hello-api-config
```

This means Kubernetes takes the keys from the ConfigMap and makes them environment variables inside the container.

Conceptually:

```text
ConfigMap
    |
    | envFrom
    v
Container environment
```

For example:

```text
GREETING=Hello from Kubernetes
```

could be accessed inside the container with:

```bash
echo "$GREETING"
```

---

# 4. Secret

Secrets are intended for sensitive configuration.

Our Secret was:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hello-api-secret
type: Opaque
stringData:
  password: "super-secret-password"
```

The application consumed the Secret using:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: hello-api-secret
        key: password
```

Conceptually:

```text
Secret
   |
   | secretKeyRef
   v
DB_PASSWORD
   |
   v
Container
```

We also mounted the Secret as a file:

```text
/etc/secrets/
```

So the application could access:

```text
/etc/secrets/password
```

---

# 5. ConfigMap vs Secret

| Object    | Purpose                            |
| --------- | ---------------------------------- |
| ConfigMap | Normal/non-sensitive configuration |
| Secret    | Sensitive configuration            |

Examples:

```text
ConfigMap:
DB_HOST
LOG_LEVEL
APP_PORT
FEATURE_X
```

```text
Secret:
DB_PASSWORD
API_TOKEN
credentials
private keys
```

---

# 6. Base64 Is NOT Encryption

This was an important security lesson.

Kubernetes Secret data may be represented using Base64.

For example:

```text
c3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

can be decoded back to:

```text
super-secret-password
```

Therefore:

> Base64 is encoding, not encryption.

Think:

```text
Base64
    |
    └── changes representation


Encryption
    |
    └── protects the data cryptographically
```

Proper Secret security involves controls such as:

* RBAC
* encryption at rest
* TLS communication
* restricting access to Secrets
* minimizing which workloads/users can read them

So:

```text
Secret != automatically secure everywhere
```

Secrets still require proper cluster security.

---

# 7. ConfigMap as a Volume

Instead of using environment variables, a ConfigMap can also be mounted as files.

We mounted the ConfigMap as:

```text
/etc/config/
```

containing:

```text
/etc/config/config.yaml
```

Conceptually:

```text
ConfigMap
    |
    v
Volume
    |
    v
/etc/config/config.yaml
```

This is useful when applications expect configuration files instead of environment variables.

---

# 8. Secret as a Volume

The Secret was also mounted as:

```text
/etc/secrets/
```

Conceptually:

```text
Secret
   |
   v
Secret volume
   |
   v
/etc/secrets/password
```

So Kubernetes configuration can be consumed in multiple ways.

---

# 9. ConfigMap Update Semantics

This was one of the most important experiments of Day 44.

We tested three methods:

```text
1. envFrom
2. normal ConfigMap volume
3. ConfigMap volume using subPath
```

They behave differently.

---

# 10. `envFrom` Update Behavior

We used:

```yaml
envFrom:
  - configMapRef:
      name: hello-api-config
```

The environment variable is populated when the container starts.

If the ConfigMap changes later:

```text
ConfigMap changes
      |
      v
Running container
      |
      X
Existing environment variable does not automatically change
```

The container needs to be recreated/restarted to receive the new environment value.

Therefore:

> ConfigMap environment variables behave like a startup snapshot.

---

# 11. Normal ConfigMap Volume Update

For a normal ConfigMap volume:

```text
ConfigMap
    |
    v
/etc/config/config.yaml
```

when the ConfigMap changes, Kubernetes can update the mounted file.

The update is not necessarily instantaneous, but the mounted file can eventually reflect the new ConfigMap value.

Therefore:

```text
Normal ConfigMap volume
        |
        v
ConfigMap changes
        |
        v
Mounted file eventually updates
```

---

# 12. `subPath` Behavior

We also mounted the ConfigMap file using:

```yaml
subPath: config.yaml
```

For example:

```yaml
volumeMounts:
  - mountPath: /etc/config-subpath/config.yaml
    name: config-file
    subPath: config.yaml
```

Unlike a normal ConfigMap volume, a `subPath` mount does not receive ConfigMap updates while the Pod is running.

Therefore:

```text
Normal volume
    → updates eventually

subPath
    → does not update until Pod recreation
```

This distinction was experimentally verified.

---

# 13. ConfigMap Update Cheat Sheet

| Consumption method | ConfigMap changes while Pod runs      |
| ------------------ | ------------------------------------- |
| `envFrom`          | ❌ Existing environment doesn't update |
| Normal volume      | ✅ File eventually updates             |
| `subPath`          | ❌ Doesn't update until Pod recreation |

This is one of the most important Day 44 tables.

---

# 14. Rollout Restart

Because environment variables don't automatically update, we used:

```bash
kubectl rollout restart deployment hello-api
```

This causes the Deployment to replace its Pods.

Conceptually:

```text
ConfigMap changes
      |
      v
rollout restart
      |
      v
Old Pods replaced
      |
      v
New Pods start
      |
      v
New Pods read current ConfigMap
```

This is a simple manual solution.

---

# 15. Checksum-Based Rollout

A better automated pattern is to make the ConfigMap content part of the Pod template's identity.

We generated a checksum:

```bash
kubectl get configmap hello-api-config -o yaml | sha256sum
```

We got:

```text
1304b0713e72d092636e5886557dd5d43feda7749d5c0d454cd95cb2875236d5
```

The checksum was then placed inside:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: "..."
```

We applied it with:

```bash
kubectl patch deployment hello-api \
  --type='strategic' \
  -p '{"spec":{"template":{"metadata":{"annotations":{"checksum/config":"1304b0713e72d092636e5886557dd5d43feda7749d5c0d454cd95cb2875236d5"}}}}}'
```

---

# 16. Why the Checksum Causes a Rollout

Kubernetes Deployments create a new ReplicaSet when the Pod template changes.

Therefore:

```text
ConfigMap changes
       |
       v
Checksum changes
       |
       v
Pod template annotation changes
       |
       v
Deployment sees template change
       |
       v
New ReplicaSet
       |
       v
New Pods
```

We actually observed:

```text
2 out of 3 new replicas have been updated
```

followed by:

```text
1 old replica is pending termination
```

and finally:

```text
deployment "hello-api" successfully rolled out
```

So the checksum rollout mechanism was successfully demonstrated.

---

# 17. Important Mistake We Discovered

Initially we ran:

```bash
kubectl annotate deployment hello-api checksum/config=...
```

That placed the annotation on the Deployment's own metadata:

```yaml
metadata:
  annotations:
```

It did NOT modify:

```yaml
spec:
  template:
    metadata:
      annotations:
```

Therefore it did not change the Pod template.

The correct location is:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: "..."
```

### Important rule

> Changing Deployment metadata is not the same thing as changing the Pod template.

```text
Deployment metadata
    → doesn't necessarily create new Pods

Pod template
    → changes create a new ReplicaSet
```

---

# 18. Failure Drill — Missing Secret Key

We intentionally referenced a key that did not exist inside a Secret.

The Pod entered:

```text
CreateContainerConfigError
```

The important troubleshooting command was:

```bash
kubectl describe pod <pod-name>
```

The Events showed that Kubernetes couldn't find the requested key.

Lesson:

> When a Pod has a configuration problem, `kubectl describe pod` is often more useful than just `kubectl get pods`.

---

# 19. Failure Drill — Missing Secret

We then referenced a Secret that didn't exist.

Again the Pod couldn't start and we observed:

```text
CreateContainerConfigError
```

The Events explained:

```text
secret "..." not found
```

The troubleshooting pattern was:

```bash
kubectl get pod <pod>
kubectl describe pod <pod>
```

Look at:

```text
Events
State
Last State
Reason
Exit Code
Restart Count
```

---

# 20. `optional: true`

We then tested:

```yaml
optional: true
```

With an optional configuration source:

```text
Secret missing
     |
     v
optional: true
     |
     v
Pod can still start
```

The missing environment variable simply wasn't provided.

Therefore:

```text
required configuration
    → missing = failure

optional configuration
    → missing = Pod can continue
```

---

# 21. Environment Variable Precedence

We tested the same variable from both:

```yaml
envFrom:
```

and:

```yaml
env:
```

For example:

```text
ConfigMap:
GREETING=Hello from Kubernetes
```

and:

```yaml
env:
  - name: GREETING
    value: "OVERRIDDEN-BY-ENV"
```

The container received:

```text
GREETING=OVERRIDDEN-BY-ENV
```

Therefore:

> An explicit `env` entry overrides the value coming from `envFrom`.

Mental model:

```text
envFrom
   |
   v
default source


env
   |
   v
explicit definition
   |
   v
wins
```

---

# 22. Resource Requests

Next we moved to CPU and memory management.

A resource request means:

> "Kubernetes, I need at least this much resource for scheduling."

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

The scheduler uses requests when deciding where the Pod can run.

Think:

```text
Request = scheduling reservation
```

---

# 23. Resource Limits

A limit means:

> "This container should not be allowed to exceed this amount."

Example:

```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

Think:

```text
Limit = runtime ceiling
```

---

# 24. Request vs Limit

The simplest way to remember:

```text
Request
    ↓
What Kubernetes reserves for scheduling


Limit
    ↓
Maximum runtime resource usage
```

---

# 25. CPU vs Memory

CPU and memory behave differently.

## CPU

CPU is compressible.

If the container reaches its CPU limit:

```text
CPU demand
    |
    v
CPU limit reached
    |
    v
Throttling
    |
    v
Container becomes slower
```

It is generally not killed merely because it wants more CPU.

---

## Memory

Memory is not compressible.

If a container exceeds its memory limit:

```text
Memory usage
     |
     v
Memory limit
     |
     v
Kernel kills process
     |
     v
OOMKilled
```

---

# 26. Pending Due to Insufficient Memory

We deliberately created a Pod requesting:

```text
memory: 100Gi
```

Our kind cluster didn't have a node capable of satisfying that request.

The Pod became:

```text
Pending
```

The `describe` output showed:

```text
Insufficient memory
```

The flow:

```text
Pod created
    |
    v
Scheduler
    |
    v
Looks for suitable node
    |
    v
No node has enough memory
    |
    v
Pod remains Pending
```

Important:

> The Pod exists, but the scheduler can't place it.

---

# 27. CPU Throttling Experiment

We created a CPU-intensive container:

```yaml
command:
  - sh
  - -c
  - while true; do :; done
```

with:

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 100m
```

The Pod stayed:

```text
Running
```

and had:

```text
0 restarts
```

But cgroup statistics showed increasing:

```text
nr_throttled
throttled_usec
```

This proved:

> CPU limits throttle CPU usage rather than normally killing the container.

---

# 28. Direct OOMKill Experiment

We created a process that continually allocated memory.

The container had a memory limit of:

```text
128Mi
```

Eventually:

```text
Memory usage
     |
     v
128Mi limit
     |
     v
Kernel kills process
     |
     v
OOMKilled
```

We observed:

```text
Reason: OOMKilled
Exit Code: 137
```

and the container restarted.

Important troubleshooting signature:

```text
OOMKilled
+
exit 137
```

is a strong indication of a memory kill.

---

# 29. `kubectl logs --previous`

When a container restarts after crashing, the previous container's logs can be useful.

The command to remember is:

```bash
kubectl logs <pod> --previous
```

This should become a troubleshooting reflex for restarting/crashing containers.

---

# 30. Liveness Probe Disguised as OOM

We created a container that:

* served a health endpoint
* continually leaked memory
* had a liveness probe
* eventually became unhealthy

We observed probe failures.

At first glance, it could look like:

```text
Liveness probe failed
```

was the root cause.

But inspecting the container's actual termination state showed:

```text
OOMKilled
Exit Code: 137
```

The correct reasoning was:

```text
Memory leak
    |
    v
Application becomes unhealthy
    |
    v
Liveness probe fails
    |
    v
Container restart
```

while the underlying resource failure could still be:

```text
OOMKilled
```

Lesson:

> Don't automatically assume the first error you see is the root cause.

Always inspect:

```bash
kubectl describe pod <pod>
```

and examine:

```text
Last State
Reason
Exit Code
Restart Count
Events
```

---

# 31. Runtime Memory Ceiling

We also tested an application-level memory ceiling.

The important idea:

```text
Container memory limit = 256Mi
Application/JVM heap limit = 64Mi
```

The application can hit its own limit before Kubernetes reaches the container limit.

For example:

```text
Application
    |
    v
JVM heap reaches -Xmx
    |
    v
OutOfMemoryError
    |
    v
Application handles it
    |
    v
Process exits
```

The container does not necessarily get:

```text
OOMKilled
```

In our experiment the application exited cleanly with:

```text
Exit Code: 0
```

Therefore:

> Not every memory failure is a Kubernetes `OOMKilled`.

---

# 32. Three Memory Failure Signatures

### Case 1 — Direct container OOM

```text
Container exceeds memory limit
        |
        v
OOMKilled
        |
        v
Exit 137
```

### Case 2 — Liveness-related memory failure

```text
Memory problem
      |
      v
Application becomes unhealthy
      |
      v
Liveness probe fails
      |
      v
Restart
```

Inspect the actual termination reason to determine whether OOM was involved.

### Case 3 — Application runtime ceiling

```text
Application reaches its own memory ceiling
        |
        v
Application throws memory error
        |
        v
Application handles it
        |
        v
Exit 0
```

No Kubernetes OOMKill is required.

---

# 33. QoS Classes

Kubernetes has three major QoS classes:

```text
BestEffort
Burstable
Guaranteed
```

---

# 34. BestEffort

We created:

```text
qos-besteffort
```

with no resource requests or limits.

Example:

```yaml
containers:
  - name: test
    image: busybox:1.36
```

QoS:

```text
BestEffort
```

Mental model:

> Nothing specified.

---

# 35. Burstable

We created:

```text
qos-burstable
```

with:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
```

QoS:

```text
Burstable
```

Mental model:

> Some resource configuration exists, but it doesn't meet the Guaranteed pattern.

---

# 36. Guaranteed

We created:

```text
qos-guaranteed
```

with:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 64Mi
```

QoS:

```text
Guaranteed
```

The Guaranteed pattern requires CPU and memory requests and limits for every container, with requests equal to limits.

Mental model:

```text
request == limit
for CPU and memory
for every container
        |
        v
Guaranteed
```

---

# 37. QoS Cheat Sheet

```text
No resources
    ↓
BestEffort


Some/unequal resource configuration
    ↓
Burstable


CPU + memory request == limit
for every container
    ↓
Guaranteed
```

QoS becomes particularly important when Kubernetes has to deal with node resource pressure and eviction.

QoS is important, but it is not the only factor Kubernetes considers when determining eviction behavior.

---

# 38. Namespace

We created:

```text
day44
```

using:

```bash
kubectl create namespace day44
```

We verified it with:

```bash
kubectl get namespaces
```

and:

```bash
kubectl get pods -n day44
```

Initially:

```text
No resources found in day44 namespace.
```

A Namespace is a logical boundary for Kubernetes resources.

Think:

```text
Cluster
│
├── default
├── monitoring
├── production
└── day44
```

---

# 39. LimitRange

We created:

```text
day44-defaults
```

with:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: day44-defaults
  namespace: day44
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 50m
        memory: 64Mi
      default:
        cpu: 100m
        memory: 128Mi
```

The meaning was:

```text
Every container without resource settings gets:

Request:
CPU = 50m
Memory = 64Mi

Limit:
CPU = 100m
Memory = 128Mi
```

---

# 40. LimitRange Experiment

We created a Pod without resources:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limitrange-test
  namespace: day44
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

We did NOT specify:

```yaml
resources:
```

But after creation:

```bash
kubectl get pod limitrange-test -n day44 \
  -o jsonpath='{.spec.containers[0].resources}'
```

returned:

```json
{
  "limits": {
    "cpu": "100m",
    "memory": "128Mi"
  },
  "requests": {
    "cpu": "50m",
    "memory": "64Mi"
  }
}
```

Therefore:

> LimitRange automatically injected the defaults.

---

# 41. LimitRange Works Per Container

We created:

```text
limitrange-sidecar
```

with:

```text
app container
sidecar container
```

Neither specified resources.

Kubernetes applied the defaults to both.

Result:

```text
app:
request = 50m CPU + 64Mi
limit   = 100m CPU + 128Mi

sidecar:
request = 50m CPU + 64Mi
limit   = 100m CPU + 128Mi
```

Total Pod request:

```text
CPU:
50m + 50m = 100m

Memory:
64Mi + 64Mi = 128Mi
```

Total Pod limits:

```text
CPU:
100m + 100m = 200m

Memory:
128Mi + 128Mi = 256Mi
```

Important:

> LimitRange applies defaults per container, not once per Pod.

---

# 42. ResourceQuota

We then created:

```text
day44-quota
```

with:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: day44-quota
  namespace: day44
spec:
  hard:
    requests.cpu: "200m"
    requests.memory: "256Mi"
    limits.cpu: "400m"
    limits.memory: "512Mi"
    pods: "4"
```

This means the entire `day44` namespace cannot exceed those quota values.

---

# 43. LimitRange vs ResourceQuota

This distinction is extremely important.

### LimitRange

```text
Per container
```

Question:

> "What should each container get by default?"

Example:

```text
CPU request = 50m
Memory request = 64Mi
```

### ResourceQuota

```text
Whole namespace
```

Question:

> "How much can the entire namespace consume?"

Example:

```text
CPU requests = 200m maximum
Memory requests = 256Mi maximum
```

Mental model:

```text
LimitRange
    ↓
per-container defaults


ResourceQuota
    ↓
namespace-wide ceiling
```

---

# 44. ResourceQuota Accounting Experiment

Initially:

```text
limitrange-test
```

consumed:

```text
requests:
CPU = 50m
Memory = 64Mi

limits:
CPU = 100m
Memory = 128Mi
```

The sidecar Pod had two containers:

```text
requests:
CPU = 100m
Memory = 128Mi

limits:
CPU = 200m
Memory = 256Mi
```

Total:

```text
requests:
CPU = 150m
Memory = 192Mi

limits:
CPU = 300m
Memory = 384Mi
```

The ResourceQuota correctly showed:

```text
requests.cpu       150m / 200m
requests.memory    192Mi / 256Mi

limits.cpu         300m / 400m
limits.memory      384Mi / 512Mi

pods               2 / 4
```

---

# 45. Filling the Quota

We created:

```text
quota-test-1
```

Because of the LimitRange, Kubernetes automatically assigned:

```text
requests:
50m CPU
64Mi memory

limits:
100m CPU
128Mi memory
```

The namespace then reached:

```text
requests.cpu       200m / 200m
requests.memory    256Mi / 256Mi

limits.cpu         400m / 400m
limits.memory      512Mi / 512Mi
```

All resource quotas were full.

---

# 46. ResourceQuota Failure Drill

We then tried to create:

```text
quota-test-2
```

The API server rejected it with:

```text
Error from server (Forbidden)
```

and:

```text
exceeded quota: day44-quota
```

The requested resources were:

```text
limits.cpu=100m
limits.memory=128Mi
requests.cpu=50m
requests.memory=64Mi
```

but the namespace was already at its limits.

Therefore:

```text
Pod creation request
       |
       v
ResourceQuota admission check
       |
       v
Quota exceeded
       |
       v
Forbidden
       |
       v
Pod NOT CREATED
```

---

# 47. Pending vs ResourceQuota Rejection

This was another important distinction.

## Insufficient node resources

```text
Pod created
     |
     v
Scheduler
     |
     v
No suitable node
     |
     v
Pending
```

## ResourceQuota exceeded

```text
kubectl apply
     |
     v
API server
     |
     v
Quota check
     |
     v
Quota exceeded
     |
     v
Forbidden
     |
     v
Pod never created
```

Therefore:

> **Pending means the Pod exists but cannot be scheduled.**

> **Forbidden/exceeded quota means the Pod was rejected before creation.**

---

# 48. Three Important Kubernetes Resource Layers

Remember these three:

```text
LimitRange
    ↓
"What should each container get?"
```

```text
ResourceQuota
    ↓
"How much can this namespace consume?"
```

```text
Scheduler
    ↓
"Which node can run this Pod?"
```

So:

```text
LimitRange    → container
ResourceQuota → namespace
Scheduler     → node
```

---

# 49. Troubleshooting Commands Learned

## Inspect Pods

```bash
kubectl get pods
```

With namespace:

```bash
kubectl get pods -n day44
```

---

## Detailed Pod information

```bash
kubectl describe pod <pod-name>
```

Use this when:

* Pod is Pending
* Pod is crashing
* Pod is not starting
* Secret/ConfigMap problems occur
* probes fail
* scheduling fails

---

## Check container resources

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.spec.containers[0].resources}'
```

---

## Check QoS

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.qosClass}'
```

---

## Check Deployment

```bash
kubectl get deployment hello-api -o yaml
```

---

## Check Pod-template annotations

```bash
kubectl get deployment hello-api \
  -o jsonpath='{.spec.template.metadata.annotations}'
```

---

## Rollout status

```bash
kubectl rollout status deployment hello-api
```

---

## Rollout restart

```bash
kubectl rollout restart deployment hello-api
```

---

## Previous container logs

```bash
kubectl logs <pod-name> --previous
```

Useful when a container restarted.

---

## Inspect ReplicaSets

```bash
kubectl get rs
```

Useful for understanding Deployment rollouts.

---

## Inspect LimitRange

```bash
kubectl describe limitrange day44-defaults -n day44
```

---

## Inspect ResourceQuota

```bash
kubectl describe resourcequota day44-quota -n day44
```

---

# 50. Complete Day 44 Mental Model

The entire day can be remembered as:

```text
                 APPLICATION
                      |
          ┌───────────┴───────────┐
          |                       |
     Configuration              Resources
          |                       |
     ┌────┴────┐             ┌────┴────┐
     |         |             |         |
 ConfigMap  Secret        Request     Limit
     |         |             |         |
 normal    sensitive      scheduler   runtime
 config     config
     |
     ├── envFrom
     ├── volume
     └── subPath
             |
             v
       update semantics
             |
             v
      checksum annotation
             |
             v
        new ReplicaSet
             |
             v
          new Pods


              NAMESPACE
                  |
          ┌───────┴────────┐
          |                |
     LimitRange       ResourceQuota
          |                |
   per-container      namespace total
       defaults            ceiling
```

---

# 51. Day 44 One-Line Rules

```text
ConfigMap = non-secret configuration.
```

```text
Secret = sensitive configuration.
```

```text
Base64 != encryption.
```

```text
envFrom = configuration becomes environment variables.
```

```text
secretKeyRef = consume one specific Secret key as an environment variable.
```

```text
Normal ConfigMap volume = updates eventually.
```

```text
subPath ConfigMap mount = doesn't receive updates until Pod recreation.
```

```text
envFrom values don't update in a running container.
```

```text
rollout restart = recreate Pods so they read new environment configuration.
```

```text
checksum = ConfigMap change can trigger a Deployment rollout.
```

```text
request = scheduling reservation.
```

```text
limit = runtime ceiling.
```

```text
CPU limit = throttling.
```

```text
Memory limit = possible OOMKill.
```

```text
OOMKilled + exit 137 = investigate memory.
```

```text
BestEffort = no resources specified.
```

```text
Burstable = resource configuration exists but doesn't meet Guaranteed rules.
```

```text
Guaranteed = CPU and memory requests equal limits for every container.
```

```text
Namespace = logical boundary.
```

```text
LimitRange = per-container defaults/constraints.
```

```text
ResourceQuota = namespace-wide resource ceiling.
```

```text
Pending = Pod exists but cannot be scheduled.
```

```text
Forbidden quota = Pod was rejected before creation.
```

---

# 52. What We Actually Proved Hands-On

Day 44 wasn't just theory.

We deliberately tested:

* ConfigMap environment variables
* ConfigMap file mounts
* Secret environment variables
* Secret file mounts
* Base64 decoding
* ConfigMap changes
* environment snapshot behavior
* normal volume updates
* `subPath` non-update behavior
* rollout restart
* checksum-based rollout
* missing Secret key
* missing Secret
* optional Secret
* `env` vs `envFrom` precedence
* insufficient node memory
* CPU throttling
* direct OOMKill
* liveness-related OOM behavior
* application runtime memory ceiling
* BestEffort QoS
* Burstable QoS
* Guaranteed QoS
* Namespace creation
* LimitRange defaults
* LimitRange per-container behavior
* ResourceQuota accounting
* ResourceQuota rejection

---

# 53. Final Day 44 Takeaway

The biggest lesson from Day 44 is:

> **Kubernetes allows us to separate application configuration from the application image and put resource usage under explicit control.**

Instead of:

```text
Application
    |
    ├── hardcoded configuration
    ├── hardcoded passwords
    └── unlimited/uncontrolled resources
```

we now have:

```text
Application
    |
    ├── ConfigMap → normal configuration
    |
    ├── Secret → sensitive configuration
    |
    ├── Requests → scheduling requirements
    |
    ├── Limits → runtime boundaries
    |
    ├── QoS → resource-pressure classification
    |
    └── Namespace guardrails
          |
          ├── LimitRange
          └── ResourceQuota
```

And the most important troubleshooting habit from this day is:

```bash
kubectl describe pod <pod>
```

Don't stop at:

```bash
kubectl get pods
```

When something goes wrong, inspect:

```text
Status
Events
State
Last State
Reason
Exit Code
Restart Count
```

That is how you move from:

> "My Pod is broken."

to:

> "My Pod is broken because the container was OOMKilled with exit code 137."

That difference is the real Kubernetes troubleshooting skill developed in Day 44.
