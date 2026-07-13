# k8s readiness-probe termination regression repro

Minimal reproduction of a kubelet behavior change where an **exec readiness
probe stops executing once a pod begins termination** — it no longer runs, so
the pod's `Ready` condition is not correctly updated during the graceful-termination
window.

Suspected cause: [kubernetes/kubernetes#130487](https://github.com/kubernetes/kubernetes/pull/130487).

- **Good:** Kubernetes **v1.33** / **v1.34** — probe keeps running; `Ready` flips to `False` when the probe starts failing.
- **Bad:** Kubernetes **v1.35** — the probe is not run during termination, so `Ready` is not updated and stays `True`.

Controllers that route traffic on `Ready` status e.g., Cilium's `LocalRedirectPolicy`, keep sending traffic to a terminating pod.

## Requirements

- [kind](https://kind.sigs.k8s.io/), `kubectl`, and a running Docker daemon.
- On macOS this was developed with [Colima](https://github.com/abiosoft/colima); any Docker context works.

## Usage

Using a single cluster named `rpr` (every `kubectl` pins the context so nothing
runs against another cluster). To compare versions, tear down and re-create with
a different `kind/cluster-v*.yaml`:

```bash
# 1. create a cluster for the version you want to test
kind create cluster --config kind/cluster-v1.35.yaml --name rpr

# 2. deploy the workload (apply returns immediately), then wait for Ready
kubectl --context kind-rpr apply -f manifests/readiness-probe-demo.yaml
kubectl --context kind-rpr rollout status deploy/readiness-demo
```

Start the watch first so you catch the transition:

```bash
# 3. (terminal 1) watch the READY column -- leave this running
# `Error` status will be printed once a pod is gone and cannot be found, safe to ignore
kubectl --context kind-rpr get pods -l app=readiness-demo -w
```

In a second terminal session delete a pod:

```bash
# 4. (terminal 2) trigger termination -- pick one:

# direct termination (delete one pod; --wait=false returns immediately):
kubectl --context kind-rpr delete pod \
  $(kubectl --context kind-rpr get pods -l app=readiness-demo -o jsonpath='{.items[0].metadata.name}') \
  --wait=false
  
# or rolling update:
kubectl --context kind-rpr rollout restart deploy/readiness-demo
```

In terminal 1 the terminating pod's `READY` column:
- flips `1/1 -> 0/1` (good)
- stays `1/1` (bad)

then the pod disappears.
Stop the watch with Ctrl-C, then:

```bash
# 5. check the probe events
# Readiness probe failed is good, that is wanted behavior pre-1.35
# Context cancelled is bad, that is 1.35 behavior
kubectl --context kind-rpr get events --field-selector reason=Unhealthy,involvedObject.kind=Pod --sort-by=.lastTimestamp

# 6. tear down
kind delete cluster --name rpr
```

On v1.33/v1.34 the terminating pod's `Ready` flips to `False` a few seconds after
termination begins; on v1.35 the probe is not run, so `Ready` is not updated.

## Workload

A single-replica Deployment whose exec readiness probe is `test ! -f /tmp/shutdown`
(Ready while the marker is absent) and whose `preStop` does `echo "" > /tmp/shutdown`
then sleeps — so during termination the probe *would* fail if still executed.

## Expected output

Watching with `kubectl get pods -l app=readiness-demo -w` (the `READY` column is
`containers-ready/total`; `0/1` means `Ready=False`). Focus on the terminating
pod's row:

```
# v1.33 / v1.34 (good) -- READY flips to 0/1 a few seconds after termination begins
readiness-demo-aaa   1/1     Running   0     5s
readiness-demo-aaa   0/1     Running   0     8s
(pod deleted)

# v1.35 (bad) -- READY stays 1/1 until the pod is force-deleted at the grace limit
readiness-demo-aaa   1/1     Running   0     5s
readiness-demo-aaa   1/1     Running   0     60s
(pod deleted)
```

The probe events (`kubectl get events --field-selector reason=Unhealthy,involvedObject.kind=Pod`)
differ between versions:

```
# v1.33 / v1.34 (good)
50s         Warning   Unhealthy   pod/readiness-demo-544bc44664-gpdgz   Readiness probe failed:

# v1.35 (bad)
1s          Warning   Unhealthy   pod/readiness-demo-ddc787b9c-k9jsk   Readiness probe errored and resulted in unknown state: rpc error: code = Canceled desc = context canceled
```
