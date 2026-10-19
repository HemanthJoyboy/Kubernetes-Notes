# ImagePullBackOff: Causes, Examples, and Fixes (Hands-on Lab)

A practice guide. For each error type you will:

1. **Create** a broken YAML file
2. **Apply** it and watch it fail
3. **See** what the error looks like
4. **Diagnose** it with the right commands
5. **Fix** it and verify
6. **Clean up**

Everything runs on a local [kind](https://kind.sigs.k8s.io/) cluster.

## Table of Contents

- [What is ImagePullBackOff?](#what-is-imagepullbackoff)
- [ImagePullBackOff vs CrashLoopBackOff](#imagepullbackoff-vs-crashloopbackoff)
- [Setup](#setup)
- [The debugging toolkit](#the-debugging-toolkit)
- [Lab 1: Typo in the image name](#lab-1-typo-in-the-image-name)
- [Lab 2: Wrong image tag](#lab-2-wrong-image-tag)
- [Lab 3: Bad registry hostname or no network](#lab-3-bad-registry-hostname-or-no-network)
- [Lab 4: Private registry without credentials](#lab-4-private-registry-without-credentials)
- [Lab 5: Local image not loaded into kind](#lab-5-local-image-not-loaded-into-kind)
- [Lab 6: imagePullPolicy problems](#lab-6-imagepullpolicy-problems)
- [Causes you cannot easily reproduce](#causes-you-cannot-easily-reproduce)
- [ImagePullBackOff in a Deployment](#imagepullbackoff-in-a-deployment)
- [Look-alike errors](#look-alike-errors)
- [Quick diagnosis table](#quick-diagnosis-table)
- [The debugging workflow](#the-debugging-workflow)
- [Clean up everything](#clean-up-everything)
- [Practice challenges](#practice-challenges)

---

## What is ImagePullBackOff?

Before a container can start, the kubelet on the node must **pull the image** from a registry (Docker Hub, GHCR, ECR, a private registry, and so on). If the pull fails, two statuses appear one after the other:

| Status | Meaning |
|---|---|
| **ErrImagePull** | The pull attempt just failed. |
| **ImagePullBackOff** | The kubelet is **waiting before it retries** the pull (the "back-off"). |

You will see the status flip between the two. The wait grows after every failure:

```
10s -> 20s -> 40s -> 80s -> 160s -> 300s (5 min max)
```

**ImagePullBackOff is a symptom, not the root cause.** The root cause is in the pull error message, which you read with `kubectl describe pod`.

The pull error usually falls into one of these buckets:

| Bucket | Example cause |
|---|---|
| Image doesn't exist | Typo in the name, wrong tag |
| Registry unreachable | Wrong hostname, DNS, firewall, proxy |
| Not allowed to pull | Private image with no credentials, expired token |
| Pulled too often | Docker Hub rate limit |
| Not the right image for the node | Wrong CPU architecture |
| Local image not on the node | Built with Docker but never loaded into kind |

---

## ImagePullBackOff vs CrashLoopBackOff

Beginners mix these up. The key difference is **whether the container ever started**.

| | ImagePullBackOff | CrashLoopBackOff |
|---|---|---|
| Container started? | **No**, the image never arrived | **Yes**, it started and then died |
| `RESTARTS` column | `0` | Keeps increasing |
| `kubectl logs` | Fails: "waiting to start", nothing to show | Shows the app's output |
| Exit code | None | Present (1, 137, 0, ...) |
| Pod phase | `Pending` | `Running` (between crashes) |
| Where to look | `kubectl describe pod` -> **Events** | `kubectl logs --previous` + exit code |

---

## Setup

Make sure your cluster is running (replace `lab` with your cluster name if different):

```bash
kind create cluster --name lab          # skip if you already have one
kubectl get nodes
```

Create a separate namespace so cleanup is one command:

```bash
kubectl create namespace imagelab
kubectl config set-context --current --namespace=imagelab
kubectl config view --minify | grep namespace     # should print: namespace: imagelab
```

> **Good news compared to crash loops:** the `image` field is one of the few fields of a running Pod that you **can** change in place. So most fixes below use `kubectl set image` and don't need a delete. Everything else in a Pod spec is still immutable.

---

## The debugging toolkit

```bash
kubectl get pods                              # STATUS: ErrImagePull / ImagePullBackOff
kubectl get pods -w                           # live watch (Ctrl+C to stop)
kubectl describe pod <pod>                    # read the Events section (most important)
kubectl get events --sort-by=.lastTimestamp   # all recent events
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'   # the exact image string
```

Test the pull **outside** Kubernetes, and **inside the node**:

```bash
docker pull <image>                                        # works on your machine?
docker exec lab-control-plane crictl pull <image>          # works from the kind node?
docker exec lab-control-plane crictl images                # what images does the node have?
```

(`lab-control-plane` is the kind node container name. List yours with `docker ps`.)

What the Events section looks like when a pull fails:

```
Events:
  Normal   Scheduled  Successfully assigned imagelab/bad-image to lab-control-plane
  Normal   Pulling    Pulling image "nginx:thisdoesnotexist"
  Warning  Failed     Failed to pull image "nginx:thisdoesnotexist": ... not found
  Warning  Failed     Error: ErrImagePull
  Normal   BackOff    Back-off pulling image "nginx:thisdoesnotexist"
  Warning  Failed     Error: ImagePullBackOff
```

> The exact wording of the pull error depends on your container runtime and Kubernetes version. Look for the key phrases (`not found`, `pull access denied`, `no such host`, `unauthorized`, `toomanyrequests`).

---

## Lab 1: Typo in the image name

**Cause:** The repository name is misspelled, so no such image exists.

### 1. Create the file

```bash
cat > typo-image.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: typo-image
spec:
  containers:
    - name: nginx
      image: ngnix:1.27        # typo: should be "nginx"
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f typo-image.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME         READY   STATUS             RESTARTS   AGE
typo-image   0/1     ErrImagePull       0          8s
typo-image   0/1     ImagePullBackOff   0          20s
```

Note `READY 0/1` and `RESTARTS 0`. The container never started.

### 4. Diagnose

```bash
kubectl describe pod typo-image
```

```
  Warning  Failed   Failed to pull image "ngnix:1.27": ... pull access denied,
                    repository does not exist or may require authorization
  Warning  Failed   Error: ErrImagePull
  Normal   BackOff  Back-off pulling image "ngnix:1.27"
```

```bash
kubectl logs typo-image
# Error: container "nginx" in pod "typo-image" is waiting to start: trying and failing to pull image
```

**Clue:** `repository does not exist or may require authorization`. This message is the same for a typo and for a private repo you can't access, so also check the spelling.

### 5. Fix

```bash
kubectl set image pod/typo-image nginx=nginx:1.27
kubectl get pods -w
```

(Syntax: `kubectl set image pod/<pod> <container-name>=<new-image>`.) Or edit the YAML file and run `kubectl apply -f typo-image.yml`.

### 6. Verify

```bash
kubectl get pods             # STATUS: Running
```

### 7. Clean up

```bash
kubectl delete pod typo-image
```

---

## Lab 2: Wrong image tag

**Cause:** The repository exists, but that **tag** doesn't.

### 1. Create the file

```bash
cat > bad-tag.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-tag
spec:
  containers:
    - name: nginx
      image: nginx:thisdoesnotexist
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f bad-tag.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME      READY   STATUS             RESTARTS   AGE
bad-tag   0/1     ImagePullBackOff   0          25s
```

### 4. Diagnose

```bash
kubectl describe pod bad-tag
```

```
  Warning  Failed   Failed to pull image "nginx:thisdoesnotexist": ...
                    docker.io/library/nginx:thisdoesnotexist: not found
```

(Older runtimes say `manifest for nginx:thisdoesnotexist not found: manifest unknown`.)

**Clue:** the word **`not found`** / **`manifest unknown`** means the repository was found, but the tag was not. That distinguishes it from a typo in the name.

Check which tags really exist on Docker Hub: `https://hub.docker.com/_/nginx/tags`.

### 5. Fix

```bash
kubectl set image pod/bad-tag nginx=nginx:1.27
kubectl get pods
```

### Tip: avoid `latest`

Using no tag or `:latest` hides version problems and changes the pull policy (see [Lab 6](#lab-6-imagepullpolicy-problems)). Pin an exact version such as `nginx:1.27`.

### 6. Clean up

```bash
kubectl delete pod bad-tag
```

---

## Lab 3: Bad registry hostname or no network

**Cause:** The registry address is wrong, or the node can't reach it (DNS failure, firewall, proxy, no internet).

### 1. Create the file

```bash
cat > bad-registry.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-registry
spec:
  containers:
    - name: app
      image: registry.invalid/team/app:1.0     # ".invalid" never resolves
EOF
```

### 2. Apply and watch

```bash
kubectl apply -f bad-registry.yml
kubectl get pods -w
```

### 3. What you will see

```
NAME           READY   STATUS             RESTARTS   AGE
bad-registry   0/1     ImagePullBackOff   0          30s
```

### 4. Diagnose

```bash
kubectl describe pod bad-registry
```

```
  Warning  Failed   Failed to pull image "registry.invalid/team/app:1.0": ...
                    dial tcp: lookup registry.invalid: no such host
```

**Clue:** `no such host` (DNS failed) or `i/o timeout` / `connection refused` (network or firewall blocking). Other network-type errors:

| Message | Meaning |
|---|---|
| `no such host` | Hostname is wrong or DNS can't resolve it |
| `i/o timeout` / `context deadline exceeded` | Node can't reach the registry (firewall, proxy, no internet) |
| `x509: certificate signed by unknown authority` | Registry uses a self-signed or private CA the node doesn't trust |
| `http: server gave HTTP response to HTTPS client` | Registry is plain HTTP but the runtime expects HTTPS |

Confirm from the node:

```bash
docker exec lab-control-plane crictl pull registry.invalid/team/app:1.0
docker exec lab-control-plane sh -c "getent hosts registry.invalid || echo 'DNS lookup failed'"
```

### 5. Fix

- Correct the registry hostname (for example `ghcr.io/team/app:1.0`), **or**
- Fix DNS/proxy/firewall so the node can reach the registry, **or**
- For private CAs, install the CA certificate on the nodes.

```bash
kubectl set image pod/bad-registry app=nginx:1.27
```

### 6. Clean up

```bash
kubectl delete pod bad-registry
```

---

## Lab 4: Private registry without credentials

**Cause:** The image exists but is **private**, and the Pod has no credentials to pull it.

This lab needs a private repository. A free Docker Hub account allows one private repo.

> **Never commit real credentials to GitHub.** Use placeholders in files you push. Use an access token, not your password.

### 1. Prepare a private image (one time)

```bash
docker login
docker pull busybox:1.36
docker tag busybox:1.36 <dockerhub-user>/private-app:1.0
docker push <dockerhub-user>/private-app:1.0
```

On hub.docker.com, open the new repository -> **Settings** -> set visibility to **Private**.

### 2. Create the file

```bash
cat > private-image.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: private-image
spec:
  containers:
    - name: app
      image: <dockerhub-user>/private-app:1.0
      command: ["sleep", "3600"]
EOF
```

### 3. Apply and watch

```bash
kubectl apply -f private-image.yml
kubectl get pods -w
```

```
NAME            READY   STATUS             RESTARTS   AGE
private-image   0/1     ImagePullBackOff   0          30s
```

### 4. Diagnose

```bash
kubectl describe pod private-image
```

```
  Warning  Failed   Failed to pull image "<user>/private-app:1.0": ...
                    pull access denied, repository does not exist or may require authorization
```

(Some registries say `unauthorized: authentication required` or `401 Unauthorized` instead.)

**Clue:** the image name is spelled correctly, but the registry refuses you. That means credentials.

### 5. Fix

**Step A: create a registry secret**

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<dockerhub-user> \
  --docker-password=<access-token> \
  --docker-email=<your-email>
```

(For other registries, change `--docker-server`, for example `ghcr.io` or `<account>.dkr.ecr.<region>.amazonaws.com`.)

**Step B: reference it in the Pod**

```diff
 spec:
+  imagePullSecrets:
+    - name: regcred
   containers:
     - name: app
```

`imagePullSecrets` is a Pod-level field and **can't be added to a running Pod**, so recreate it:

```bash
kubectl delete pod private-image
kubectl apply -f private-image.yml
kubectl get pods -w
```

### 6. Verify

```bash
kubectl get pods                                    # Running
kubectl describe pod private-image | grep -A3 Events
```

### Make it automatic for every Pod in the namespace

Attach the secret to the default ServiceAccount so you don't repeat `imagePullSecrets` in each Pod:

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```

### Checks if it still fails

```bash
kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d   # contents correct?
```

- The secret must be in the **same namespace** as the Pod.
- The secret's registry address must match the registry in the image name.
- Expired or revoked tokens give the same `unauthorized` error.

### 7. Clean up

```bash
kubectl delete pod private-image
kubectl delete secret regcred
kubectl patch serviceaccount default -p '{"imagePullSecrets":[]}'
```

---

## Lab 5: Local image not loaded into kind

**Cause:** You built an image on your machine with `docker build`, but kind nodes are separate containers with their **own** image store. Your local Docker images are not visible to the cluster. This is one of the most common problems when practicing with kind.

### 1. Build a local image

```bash
mkdir -p local-image && cd local-image
cat > Dockerfile << 'EOF'
FROM busybox:1.36
CMD ["sleep", "3600"]
EOF
docker build -t myapp:1.0 .
cd ..
docker images | grep myapp          # exists on your machine
```

### 2. Create the file

```bash
cat > local-image.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: local-image
spec:
  containers:
    - name: app
      image: myapp:1.0
EOF
```

### 3. Apply and watch

```bash
kubectl apply -f local-image.yml
kubectl get pods -w
```

```
NAME          READY   STATUS             RESTARTS   AGE
local-image   0/1     ImagePullBackOff   0          25s
```

### 4. Diagnose

```bash
kubectl describe pod local-image
```

```
  Warning  Failed   Failed to pull image "myapp:1.0": ...
                    docker.io/library/myapp:1.0: pull access denied,
                    repository does not exist or may require authorization
```

Kubernetes tried to pull `myapp:1.0` from **Docker Hub**, where it doesn't exist. Confirm the node doesn't have it:

```bash
docker exec lab-control-plane crictl images | grep myapp      # prints nothing
```

**Clue:** the image works with `docker run` on your machine but fails in the cluster. It's a kind-specific problem.

### 5. Fix

Load the image into every kind node:

```bash
kind load docker-image myapp:1.0 --name lab
docker exec lab-control-plane crictl images | grep myapp      # now it is there
```

The kubelet retries on its own at the next back-off. To skip the wait, recreate the Pod:

```bash
kubectl delete pod local-image
kubectl apply -f local-image.yml
kubectl get pods                # Running
```

### Pitfalls

| Pitfall | Why | Fix |
|---|---|---|
| Image tagged `:latest` (or no tag) still fails after loading | The default pull policy for `latest` is `Always`, so the kubelet tries Docker Hub anyway | Use an explicit tag (`myapp:1.0`) or set `imagePullPolicy: IfNotPresent` |
| Rebuilt the image but kept the same tag | The node keeps using the old copy | Use a new tag for every build, then `kind load` again |
| Multi-node cluster, image only on one node | Pod scheduled on a node without the image | `kind load docker-image` loads onto all nodes in the cluster, so use it rather than copying by hand |

### Alternative: a real registry

For realistic practice, push to a registry (Docker Hub, GHCR) and pull from there, as in [Lab 4](#lab-4-private-registry-without-credentials).

### 6. Clean up

```bash
kubectl delete pod local-image
docker exec lab-control-plane crictl rmi docker.io/library/myapp:1.0   # optional
rm -rf local-image
```

---

## Lab 6: imagePullPolicy problems

`imagePullPolicy` controls **when** the kubelet pulls the image.

| Policy | Behavior | Default when |
|---|---|---|
| `IfNotPresent` | Pull only if the node doesn't already have the image | Tag is specified and isn't `latest` |
| `Always` | Always check the registry (fast if unchanged, fails if registry unreachable) | Tag is `latest` or missing |
| `Never` | Never pull; use only what is already on the node | Must be set explicitly |

### The `Never` failure: ErrImageNeverPull

#### 1. Create the file

```bash
cat > never-pull.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: never-pull
spec:
  containers:
    - name: web
      image: httpd:2.4
      imagePullPolicy: Never        # node must already have the image
EOF
```

#### 2. Apply and check

```bash
kubectl apply -f never-pull.yml
kubectl get pods
```

```
NAME         READY   STATUS              RESTARTS   AGE
never-pull   0/1     ErrImageNeverPull   0          10s
```

#### 3. Diagnose

```bash
kubectl describe pod never-pull
```

```
  Warning  ErrImageNeverPull  Container image "httpd:2.4" is not present with pull policy of Never
```

**Clue:** status `ErrImageNeverPull` (no back-off retry loop to a registry, because pulling is forbidden).

#### 4. Fix

Either allow pulling:

```bash
kubectl delete pod never-pull
sed -i '/imagePullPolicy: Never/d' never-pull.yml
kubectl apply -f never-pull.yml
```

Or make sure the node has the image (`kind load docker-image httpd:2.4 --name lab`).

(`imagePullPolicy` cannot be changed on a running Pod, so delete and recreate.)

#### 5. Clean up

```bash
kubectl delete pod never-pull
```

### The `latest` / `Always` failure

This one is covered in the [Lab 5 pitfalls](#pitfalls): a locally loaded image with `:latest` still fails because the policy is `Always`.

---

## Causes you cannot easily reproduce

These are common in real life. Learn to recognize them.

### Docker Hub rate limit

```
Failed to pull image "nginx:1.27": ... toomanyrequests: You have reached your pull rate limit.
```

Anonymous pulls from Docker Hub are limited per IP. Many nodes behind one NAT, or CI clusters, hit this quickly.

**Fix:** authenticate with an `imagePullSecret` (see Lab 4), use a registry mirror or cache, or host images in your own registry (GHCR, ECR, GCR).

### CPU architecture mismatch

```
Failed to pull image "...": no match for platform in manifest: not found
no matching manifest for linux/arm64/v8 in the manifest list entries
```

The image was built for `amd64` only, but the node is `arm64` (or the reverse). Common on Apple Silicon.

**Fix:** use a multi-architecture image, build with `docker buildx build --platform linux/amd64,linux/arm64`, or choose a different image/tag.

### Expired or rotated credentials

The Pod worked yesterday but fails today with `unauthorized`. Cloud registry tokens (for example AWS ECR, valid about 12 hours) expire.

**Fix:** refresh the secret, or use the cloud provider's native node/service-account authentication instead of a static secret.

### Image pulled but too large or slow

Events show `Pulling image` for a very long time, then a timeout.

**Fix:** use smaller base images (alpine, slim, distroless), pre-pull images on nodes, or use a closer registry.

---

## ImagePullBackOff in a Deployment

A Deployment is where this happens in real life, usually after changing the image tag.

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl rollout status deployment/web

# push a bad tag
kubectl set image deployment/web nginx=nginx:does-not-exist
kubectl get pods
```

```
NAME                   READY   STATUS             RESTARTS   AGE
web-6d8c9b7f5d-abcde   1/1     Running            0          2m      <- old Pods keep serving
web-6d8c9b7f5d-fghij   1/1     Running            0          2m
web-7f4b8c6d9c-klmno   0/1     ImagePullBackOff   0          20s     <- new Pod is stuck
```

The rolling update **stops** and the old Pods keep running, so users aren't affected. That is the safety built into rolling updates.

```bash
kubectl rollout status deployment/web --timeout=30s    # times out: waiting for rollout
kubectl describe pod <new-pod>                         # find the pull error
```

**Fix:** either set the correct image, or roll back:

```bash
kubectl set image deployment/web nginx=nginx:1.27      # correct the tag
# or
kubectl rollout undo deployment/web                    # go back to the last working revision
```

Clean up:

```bash
kubectl delete deployment web
```

---

## Look-alike errors

| Status | Meaning | How it differs |
|---|---|---|
| **InvalidImageName** | The image string is malformed (uppercase letters in the repository, bad characters). | The kubelet rejects it before trying to pull. Fix the string. |
| **ErrImageNeverPull** | `imagePullPolicy: Never` and the image isn't on the node. | See Lab 6. |
| **CreateContainerConfigError** | The image was pulled fine, but a referenced Secret/ConfigMap is missing. | Image is OK; fix the missing object. |
| **CrashLoopBackOff** | The image pulled and the container started, then crashed. | See the CrashLoopBackOff README. |
| **Pending** with no pull events | Pod isn't scheduled to a node yet. | Check `FailedScheduling` in the events. |

Quick demo of `InvalidImageName`:

```bash
kubectl run invalid --image=Nginx:1.27
kubectl get pods
# STATUS: InvalidImageName
kubectl describe pod invalid      # "invalid reference format: repository name must be lowercase"
kubectl delete pod invalid
```

---

## Quick diagnosis table

| Message in Events | Most likely cause | Fix |
|---|---|---|
| `repository does not exist or may require authorization` | Typo in name, **or** private repo without credentials | Check the spelling, then add `imagePullSecrets` |
| `not found` / `manifest unknown` | Tag doesn't exist | Use a tag that exists |
| `no such host` | Wrong registry hostname or DNS problem | Fix the hostname, check DNS |
| `i/o timeout` / `context deadline exceeded` | Node can't reach the registry | Check firewall, proxy, internet |
| `x509: certificate signed by unknown authority` | Private CA or self-signed cert | Install the CA on the nodes |
| `unauthorized` / `401` / `authentication required` | Missing, wrong, or expired credentials | Create or refresh the pull secret |
| `toomanyrequests` | Docker Hub rate limit | Authenticate, use a mirror |
| `no matching manifest for linux/arm64` | Architecture mismatch | Use a multi-arch image |
| Works locally with Docker, fails in kind | Image not loaded into the kind nodes | `kind load docker-image <image> --name <cluster>` |
| `ErrImageNeverPull` | `imagePullPolicy: Never` and image absent | Load the image or change the policy |
| New Deployment Pods stuck, old Pods fine | Bad image in the new revision | Fix the image or `kubectl rollout undo` |

---

## The debugging workflow

```
1. kubectl get pods                  -> confirm ErrImagePull / ImagePullBackOff, RESTARTS is 0
2. kubectl describe pod <pod>        -> read the Events: copy the exact pull error
3. Read the image string             -> kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'
4. Reproduce the pull yourself       -> docker pull <image>
                                        docker exec <node> crictl pull <image>
5. Match the error to the table above, then fix:
     - wrong name/tag     -> kubectl set image pod/<pod> <container>=<image>
     - missing credentials-> create secret + imagePullSecrets (recreate Pod)
     - local image (kind) -> kind load docker-image <image> --name <cluster>
6. kubectl get pods -w               -> watch it reach Running
```

---

## Clean up everything

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace imagelab
rm -f typo-image.yml bad-tag.yml bad-registry.yml private-image.yml \
      local-image.yml never-pull.yml
rm -rf local-image
```

## Practice challenges

1. Create a Pod that pulls `nginx:1.27` successfully, then use `kubectl set image` to change it to a broken tag and watch how the status changes. Then fix it again.
2. Run `docker exec <node> crictl pull <bad-image>` and compare its error message with what `kubectl describe pod` shows.
3. Build two versions of a local image (`myapp:1.0` and `myapp:2.0`), load only `1.0`, and deploy `2.0` to reproduce Lab 5 inside a Deployment. Then fix it and roll out.
4. Without looking at the table, write down which error message goes with each cause: typo, wrong tag, bad hostname, private repo, rate limit.
