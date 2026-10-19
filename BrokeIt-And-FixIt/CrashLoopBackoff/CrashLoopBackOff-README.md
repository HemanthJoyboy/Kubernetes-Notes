# CrashLoopBackOff: Causes, Examples, and Fixes (Hands-on Lab)

A practice guide. For each error type you will:

1. **Create** a broken YAML file
2. **Apply** it and watch it fail
3. **See** what the error looks like
4. **Diagnose** it with the right commands
5. **Fix** it and verify
6. **Clean up**

Everything runs on a local [kind](https://kind.sigs.k8s.io/) cluster.

## Table of Contents

- [What is CrashLoopBackOff?](#what-is-crashloopbackoff)
- [Setup](#setup)
- [The debugging toolkit](#the-debugging-toolkit)
- [Exit codes cheat sheet](#exit-codes-cheat-sheet)
- [Lab 1: Wrong command or entrypoint](#lab-1-wrong-command-or-entrypoint)
- [Lab 2: Container exits immediately](#lab-2-container-exits-immediately)
- [Lab 3: Missing environment variable](#lab-3-missing-environment-variable)
- [Lab 4: Bad configuration file](#lab-4-bad-configuration-file)
- [Lab 5: Application error or missing dependency](#lab-5-application-error-or-missing-dependency)
- [Lab 6: OOMKilled (out of memory)](#lab-6-oomkilled-out-of-memory)
- [Lab 7: Failing liveness probe](#lab-7-failing-liveness-probe)
- [Lab 8: Permission problems](#lab-8-permission-problems)
- [Look-alike errors (not crash loops)](#look-alike-errors-not-crash-loops)
- [Quick diagnosis table](#quick-diagnosis-table)
- [The debugging workflow](#the-debugging-workflow)
- [Clean up everything](#clean-up-everything)

---

## What is CrashLoopBackOff?

It means your container **starts, exits, gets restarted by Kubernetes, and this keeps repeating**.

Kubernetes waits longer between each restart (the "back-off"):

```
10s -> 20s -> 40s -> 80s -> 160s -> 300s (5 min max)
```

The timer resets after the container runs successfully for 10 minutes.

**CrashLoopBackOff is a symptom, not the root cause.** It says "the container keeps dying." Your job is to find out *why*.

Why does Kubernetes restart it at all? Because a Pod's default `restartPolicy` is `Always`. The kubelet restarts any container that exits, whether it exited with an error or finished normally.

---

## Setup

Make sure your cluster is running:

```bash
kind create cluster --name lab          # skip if you already have one
kubectl get nodes
```

Create a separate namespace for the labs so cleanup is one command:

```bash
kubectl create namespace crashlab
kubectl config set-context --current --namespace=crashlab
kubectl config view --minify | grep namespace     # should print: namespace: crashlab
```

> **Important rule for all labs:** most fields of a running Pod (command, env, probes, image args) are **immutable**. You cannot edit them with `kubectl apply`. To apply a fix to a bare Pod, **delete it and create it again**. (With a Deployment, editing the template triggers a rollout automatically.)

---

## The debugging toolkit

Use these every single time:

```bash
kubectl get pods                          # STATUS and RESTARTS
kubectl get pods -w                       # live watch (Ctrl+C to stop)
kubectl describe pod <pod>                # Events + "Last State" + Exit Code
kubectl logs <pod>                        # current container logs
kubectl logs <pod> --previous             # logs from the crashed run (most useful!)
kubectl get pod <pod> -o yaml             # full spec and status
kubectl get events --sort-by=.lastTimestamp
```

Pull out just the exit code and reason:

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'
```

In `kubectl describe pod`, look at these parts:

```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
...
Events:
  Warning  BackOff  Back-off restarting failed container
```

---

## Exit codes cheat sheet

| Exit code | Meaning | Typical cause |
|---|---|---|
| **0** | Exited successfully | Process finished, nothing long-running |
| **1** | Generic application error | Missing env var, bad config, app exception |
| **2** | Misuse of shell command / bad arguments | Wrong flags or args |
| **126** | Command found but not executable | Permission problem on binary/script |
| **127** | Command not found | Wrong command or entrypoint |
| **128** | Container could not start | Bad command path (from the runtime) |
| **137** | Killed by SIGKILL (128+9) | **OOMKilled**, or killed after failed liveness probe |
| **139** | Segmentation fault (128+11) | Bug in app or native library |
| **143** | SIGTERM (128+15) | Graceful termination |

---

## Lab 1: Wrong command or entrypoint

**Cause:** `command` points to a binary that doesn't exist, or a whole command line was put into one list element.

### 1. Create the file

```bash
cat > wrong-command.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: wrong-command
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      command: ["nginx -g 'daemon off;'"]   # WRONG: whole string = one executable name
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f wrong-command.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME            READY   STATUS             RESTARTS      AGE
wrong-command   0/1     CrashLoopBackOff   4 (30s ago)   2m
```

(Early on you may briefly see `RunContainerError` or `StartError` before it becomes `CrashLoopBackOff`.)

### 4. Diagnose

```bash
kubectl describe pod wrong-command
```

```
    Last State:     Terminated
      Reason:       StartError
      Message:      failed to create containerd task: ... exec: "nginx -g 'daemon off;'":
                    executable file not found in $PATH: unknown
      Exit Code:    128
```

```bash
kubectl logs wrong-command      # EMPTY. The process never started.
```

**Clue:** empty logs + `StartError` or exit code 127/128 = command problem.

### 5. Fix

Each argument must be a separate list item:

```diff
-      command: ["nginx -g 'daemon off;'"]
+      command: ["nginx", "-g", "daemon off;"]
```

Or better, **remove `command` entirely** because nginx's default already works.

```bash
kubectl delete pod wrong-command
# edit wrong-command.yml with the fix, then:
kubectl apply -f wrong-command.yml
```

### 6. Verify

```bash
kubectl get pods        # STATUS: Running, RESTARTS: 0
```

### Bonus: command vs args

| Kubernetes field | Docker equivalent | Overrides |
|---|---|---|
| `command` | `ENTRYPOINT` | The image's entrypoint |
| `args` | `CMD` | The image's default arguments |

Check what an image runs by default:

```bash
docker inspect nginx:1.27 --format '{{.Config.Entrypoint}} {{.Config.Cmd}}'
```

**Script variants:**

- `command: ["/app/start.sh"]` where the file is missing gives exit **127/128** (`no such file or directory`).
- File exists but isn't executable gives exit **126** (`permission denied`). Fix with `chmod +x start.sh` in the Dockerfile, or use `command: ["sh", "/app/start.sh"]`.

### 7. Clean up

```bash
kubectl delete pod wrong-command
```

---

## Lab 2: Container exits immediately

**Cause:** A container lives only as long as its main process (PID 1). If that process finishes (exit code **0**), the container stops. Because `restartPolicy` is `Always`, Kubernetes restarts it forever.

### 1. Create the file

```bash
cat > exit-immediately.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: exit-immediately
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["echo", "hello world"]    # prints and exits
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f exit-immediately.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME               READY   STATUS             RESTARTS     AGE
exit-immediately   0/1     CrashLoopBackOff   5 (1m ago)   4m
```

(You may also see `Completed` between restarts.)

### 4. Diagnose

```bash
kubectl describe pod exit-immediately
kubectl logs exit-immediately
```

```
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0

hello world
```

**Clue:** exit code **0** and reason **Completed**. The app didn't fail. It finished, and Kubernetes expected it to keep running.

### 5. Fix

Pick the one that matches your goal.

**A long-running service:** keep the process in the foreground.

**A debug/practice container:** give it something that never ends.

```diff
-      command: ["echo", "hello world"]
+      command: ["sleep", "3600"]
```

**A one-time task:** don't use `restartPolicy: Always`.

```yaml
spec:
  restartPolicy: Never      # or use a Job
```

```bash
kubectl delete pod exit-immediately
kubectl apply -f exit-immediately.yml
kubectl get pods
```

### Bonus: the daemonizing trap

```yaml
      image: nginx:1.27
      command: ["nginx"]       # nginx forks into the background, PID 1 exits with 0
```

Fix: run it in the foreground.

```yaml
      command: ["nginx", "-g", "daemon off;"]
```

(The image's default command already does this, so usually don't override it.)

### 6. Clean up

```bash
kubectl delete pod exit-immediately
```

---

## Lab 3: Missing environment variable

**Cause:** The app requires an environment variable (password, DB host, API key) and exits when it isn't set.

### 1. Create the file

```bash
cat > missing-env.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
spec:
  containers:
    - name: postgres
      image: postgres:16
      # POSTGRES_PASSWORD is missing!
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f missing-env.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME           READY   STATUS             RESTARTS      AGE
postgres-pod   0/1     CrashLoopBackOff   3 (20s ago)   1m
```

### 4. Diagnose

```bash
kubectl logs postgres-pod
```

```
Error: Database is uninitialized and superuser password is not specified.
       You must specify POSTGRES_PASSWORD to a non-empty value for the
       superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".
```

```bash
kubectl describe pod postgres-pod
```

```
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

**Clue:** this is the easy kind. **The logs tell you exactly what is missing.**

### 5. Fix

**Option 1: plain value (practice only):**

```diff
       image: postgres:16
-      # POSTGRES_PASSWORD is missing!
+      env:
+        - name: POSTGRES_PASSWORD
+          value: "mysecretpassword"
```

**Option 2: from a Secret (the correct way):**

```bash
kubectl create secret generic pg-secret --from-literal=password=mysecretpassword
```

```yaml
      env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: password
```

Recreate the Pod:

```bash
kubectl delete pod postgres-pod
kubectl apply -f missing-env.yml
```

### 6. Verify

```bash
kubectl get pods
kubectl logs postgres-pod | tail -5     # "database system is ready to accept connections"
```

### Related error (not a crash loop)

If you *reference* a Secret or ConfigMap that **doesn't exist** (or a wrong key), you get `CreateContainerConfigError`, not CrashLoopBackOff. See [Look-alike errors](#look-alike-errors-not-crash-loops).

### 7. Clean up

```bash
kubectl delete pod postgres-pod
kubectl delete secret pg-secret --ignore-not-found
```

---

## Lab 4: Bad configuration file

**Cause:** The app starts, reads its config file, finds it invalid, and exits.

### 1. Create the file

```bash
cat > bad-config.yml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  default.conf: |
    server {
      listn 80;
      location / {
        return 200 "hello\n";
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: bad-config
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: conf
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: conf
      configMap:
        name: nginx-conf
EOF
```

The typo is `listn` (should be `listen`).

### 2. Apply and watch

```bash
kubectl apply -f bad-config.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME         READY   STATUS             RESTARTS      AGE
bad-config   0/1     CrashLoopBackOff   3 (25s ago)   1m
```

### 4. Diagnose

```bash
kubectl logs bad-config
```

```
nginx: [emerg] unknown directive "listn" in /etc/nginx/conf.d/default.conf:2
```

**Clue:** the log names the exact file and line number.

You can also inspect the config the Pod received:

```bash
kubectl get configmap nginx-conf -o yaml
```

### 5. Fix

```bash
sed -i 's/listn/listen/' bad-config.yml
kubectl apply -f bad-config.yml        # updates the ConfigMap
kubectl delete pod bad-config          # recreate so the app re-reads the config
kubectl apply -f bad-config.yml
```

(With a Deployment, use `kubectl rollout restart deployment <name>` instead.)

### 6. Verify

```bash
kubectl get pods
kubectl port-forward pod/bad-config 8080:80 &
curl localhost:8080                    # hello
kill %1
```

### Other bad-config variations to recognize

| Symptom in the logs | Likely cause |
|---|---|
| `connection refused` to a DB | Wrong DB host or port in the config |
| `no such file or directory` | Wrong `mountPath` or `subPath`, file not mounted where the app expects |
| `key not found` / app ignores settings | ConfigMap key name doesn't match what the app expects |
| `yaml: line 4: mapping values are not allowed` | Invalid YAML/JSON syntax inside the config |

**Tip:** test config **before** deploying, for example run `nginx -t` in a temporary container.

### 7. Clean up

```bash
kubectl delete -f bad-config.yml
```

---

## Lab 5: Application error or missing dependency

**Cause:** The app itself crashes: an unhandled exception, a missing library, or a dependency (like a database) that isn't reachable.

### 5a. Missing library

#### 1. Create the file

```bash
cat > app-crash.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app-crash
spec:
  containers:
    - name: python-app
      image: python:3.12-slim
      command: ["python", "-c", "import requests; print(requests.get('http://example.com').status_code)"]
EOF
```

#### 2. Apply and watch

```bash
kubectl apply -f app-crash.yml
kubectl get pods -w
```

#### 3. What you will see

```
NAME        READY   STATUS             RESTARTS      AGE
app-crash   0/1     CrashLoopBackOff   3 (22s ago)   1m
```

#### 4. Diagnose

```bash
kubectl logs app-crash
```

```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'requests'
```

Exit code is **1**.

**Clue:** a stack trace in the logs means the app is crashing. Read the **last line** of the traceback first.

#### 5. Fix

**The real fix** is to put the dependency into your image:

```dockerfile
FROM python:3.12-slim
RUN pip install requests
```

**For this lab only**, install it at start-up so you don't need to build an image:

```diff
-      command: ["python", "-c", "import requests; print(requests.get('http://example.com').status_code)"]
+      command: ["sh", "-c", "pip install -q requests && python -c \"import requests; print(requests.get('http://example.com').status_code)\" && sleep 3600"]
```

```bash
kubectl delete pod app-crash
kubectl apply -f app-crash.yml
kubectl logs app-crash              # prints 200 (needs internet access from the node)
```

#### 6. Clean up

```bash
kubectl delete pod app-crash
```

### 5b. Dependency not ready (init container)

A common problem: your app starts before the database is ready. An init container can make the Pod **wait** instead of crash.

#### 1. Create the file

```bash
cat > wait-for-db.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z db-service 5432; do echo waiting for db; sleep 2; done"]
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
EOF
```

#### 2. Apply and watch (the dependency does not exist yet)

```bash
kubectl apply -f wait-for-db.yml
kubectl get pods
```

```
NAME            READY   STATUS     RESTARTS   AGE
app-with-init   0/1     Init:0/1   0          20s
```

```bash
kubectl logs app-with-init -c wait-for-db
# waiting for db
# waiting for db
```

The Pod waits in `Init:0/1`. That is not a crash, it is working as designed.

#### 3. Fix: create the missing dependency

```bash
kubectl create deployment db --image=nginx
kubectl expose deployment db --name=db-service --port=5432 --target-port=80
kubectl get pods -w          # init container completes, then app container starts -> Running
```

> If an init container itself keeps **crashing**, the status is `Init:CrashLoopBackOff`. Debug it the same way, but add `-c <init-container-name>` to the logs command.

#### 4. Clean up

```bash
kubectl delete pod app-with-init
kubectl delete svc db-service
kubectl delete deployment db
```

---

## Lab 6: OOMKilled (out of memory)

**Cause:** The container uses more memory than its `limits.memory`, so the kernel kills it.

### 1. Create the file

```bash
cat > oom.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]   # tries to use 250 MB
      resources:
        limits:
          memory: "100Mi"        # only 100 MB allowed
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f oom.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME      READY   STATUS      RESTARTS      AGE
oom-pod   0/1     OOMKilled   2 (15s ago)   40s
# a bit later:
oom-pod   0/1     CrashLoopBackOff   4 (20s ago)   2m
```

### 4. Diagnose

```bash
kubectl describe pod oom-pod
```

```
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
```

```bash
kubectl logs oom-pod --previous      # usually no error message at all
```

**Clue:** reason **OOMKilled** and exit code **137**. The logs are often empty because the kernel killed the process abruptly.

### 5. Fix

- Raise the memory limit to what the app really needs, **or**
- Fix the memory leak / tune the app (for example JVM heap with `-Xmx`)

```diff
       resources:
+        requests:
+          memory: "300Mi"
         limits:
-          memory: "100Mi"
+          memory: "512Mi"
```

```bash
kubectl delete pod oom-pod
kubectl apply -f oom.yml
kubectl get pods
```

Check real usage (needs metrics-server, which kind does not include by default):

```bash
kubectl top pod oom-pod
```

### 6. Clean up

```bash
kubectl delete pod oom-pod
```

---

## Lab 7: Failing liveness probe

**Cause:** The app is running fine, but the liveness probe checks the wrong path or port, so the **kubelet kills the container** on purpose, over and over.

### 1. Create the file

```bash
cat > bad-liveness.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-liveness
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      livenessProbe:
        httpGet:
          path: /healthz          # nginx has no /healthz, so it returns 404
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
        failureThreshold: 2
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f bad-liveness.yml
kubectl get pods -w
```

### 3. What you will see

The Pod looks healthy for a few seconds, then restarts again and again:

```
NAME           READY   STATUS             RESTARTS      AGE
bad-liveness   1/1     Running            2 (8s ago)    50s
bad-liveness   0/1     CrashLoopBackOff   5 (10s ago)   3m
```

The status alternates between `Running` and `CrashLoopBackOff`, and `RESTARTS` keeps climbing.

### 4. Diagnose

```bash
kubectl describe pod bad-liveness
```

```
Events:
  Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    Container nginx failed liveness probe, will be restarted
```

```bash
kubectl logs bad-liveness            # looks normal, nginx started fine
```

**Clue:** normal logs, but `Unhealthy` and `Killing` events and a growing restart count. The exit code is usually **137** (SIGKILL) or 0, because nginx handles SIGTERM gracefully.

### 5. Fix

Probe a path or port that really exists:

```diff
       livenessProbe:
         httpGet:
-          path: /healthz
+          path: /
           port: 80
-        initialDelaySeconds: 3
-        periodSeconds: 5
+        initialDelaySeconds: 10
+        periodSeconds: 10
```

For **slow-starting apps**, add a startup probe so liveness doesn't kill the app before it is ready:

```yaml
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 5
```

```bash
kubectl delete pod bad-liveness
kubectl apply -f bad-liveness.yml
kubectl get pods
```

### Probe reminders

| Probe | If it fails |
|---|---|
| **Startup** | Container is killed (other probes are off until it passes) |
| **Liveness** | Container is **restarted** |
| **Readiness** | Pod is removed from Service endpoints, **not** restarted |

### 6. Clean up

```bash
kubectl delete pod bad-liveness
```

---

## Lab 8: Permission problems

**Cause:** A `securityContext` runs the container as a non-root user (or the filesystem is read-only), but the app needs to write to a directory or bind a privileged port.

### 1. Create the file

```bash
cat > permission.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: permission-denied
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
  containers:
    - name: nginx
      image: nginx:1.27          # the standard nginx image expects to run as root
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f permission.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME                READY   STATUS             RESTARTS      AGE
permission-denied   0/1     CrashLoopBackOff   3 (18s ago)   1m
```

### 4. Diagnose

```bash
kubectl logs permission-denied
```

```
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

Depending on the image version, you may instead see:

```
nginx: [emerg] open() "/run/nginx.pid" failed (13: Permission denied)
```

**Clue:** search the logs for `Permission denied`, `EACCES`, or `read-only file system`.

### 5. Fix

**Option 1 (recommended): use an image built for non-root**

```diff
-      image: nginx:1.27
+      image: nginxinc/nginx-unprivileged:1.27    # listens on 8080, not 80
```

**Option 2: don't force a non-root user** (if your policy allows it)

```diff
-  securityContext:
-    runAsUser: 1000
-    runAsNonRoot: true
```

```bash
kubectl delete pod permission-denied
kubectl apply -f permission.yml
kubectl get pods
```

### Notes

- Ports below 1024 can require extra privileges for non-root users, so prefer ports like 8080.
- With `readOnlyRootFilesystem: true`, give the app a writable location through an `emptyDir` volume mounted where it writes (for example `/tmp`).
- If a mounted volume isn't writable by your user, `fsGroup` in the Pod `securityContext` can fix the ownership.

### 6. Clean up

```bash
kubectl delete pod permission-denied
```

---

## Look-alike errors (not crash loops)

These look similar but are different problems.

### `CreateContainerConfigError`: missing Secret/ConfigMap reference

```bash
cat > missing-secret.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: missing-secret
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: does-not-exist
              key: password
EOF
kubectl apply -f missing-secret.yml
kubectl get pods
```

```
NAME             READY   STATUS                       RESTARTS   AGE
missing-secret   0/1     CreateContainerConfigError   0          15s
```

```bash
kubectl describe pod missing-secret
# Error: secret "does-not-exist" not found
```

**Fix:** create the missing object. The Pod then starts by itself on the next retry.

```bash
kubectl create secret generic does-not-exist --from-literal=password=abc123
kubectl get pods -w
kubectl delete pod missing-secret
kubectl delete secret does-not-exist
```

### `ImagePullBackOff` / `ErrImagePull`: wrong image or no access

```bash
kubectl run bad-image --image=nginx:thisdoesnotexist
kubectl get pods
```

```
NAME        READY   STATUS             RESTARTS   AGE
bad-image   0/1     ImagePullBackOff   0          30s
```

```bash
kubectl describe pod bad-image     # Events show: manifest for nginx:thisdoesnotexist not found
```

**Fix:** correct the image name or tag (and add `imagePullSecrets` for private registries).

```bash
kubectl delete pod bad-image
```

### `Pending`: not scheduled

See the scheduler section in the main README. Check `kubectl describe pod` for a `FailedScheduling` event (not enough CPU/memory, node selector mismatch, taints).

---

## Quick diagnosis table

| What you see | Most likely cause | Fix |
|---|---|---|
| Logs empty, `StartError`, exit 127/128 | Wrong command or entrypoint | Fix `command`/`args`, check the path |
| Exit **0**, reason `Completed` | Process finished or daemonized | Run in foreground, or use a Job |
| Exit **1**, logs say "X is required" | Missing env var | Add `env` or a Secret |
| Exit **1**, logs show a config parse error | Bad config | Fix the ConfigMap, restart the Pod |
| Exit **1**, stack trace in logs | App bug or missing dependency | Fix code/image, add an init container |
| Exit **137**, `OOMKilled` | Memory limit too low | Increase `limits.memory` |
| Normal logs, events show `Liveness probe failed` | Bad probe | Fix probe path/port, add a startup probe |
| Logs show `Permission denied` | `securityContext` or filesystem | Use an unprivileged image, add `emptyDir` |
| `CreateContainerConfigError` | Missing ConfigMap/Secret reference | Create the object (not a crash loop) |
| `ImagePullBackOff` | Wrong image name or no access | Fix image and credentials (not a crash loop) |

---

## The debugging workflow

```
1. kubectl get pods                  -> confirm CrashLoopBackOff, note RESTARTS
2. kubectl describe pod <pod>        -> read Last State, Exit Code, Events
3. kubectl logs <pod> --previous     -> read why the last run died
4. Match exit code + log message to the table above
5. Fix the YAML or image -> delete + apply the Pod -> kubectl get pods -w
```

For multi-container Pods, add `-c <container-name>` to `logs`.

---

## Clean up everything

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace crashlab
rm -f wrong-command.yml exit-immediately.yml missing-env.yml bad-config.yml \
      app-crash.yml wait-for-db.yml oom.yml bad-liveness.yml permission.yml missing-secret.yml
```

## Practice challenges

Once you finish the labs, test yourself:

1. Break a Pod in a new way (wrong port, wrong image tag, bad `args`) and predict the status before checking.
2. Wrap one of the broken Pods in a **Deployment** and watch how the rollout behaves when the new Pods crash (`kubectl rollout status`, `kubectl rollout undo`).
3. Have a friend change one line in a working YAML, then diagnose it using only `get`, `describe`, and `logs --previous`.
4. Write the exit code and the log clue for each error from memory, then compare with the cheat sheet.
