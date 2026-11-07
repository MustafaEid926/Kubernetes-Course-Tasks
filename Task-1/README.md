# Task Session 1: Multi-container Pods (Init Container & Sidecar)

MLOps MENA Community - Kubernetes Course. An nginx "hello" page showing the current
time, running on a 3-node Minikube cluster (Calico). Each Pod has three containers
and the manifest defines **no volumes**, using `shareProcessNamespace: true` instead:

| Container | Type | Job |
|-----------|------|-----|
| `init-done` | Init container | Prints `done` and exits; other containers start only if it succeeds |
| `nginx` | Main container | Writes the initial `index.html` with the time and serves it on port 80 |
| `time-updater` | Sidecar | Rewrites `index.html` every 5 seconds through `/proc/<nginx-pid>/root` |

## Files

- `namespace.yaml` - the `webapp` namespace
- `deployment.yaml` - `web-deployment`, 3 replicas, label `app: web`, the three containers

## How it was run

Reference commands:

```bash
minikube start --nodes 3 --cni=calico --cpus 2 --memory 2200
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w          # wait for 3 Pods, each 2/2 Running
kubectl port-forward -n webapp deploy/web-deployment 8080:80   # second terminal
```

How it actually went on this machine: the control plane came up with the command
above, but adding the two worker nodes through minikube timed out in minikube's
node-creation step on this Windows host. The workers were instead started with
`minikube node start` and joined manually with `kubeadm join` (a join token was
created on the control plane, and the kubelet's binary directory was added to
PATH for the join so kubeadm's preflight checks could pass). Final state was the
required one: 3 nodes `Ready`, all on Kubernetes v1.37.0.

## Verification results

| # | Check | Command | Result |
|---|-------|---------|--------|
| 1 | 3 nodes Ready | `kubectl get nodes` | all 3 `Ready`, v1.37.0 - screenshots/cluster-final-state.jpg |
| 2 | Deployment ready | `kubectl get deploy web-deployment -n webapp` | READY `3/3`, UP-TO-DATE `3`, AVAILABLE `3` - screenshots/cluster-final-state.jpg |
| 3 | 3 Pods 2/2 Running | `kubectl get pods -n webapp` | 3 Pods `2/2 Running`, 0 restarts - screenshots/cluster-final-state.jpg (rollout: screenshots/03-pods.png) |
| 4 | Init printed done | `kubectl logs <pod> -n webapp -c init-done` | prints `done` (output below) |
| 5 | Sidecar sees nginx process | `kubectl exec <pod> -n webapp -c time-updater -- ps` | nginx master + workers visible from the sidecar (output below) |
| 6 | Time updates every 5s | page at `localhost:8080`, refreshed after a few seconds | the timestamp changed between refreshes (04:12:13 -> 04:12:28): screenshots/06-page-time-1.png and screenshots/06-page-time-2.png |

Init container logs (check 4):

```
PS> kubectl logs web-deployment-5dc777c96f-8hmdb -c init-done -n webapp
done
```

Sidecar process view (check 5) - the sidecar sees the nginx master and worker
processes because the Pod shares one process namespace:

```
PS> kubectl exec web-deployment-5dc777c96f-8hmdb -c time-updater -n webapp -- ps
PID   USER     TIME  COMMAND
    1 65535     0:00 /pause
   13 root      0:00 /bin/sh -c echo "<h1>hello in MLOPS MENA ..." && nginx -g 'daemon off;'
   20 root      0:00 nginx: master process nginx -g daemon off;
   21 101       0:00 nginx: worker process
   ...
   37 root      0:00 sh -c while true; do PID=$(pgrep -o nginx) ... sleep 5; done
```

## Init failure experiment

Changed only the init container command to `["sh", "-c", "exit 1"]` and re-applied
(without deleting the Deployment).

- New Pod status: Pending -> `Init:0/1` -> `Init:Error` -> `Init:CrashLoopBackOff`
  SCREENSHOT: screenshots/07-crashloop.png
- The page stayed reachable during the whole failed rollout, with the timestamp
  still updating: screenshots/08-page-during-failure.png (04:18:12, mid-failure)
- `kubectl describe pod` on the broken Pod showed the init container
  `State: Terminated, Reason: Error, Exit Code: 1`, restarting with back-off,
  while `nginx` and `time-updater` stayed `Waiting` / `PodInitializing`.

Observations (my own notes from the run):

- The new Pod got stuck in `Init:Error` / `Init:CrashLoopBackOff`: the init
  container exited with `Exit Code: 1`, and Kubernetes kept restarting it with
  back-off.
- `nginx` and `time-updater` never started in the new Pod - `kubectl describe`
  showed them `Waiting` with reason `PodInitializing`. Containers start only
  after every init container completes successfully.
- The page was still reachable during the whole failure. The Deployment's rolling
  update kept the 3 old healthy Pods `2/2 Running` (the new ReplicaSet could not
  become ready, so the old one was never scaled down), so the application kept
  serving with no downtime.

## Recovery

Restored the working init command (`["sh", "-c", "echo done"]`) and re-applied.
The Deployment rolled back to the original ReplicaSet: its 3 Pods (which were
never replaced) stayed `2/2 Running` with 0 restarts, the broken Pod was
terminated, and `kubectl rollout status` reported
`deployment "web-deployment" successfully rolled out`.
Final state screenshot: screenshots/cluster-final-state.jpg

## Cluster

Kept for the next tasks; only the namespace would be removed with
`kubectl delete namespace webapp`.
