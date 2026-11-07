# Task-1: Hello page with nginx on a 3-node minikube cluster

A "hello" page served by nginx on a 3-node minikube cluster. The page shows the current time and refreshes every 5 seconds:

```
hello in MLOPS MENA COMMUNITY : 2026-11-07 17:00:05
```

Each Pod has three containers and no volume:

| Container | Type | Job |
|---|---|---|
| init-done | Init container | Prints `done` and exits. The other containers start only if it succeeds. |
| nginx | Main container | Writes `index.html` with the time, then serves it on port 80 |
| time-updater | Sidecar | Rewrites `index.html` with the new time every 5 seconds |

The Pod uses `shareProcessNamespace: true`, so the sidecar sees the nginx process. The sidecar has the `SYS_PTRACE` capability and writes into the nginx container through `/proc/<nginx-pid>/root`.

---

## 0. Tools installed

I installed Docker, kubectl and minikube on Ubuntu (WSL2) and checked the versions.

```bash
docker --version
docker run --rm hello-world
kubectl version --client
minikube version
```

![Tools installed](screenshots/01-tools.png)

---

## 1. Create the cluster

A cluster with 1 control-plane and 2 workers, using the Calico network plugin.

```bash
minikube start --nodes 3 --cni=calico --cpus 2 --memory 2200
kubectl get nodes
```

All 3 nodes are `Ready`: `minikube`, `minikube-m02`, `minikube-m03`.

![Cluster](screenshots/02-cluster.png)

---

## 2. Create the namespace

```bash
kubectl create namespace webapp
kubectl get ns webapp
```

![Namespace](screenshots/03-namespace.png)

---

## 3. Write the Deployment

The file is `deployment.yaml` in this folder. Main points:

- `web-deployment` in namespace `webapp`, 3 replicas, label `app: web`
- `shareProcessNamespace: true` at Pod level
- init container `init-done` (busybox:1.36) runs `echo done`
- `nginx` (nginx:1.27) writes the page, then starts nginx in the foreground
- `time-updater` (busybox:1.36) with `SYS_PTRACE` rewrites the page every 5 seconds
- no volumes

---

## 4. Deploy and check

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w
```

All 3 Pods are `2/2 Running`. The READY column shows 2/2 because the init container has already finished, so only nginx and time-updater count.

![Pods running](screenshots/05-pods-running.png)

### Open the page

```bash
kubectl port-forward -n webapp deploy/web-deployment 8080:80
```

Opened `http://localhost:8080`. After refreshing 5 seconds later, the time changed.

![Browser, first load](screenshots/06-browser-1.png)
![Browser, after refresh](screenshots/07-browser-2.png)

### Verification

```bash
kubectl get deploy web-deployment -n webapp
kubectl get pods -n webapp -l app=web
kubectl get pods -n webapp -l app=web -o jsonpath='{.items[0].spec.initContainers[*].name} {.items[0].spec.containers[*].name}'
kubectl logs deploy/web-deployment -n webapp -c init-done
kubectl exec deploy/web-deployment -n webapp -c time-updater -- ps
```

Results:

- Deployment is `3/3` ready
- 3 Pods, each `2/2 Running`
- Containers: `init-done nginx time-updater`
- Init log prints `done`
- `ps` in the sidecar lists `nginx: master process`, so the sidecar sees nginx

![Verification](screenshots/08-verification.png)

### The time updates and there is no volume

```bash
kubectl exec deploy/web-deployment -n webapp -c nginx -- cat /usr/share/nginx/html/index.html
sleep 6
kubectl exec deploy/web-deployment -n webapp -c nginx -- cat /usr/share/nginx/html/index.html
```

The time is different between the two `cat` outputs.

![Time update and no volumes](screenshots/09-time.png)

---

## 5. Make the init container fail

I changed only the init container command in `deployment.yaml`:

```yaml
command: ["sh", "-c", "echo 'init failed'; exit 1"]
```

Then applied the file again without deleting the Deployment:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -w
```

The new Pod goes to `Init:Error` and then `Init:CrashLoopBackOff` with `0/2`. The 3 old Pods stay `2/2 Running`.

![Failing Pod](screenshots/10-failing-pod.png)

```bash
kubectl logs <new-pod> -n webapp -c init-done
kubectl describe pod <new-pod> -n webapp
```

The log shows `init failed`, and `describe` shows `init-done` with `Exit Code: 1`. The nginx and time-updater containers are waiting and never started.

![Logs and describe](screenshots/11-logs-describe.png)

```bash
kubectl rollout status deploy/web-deployment -n webapp
```

The rollout keeps waiting for the new Pod, and after the progress deadline it fails with:

```
error: deployment "web-deployment" exceeded its progress deadline
```

This is the expected result: the new Pod can never become ready, so the rollout cannot finish.

### Answers

**Which status does the new Pod show?**
`Init:Error`, then `Init:CrashLoopBackOff`, with READY `0/2`.

**Do nginx and time-updater start?**
No. Init containers must succeed before the main containers start. Because `init-done` exits with code 1, Kubernetes keeps retrying it with an increasing back-off and never starts nginx or time-updater.

**Is the page still reachable? Why?**
Yes. The Deployment uses a rolling update, which only removes an old Pod after a new one is ready. The new Pod never becomes ready, so the 3 old Pods keep running and serving the page.

---

## 6. Fix it

I changed the init container command back to `echo done` and applied the file again:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n webapp -l app=web
kubectl get deploy web-deployment -n webapp
```

All 3 Pods are `2/2 Running` and the Deployment shows `READY 3/3`, `UP-TO-DATE 3`.

![Fixed](screenshots/12-fixed.png)