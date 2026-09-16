# Runbook: checkout-api OOMKilled Incident Response

## Purpose

Use this runbook when `checkout-api` pods are restarting unexpectedly and Kubernetes reports the container was terminated with `OOMKilled`.

An `OOMKilled` event means the container exceeded its configured memory limit and Kubernetes terminated it.

For `checkout-api`, the previously known-good memory limit is:

```text
128Mi
```

---

# Diagnosis

## 1. Check pod status and restart counts

Start by checking the `checkout-api` pods:

```bash
kubectl get pods -n checkout-api
```

Example:

```text
NAME                            READY   STATUS    RESTARTS   AGE
checkout-api-7c8f7b6d4-x9k2p    1/1     Running   6          10m
```

A rapidly increasing `RESTARTS` count is a strong indication that the container is repeatedly crashing.

Watch the pods live if necessary:

```bash
kubectl get pods -n checkout-api -w
```

Press `Ctrl+C` to stop watching.

---

## 2. Confirm the container was OOMKilled

Get the pod name:

```bash
kubectl get pods -n checkout-api
```

Then describe the affected pod:

```bash
kubectl describe pod <pod-name> -n checkout-api
```

Look for:

```text
Last State:
  Terminated:
    Reason: OOMKilled
```

You can also inspect the previous container termination state directly:

```bash
kubectl get pod <pod-name> -n checkout-api \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'; echo
```

Expected output:

```text
OOMKilled
```

---

## 3. Check the configured memory request and limit

Inspect the Deployment:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'; echo
```

For easier-to-read output:

```bash
kubectl get deployment checkout-api -n checkout-api -o yaml
```

Look for:

```yaml
resources:
  requests:
    memory: "64Mi"
  limits:
    memory: "128Mi"
```

To display only the memory limit:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}'; echo
```

If this returns an unexpectedly low value such as:

```text
8Mi
```

the configured limit is likely the immediate cause of the incident.

---

## 4. Check recent Kubernetes events

Check for restarts, failed containers, and OOM-related events:

```bash
kubectl get events -n checkout-api \
  --sort-by='.lastTimestamp'
```

To view only the most recent events:

```bash
kubectl get events -n checkout-api \
  --sort-by='.lastTimestamp' | tail -20
```

Look for events involving:

```text
Killing
BackOff
Started
Created
```

---

## 5. Check current memory usage

If Metrics Server is available:

```bash
kubectl top pods -n checkout-api
```

Example:

```text
NAME                            CPU(cores)   MEMORY(bytes)
checkout-api-7c8f7b6d4-x9k2p    4m           47Mi
```

Compare observed usage against the configured limit.

For example, if the application normally uses approximately `40-60Mi` but has an `8Mi` limit, the resource configuration is clearly insufficient.

You can also check memory usage for all containers:

```bash
kubectl top pods -n checkout-api --containers
```

If `kubectl top` is unavailable, continue diagnosis using the pod termination state, configured limits, logs, and your monitoring system.

---

## 6. Check application logs

Inspect the current logs:

```bash
kubectl logs <pod-name> -n checkout-api
```

Because an OOMKilled container has already restarted, the previous container's logs may be more useful:

```bash
kubectl logs <pod-name> -n checkout-api --previous
```

Look for abnormal memory-related behavior, unusually large requests, application errors, or other evidence that could indicate a genuine application problem.

If the application was operating normally immediately before termination and the memory limit was recently lowered, configuration should be considered the primary suspect.

---

## 7. Determine whether this is a bad limit or a possible memory leak

A recent resource-limit change strongly suggests a configuration problem.

Check the Deployment rollout history:

```bash
kubectl rollout history deployment/checkout-api -n checkout-api
```

Inspect the current Deployment:

```bash
kubectl describe deployment checkout-api -n checkout-api
```

Compare:

* Current memory limit
* Previous known-good memory limit
* Normal observed application memory usage
* Recent configuration changes
* Whether memory usage grows continuously over time

In the documented incident, `checkout-api` was changed from:

