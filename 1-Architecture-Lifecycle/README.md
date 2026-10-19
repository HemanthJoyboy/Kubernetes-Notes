# Kubernetes Learning Notes

My notes on how Kubernetes works: the architecture, what happens when you create a workload, and how a Pod gets assigned to a node.

## Table of Contents

1. [Architecture: who does what](#1-architecture-who-does-what)
2. [What happens when you run `kubectl apply -f deployment.yaml`](#2-what-happens-when-you-run-kubectl-apply--f-deploymentyaml)
3. [How the scheduler assigns a Pod to a node](#3-how-the-scheduler-assigns-a-pod-to-a-node)

---

## 1. Architecture: who does what

A Kubernetes cluster is a **control plane** (the brain that decides) plus **worker nodes** (the machines that run your containers). Every component does one job and talks to the others through the API server.

### Control plane components

| Component | Job | Analogy |
|---|---|---|
| **kube-apiserver** | The only front door. Authenticates you, validates requests, and reads/writes the cluster state in etcd. Every other component talks to it, never to each other. | Reception desk |
| **etcd** | Key-value database holding the entire cluster state (every Pod, Deployment, Secret). Lose it and you lose the cluster's memory. | The filing cabinet |
| **kube-scheduler** | Watches for Pods with no node assigned and picks the best node, based on CPU/memory requests, taints, affinity. It only decides; it does not start anything. | Seating planner |
| **kube-controller-manager** | Runs many control loops (Deployment, ReplicaSet, Node, Job controllers). Each loop compares desired state with actual state and fixes the difference. | Thermostat |
| **cloud-controller-manager** | Talks to the cloud provider for load balancers, disks, and node lifecycle (only on cloud clusters). | Cloud liaison |

### Worker node components

| Component | Job |
|---|---|
| **kubelet** | Agent on every node. Watches the API server for Pods assigned to its node, tells the container runtime to start them, runs probes, and reports status back. |
| **Container runtime** (containerd, CRI-O) | Actually pulls images and runs containers. Kubernetes talks to it through the CRI interface. |
| **kube-proxy** | Programs iptables/IPVS rules on the node so Service IPs route to the right Pods. |
| **CNI plugin** (Calico, Flannel, Cilium) | Gives every Pod an IP and makes Pod-to-Pod networking work across nodes. |

### Three ideas that explain everything

1. **Declarative:** you describe the desired state in YAML ("I want 3 replicas"); you do not say how to get there.
2. **Reconciliation loop:** controllers constantly compare desired state (in etcd) with actual state and act to close the gap. This is why a deleted Pod comes back.
3. **Everything goes through the API server:** components never call each other directly. They watch the API server for changes that concern them.

---

## 2. What happens when you run `kubectl apply -f deployment.yaml`

Take a Deployment with 3 replicas of nginx. Here is the full journey, in order.

1. **kubectl** reads your YAML, converts it to JSON, finds your credentials and API server address in `~/.kube/config`, and sends an HTTPS request to the API server.
2. **kube-apiserver** runs the request through three gates: **authentication** (who are you?), **authorization** (RBAC: may you create Deployments here?), and **admission control** (mutating/validating webhooks, quotas, defaults). If any gate says no, you get an error immediately.
3. **kube-apiserver** validates the object schema and saves the Deployment to **etcd**. Your `kubectl apply` returns `deployment.apps/web created` here. Nothing is running yet, only the desired state is stored.
4. **Deployment controller** (inside controller-manager) sees a new Deployment via its watch on the API server. It creates a **ReplicaSet** object (again through the API server, saved in etcd).
5. **ReplicaSet controller** sees the ReplicaSet wants 3 Pods but 0 exist, so it creates 3 **Pod** objects. These Pods have no `nodeName` yet and are in `Pending` state.
6. **kube-scheduler** watches for Pods without a node. For each one it filters nodes (enough CPU/memory? taints? selectors?), scores the survivors, and writes the chosen node name into the Pod via the API server (a "binding").
7. **kubelet** on the chosen node watches the API server, notices a Pod is now assigned to it, and acts.
8. **kubelet** asks the container runtime (containerd) to pull the image and start the containers. The CNI plugin gives the Pod an IP. Volumes are mounted.
9. **kubelet** runs probes and reports Pod status (`Running`, `Ready`) back to the API server, which writes it to etcd.
10. **kube-proxy** updates network rules if a Service selects these Pods, so traffic can reach them.
11. **Forever after:** if a Pod dies, the ReplicaSet controller notices actual (2) differs from desired (3) and creates a new one. If a node dies, the node controller marks it `NotReady`, and Pods are rescheduled elsewhere.

### Flow at a glance

```
kubectl
  -> API server (authn -> authz -> admission -> validate -> save to etcd)
  -> Deployment controller   creates ReplicaSet
  -> ReplicaSet controller   creates Pods (Pending, no nodeName)
  -> Scheduler               binds each Pod to a node (sets nodeName)
  -> Kubelet                 pulls image, starts containers (via containerd + CNI)
  -> Kubelet                 reports Running / Ready
  -> kube-proxy              updates Service rules
```

### Who checks nodes, pods and containers?

| What | Checked by |
|---|---|
| Is the node alive? | kubelet sends heartbeats (Lease objects); node controller marks the node `NotReady` if they stop |
| Are the right number of Pods running? | ReplicaSet controller |
| Is the rollout progressing? | Deployment controller |
| Is the container healthy? | kubelet, via liveness/readiness/startup probes, and restarts it per `restartPolicy` |
| Where should a Pod run? | kube-scheduler (decides once, at creation) |
| Is the container actually running? | Container runtime, reported up by kubelet |

### See it live

Open two terminals. In one run:

```bash
kubectl get events --watch
```

In the other run:

```bash
kubectl apply -f deployment.yaml
```

You will see `Scheduled`, `Pulling`, `Pulled`, `Created`, `Started` in sequence, which is steps 6 to 8 above.

---

## 3. How the scheduler assigns a Pod to a node

> The assignment is just **one field** on the Pod object, saved in etcd. Nothing is "sent" to the node.

This section zooms in on step 6 and step 7 above.

### Where it is stored

Every Pod is a record (an object) in etcd. Before scheduling, the record looks like this:

```yaml
kind: Pod
metadata:
  name: web-abc12
spec:
  containers:
  - name: nginx
    image: nginx:1.27
  # nodeName: (empty)  <- no node chosen yet
status:
  phase: Pending
```

After scheduling, the only change is one field:

```yaml
spec:
  nodeName: worker-1    # <- the scheduler filled this in
```

That is the whole "assignment":

| `spec.nodeName` | Meaning |
|---|---|
| Empty | The Pod is **unscheduled** |
| Set (e.g. `worker-1`) | The Pod **belongs to that node** |

### How the scheduler writes it

1. The scheduler watches the API server: *"show me all Pods where `nodeName` is empty."*
2. It picks the best node (`worker-1`).
3. It sends the API server a small **Binding** request: *"bind Pod `web-abc12` to `worker-1`."*
4. The API server validates it and saves `spec.nodeName: worker-1` into that Pod's record in etcd.

```
Scheduler --(Binding request)--> API Server --(writes nodeName)--> etcd
```

The scheduler **never talks to the node** and **never touches etcd directly**.

### How the node finds out

The kubelet on every node keeps a watch on the API server: *"show me all Pods where `nodeName` equals my own node name."*

- The kubelet on **worker-1** sees the Pod appear in its list, so it starts it.
- The kubelet on **worker-2** never sees that Pod, so it ignores it.

So the node isn't *told*; it *notices* because its filter now matches. The `nodeName` field is the **handoff between the scheduler and the kubelet**.

### See it yourself

```bash
kubectl apply -f deployment.yaml
kubectl get pod <pod-name> -o yaml | grep -n "nodeName"
kubectl get pods -o wide          # NODE column reads from the same field
kubectl describe pod <pod-name>   # Events: "Successfully assigned default/web-abc12 to worker-1"
```

That Event line, `Successfully assigned ... to worker-1`, is the scheduler's binding being recorded.

### Try an unschedulable Pod

Create a Pod that asks for a node label that no node has yet:

```bash
kubectl run big --image=nginx --overrides='{"spec":{"nodeSelector":{"disk":"nvme"}}}'
kubectl get pod big -o wide                          # NODE is <none>, status Pending
kubectl get pod big -o jsonpath='{.spec.nodeName}'   # prints nothing
kubectl describe pod big                             # FailedScheduling event
```

Now label a node so the Pod fits, and watch the scheduler fill the field in:

```bash
kubectl label node <node-name> disk=nvme
kubectl get pod big -o wide                          # NODE now shows the node
```

Clean up:

```bash
kubectl delete pod big
kubectl label node <node-name> disk-
```

### Skipping the scheduler

You can bypass the scheduler by setting `nodeName` yourself in the YAML. This shows that the field alone is what makes the kubelet start the Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: worker-1      # no scheduler involved
  containers:
  - name: nginx
    image: nginx:1.27
```

> In production, prefer `nodeSelector`, affinity, or taints/tolerations so the scheduler still checks resources and constraints.

### Summary

- A Pod's node assignment is a single field: `spec.nodeName`, stored in etcd.
- The **scheduler** fills it in through a **Binding** request to the API server.
- The **kubelet** on each node watches for Pods whose `nodeName` matches its own name, and starts them.
- Nodes are not "told" anything; they notice via their watch.
- `kubectl describe pod` shows the binding as `Successfully assigned <pod> to <node>`.