```text
128Mi
```

to:

```text
8Mi
```

The `8Mi` limit was below normal application requirements and caused repeated OOMKills.

---

# Resolution

## 1. Restore the known-good memory limit

If the current memory limit is incorrectly set below the application's normal requirements, restore it to the known-good value.

For this incident:

```text
128Mi
```

If an emergency change directly in Kubernetes is appropriate:

```bash
sed -i 's/"8Mi"/"128Mi"/g' checkout-api-deployment.yaml
kubectl apply -f checkout-api-deployment.yaml
```

If CPU limits or memory requests are already defined and managed through infrastructure code, avoid unintentionally changing them.

The preferred permanent fix is to update the application's source configuration, such as Terraform or Kubernetes manifests, and then deploy it through the normal deployment process.

---

## 2. Monitor the rollout

Watch the Deployment:

```bash
kubectl rollout status deployment/checkout-api -n checkout-api
```

Check the new pods:

```bash
kubectl get pods -n checkout-api
```

You should see the new pods reach:

```text
READY   STATUS    RESTARTS
1/1     Running   0
```

---

## 3. Verify the corrected memory limit

Confirm the Deployment now has the expected value:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}'; echo
```

Expected:

```text
128Mi
```

---

## 4. Verify memory usage

If Metrics Server is available:

```bash
kubectl top pods -n checkout-api
```

Confirm that normal memory consumption remains comfortably below the new limit.

Example:

```text
NAME                            CPU(cores)   MEMORY(bytes)
checkout-api-xxxxxxxxxx-abcde    5m           52Mi
```

With a `128Mi` limit, `52Mi` of usage provides substantially more headroom than the failed `8Mi` configuration.

---

## 5. Verify restart counts are stable

Check pod status:

```bash
kubectl get pods -n checkout-api
```

Wait briefly and run the command again:

```bash
kubectl get pods -n checkout-api
```

The `RESTARTS` count should no longer increase.

---

## 6. Verify application health

Check the Kubernetes Service:

```bash
kubectl get service checkout-api -n checkout-api
```

Check the Deployment:

```bash
kubectl get deployment checkout-api -n checkout-api
```

Verify available replicas:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.status.availableReplicas}'; echo
```

If the application exposes its `/health` endpoint through the pod, test it directly:

```bash
kubectl port-forward deployment/checkout-api -n checkout-api 8000:8000
```

Then from another terminal:

```bash
curl http://localhost:8000/health
```

A successful response should confirm that the application is functioning normally.

---

# Rollback Option

If the incident immediately followed a Kubernetes Deployment change and the previous revision is known to be healthy, inspect rollout history:

```bash
kubectl rollout history deployment/checkout-api -n checkout-api
```

Then roll back:

```bash
kubectl rollout undo deployment/checkout-api -n checkout-api
```

Monitor the rollback:

```bash
kubectl rollout status deployment/checkout-api -n checkout-api
```

Verify the restored memory configuration afterward:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}'; echo
```

---

# Resolution Criteria

The incident can be considered resolved when all of the following are true:

* `checkout-api` pods are `Running` and `Ready`.
* Restart counts are no longer increasing.
* `Last State` no longer shows new `OOMKilled` events.
* The Deployment has the expected memory limit.
* Memory usage remains below the configured limit.
* The `/health` endpoint responds successfully.
* Requests are completing normally.
* The permanent resource configuration has been corrected in the source of truth.

---

# Follow-Up

After service is restored:

1. Ensure the `128Mi` limit is stored in the actual deployment source, such as Terraform or Kubernetes manifests, rather than relying only on an emergency `kubectl` change.
2. Review monitoring data to determine normal and peak memory usage.
3. Add validation or review requirements for unusually low resource limits.
4. Consider alerting on repeated pod restarts and `OOMKilled` events.
5. Document the incident and any changes made during recovery.

A direct `kubectl` fix restores service quickly, but the source-controlled configuration must also be corrected or the bad limit may return during the next deployment.
