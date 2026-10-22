# CKA Practice — Storage

> **Curriculum coverage:** Storage — storage classes & dynamic provisioning; volume types, access modes & reclaim policies; PVs & PVCs. Troubleshooting — storage-related pod and claim failures. Workloads — StatefulSet volume claim templates, init/sidecar containers sharing volumes.
> **Exercises:** 20 (Easy 6 · Medium 8 · Advanced 6)
> **Setup mode:** Manual
> **Environment:** kind cluster, assumed 1 control plane + 2 workers; shell on the kind host with `docker` access; Kubernetes ≥ 1.32 (exercise 17)

## How this sheet works

- Everything lives in namespace `cka-storage`; cluster-scoped objects are prefixed `cka-storage-`.
- Exercises 12, 13, 18 and 19 are troubleshooting tasks. Each has a collapsed **Fault** block directly above it: expand it, run it **without reading**, collapse it, then start the exercise. Apply each fault just before its own exercise, not all up front.
- Exercises are independent of each other; only exercise 6's expected output changes if exercise 3 is done first.

## Setup

1. Namespace and default context:

   ```bash
   kubectl create namespace cka-storage
   kubectl config set-context --current --namespace=cka-storage
   ```

2. A claim waiting for a volume (used by ex 3):

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: logs-claim
     namespace: cka-storage
   spec:
     storageClassName: manual
     accessModes: ["ReadWriteOnce"]
     resources:
       requests:
         storage: 500Mi
   EOF
   ```

3. A log-writing Deployment (used by ex 7):

   ```bash
   kubectl create deployment reporter --image=busybox:1.36 \
     -- sh -c 'while true; do date >> /var/log/report.log; sleep 5; done'
   ```

4. ConfigMaps and a Secret (used by ex 5, 9, 10):

   ```bash
   kubectl create configmap app-settings --from-literal=LOG_LEVEL=info --from-literal=MODE=batch
   kubectl create configmap web-config \
     --from-literal=index.html='<h1>Served from a ConfigMap</h1>' \
     --from-literal=maintenance.html='<h1>Down for maintenance</h1>'
   kubectl create secret generic db-cred --from-literal=username=admin --from-literal=password=s3cret
   ```

5. Statically provisioned volumes (used by ex 6, 11, 15) and a claim bound to one of them:

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: cka-storage-pv-small
   spec:
     capacity: {storage: 100Mi}
     accessModes: ["ReadWriteOnce"]
     storageClassName: manual-small
     hostPath: {path: /mnt/cka-small, type: DirectoryOrCreate}
   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: cka-storage-pv-silver
     labels: {tier: silver}
   spec:
     capacity: {storage: 2Gi}
     accessModes: ["ReadWriteOnce"]
     storageClassName: manual-tiered
     hostPath: {path: /mnt/cka-silver, type: DirectoryOrCreate}
   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: cka-storage-pv-gold
     labels: {tier: gold}
   spec:
     capacity: {storage: 3Gi}
     accessModes: ["ReadWriteOnce"]
     storageClassName: manual-tiered
     hostPath: {path: /mnt/cka-gold, type: DirectoryOrCreate}
   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: cka-storage-pv-archive
   spec:
     capacity: {storage: 1Gi}
     accessModes: ["ReadWriteOnce"]
     persistentVolumeReclaimPolicy: Retain
     storageClassName: manual-archive
     hostPath: {path: /mnt/cka-archive, type: DirectoryOrCreate}
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: archive-old
     namespace: cka-storage
   spec:
     storageClassName: manual-archive
     accessModes: ["ReadWriteOnce"]
     resources:
       requests: {storage: 1Gi}
   EOF
   ```

6. Headless Service for a StatefulSet (used by ex 17):

   ```bash
   kubectl create service clusterip cache --clusterip=None --tcp=6379:6379
   ```

7. A directory on the first worker node (used by ex 16):

   ```bash
   W1=$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
   echo "first worker: $W1"
   docker exec "$W1" mkdir -p /mnt/cka-local
   ```

8. Legacy data to migrate (used by ex 20):

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: legacy-data
     namespace: cka-storage
   spec:
     accessModes: ["ReadWriteOnce"]
     resources:
       requests: {storage: 256Mi}
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: legacy
     namespace: cka-storage
   spec:
     containers:
     - name: app
       image: busybox:1.36
       command: ["sh", "-c", "for i in $(seq 1 50); do echo \"record $i\" >> /data/records.txt; done; sleep 1d"]
       volumeMounts:
       - {name: data, mountPath: /data}
     volumes:
     - name: data
       persistentVolumeClaim: {claimName: legacy-data}
   EOF
   ```

9. Verify:

   ```bash
   kubectl get pvc,deploy,pod,cm,secret,svc
   # logs-claim Pending, archive-old Bound, legacy-data Bound, reporter 1/1, legacy Running
   kubectl get pv | grep cka-storage-
   # small/silver/gold Available, archive Bound
   ```

## Exercises

### Exercise 1 — Inspect the default StorageClass · Easy

Find the cluster's default StorageClass. Write its name, provisioner, reclaim policy and volume binding mode, one per line in that order, to `/tmp/cka-storage-ex1.txt`.

<details>
<summary>Solution</summary>

```bash
kubectl get sc        # default is marked "(default)" — on kind: standard
kubectl get sc standard -o jsonpath='{.metadata.name}{"\n"}{.provisioner}{"\n"}{.reclaimPolicy}{"\n"}{.volumeBindingMode}{"\n"}' > /tmp/cka-storage-ex1.txt
```

**Verify:**

```bash
cat /tmp/cka-storage-ex1.txt
# standard
# rancher.io/local-path
# Delete
# WaitForFirstConsumer
```

**Why:** The default class is the one annotated `storageclass.kubernetes.io/is-default-class: "true"`. Its binding mode explains behaviour you'll see in several later exercises.

</details>

### Exercise 2 — Dynamic provisioning and delayed binding · Easy

Create a PVC `notes-data` (256Mi, RWO) using the default StorageClass. **Before** anything uses it, write its status to `/tmp/cka-storage-ex2.txt`. Then create a pod `notes` (image `busybox:1.36`) that mounts the claim at `/data` and appends the current date to `/data/notes.txt` once, then sleeps.

<details>
<summary>Solution</summary>

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 256Mi}
EOF
kubectl get pvc notes-data -o jsonpath='{.status.phase}' > /tmp/cka-storage-ex2.txt   # Pending
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: notes
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "date >> /data/notes.txt; sleep 1d"]
    volumeMounts:
    - {name: data, mountPath: /data}
  volumes:
  - name: data
    persistentVolumeClaim: {claimName: notes-data}
```

**Verify:**

```bash
cat /tmp/cka-storage-ex2.txt              # Pending
kubectl get pvc notes-data                # Bound
kubectl exec notes -- cat /data/notes.txt # one date line
```

**Why:** With `WaitForFirstConsumer`, a PVC stays `Pending` until a pod is scheduled, so the volume can be created on the right node. On the exam, a Pending claim in this situation is expected, not a fault.

</details>

### Exercise 3 — Satisfy a pending claim · Easy

PVC `logs-claim` is Pending. Without modifying it, create a PersistentVolume `cka-storage-pv-logs` that it will bind to. Use `hostPath` `/mnt/cka-logs` and reclaim policy `Retain`.

<details>
<summary>Solution</summary>

```bash
kubectl get pvc logs-claim -o yaml    # storageClassName: manual, 500Mi, ReadWriteOnce
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cka-storage-pv-logs
spec:
  capacity: {storage: 500Mi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath: {path: /mnt/cka-logs, type: DirectoryOrCreate}
```

**Verify:** `kubectl get pvc logs-claim` shows `Bound` with volume `cka-storage-pv-logs`.

**Why:** Binding requires the same `storageClassName`, a compatible access mode, and capacity greater than or equal to the request. Read the claim before writing the PV; a class-name mismatch is the usual reason nothing binds.

</details>

### Exercise 4 — Memory-backed scratch space · Easy

Create a pod `cache-pod` (image `busybox:1.36`, command `sleep 1d`) with a scratch volume at `/cache` that is stored in RAM, not on the node's disk, and is limited to 64Mi.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "1d"]
    volumeMounts:
    - {name: scratch, mountPath: /cache}
  volumes:
  - name: scratch
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

**Verify:** `kubectl exec cache-pod -- df -h /cache` shows a `tmpfs` filesystem of about 64M.

**Why:** `medium: Memory` mounts tmpfs. What you write there counts toward the container's memory usage, and exceeding `sizeLimit` gets the pod evicted.

</details>

### Exercise 5 — ConfigMap as files · Easy

Create a pod `config-reader` (image `busybox:1.36`, command `sleep 1d`) that exposes every key of ConfigMap `app-settings` as a file under `/etc/app`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-reader
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "1d"]
    volumeMounts:
    - {name: settings, mountPath: /etc/app}
  volumes:
  - name: settings
    configMap: {name: app-settings}
```

**Verify:**

```bash
kubectl exec config-reader -- ls /etc/app          # LOG_LEVEL  MODE
kubectl exec config-reader -- cat /etc/app/MODE    # batch
```

**Why:** Each key becomes a file. Unlike environment variables, a mounted ConfigMap is updated in the running pod after the ConfigMap changes (with a short delay), as long as you don't use `subPath`.

</details>

### Exercise 6 — Sort volumes by size · Easy

List the names of all PersistentVolumes whose name starts with `cka-storage-`, ordered from smallest to largest capacity, one name per line, into `/tmp/cka-storage-ex6.txt`.

<details>
<summary>Solution</summary>

```bash
kubectl get pv --sort-by=.spec.capacity.storage \
  -o custom-columns=NAME:.metadata.name --no-headers | grep '^cka-storage-' > /tmp/cka-storage-ex6.txt
```

**Verify:**

```bash
cat /tmp/cka-storage-ex6.txt
# cka-storage-pv-small
# cka-storage-pv-logs      (if exercise 3 is done)
# cka-storage-pv-archive
# cka-storage-pv-silver
# cka-storage-pv-gold
```

**Why:** `--sort-by` compares capacities as quantities, so `100Mi` sorts before `1Gi`. This exact pattern appears in the kubectl cheat sheet, and it's worth knowing where to find it on exam day.

</details>

### Exercise 7 — Sidecar reading a shared log · Medium

Deployment `reporter` writes a timestamp to `/var/log/report.log` every 5 seconds. Add a container `log-reader` (image `busybox:1.36`) that continuously prints that file to its own stdout. The two containers must share the log directory through a volume that exists only as long as the pod.

<details>
<summary>Solution</summary>

```bash
kubectl edit deployment reporter
```

The pod template spec becomes (container name from `kubectl create deployment` is `busybox`):

```yaml
    spec:
      containers:
      - name: busybox
        image: busybox:1.36
        command: ["sh", "-c", "while true; do date >> /var/log/report.log; sleep 5; done"]
        volumeMounts:
        - {name: logs, mountPath: /var/log}
      - name: log-reader
        image: busybox:1.36
        command: ["sh", "-c", "tail -F /var/log/report.log"]
        volumeMounts:
        - {name: logs, mountPath: /var/log}
      volumes:
      - name: logs
        emptyDir: {}
```

**Verify:**

```bash
kubectl rollout status deploy/reporter
kubectl logs deploy/reporter -c log-reader --tail=3
```

**Why:** "Exists only as long as the pod" means `emptyDir`. `tail -F` keeps retrying until the file appears; `tail -f` exits if the reader starts first.

</details>

### Exercise 8 — Custom StorageClass with Retain · Medium

Create a StorageClass `cka-storage-retain` that uses the same provisioner as the default class, keeps volumes after their claim is deleted, delays binding until a pod uses the claim, and allows volume expansion. It must **not** become the default. Then create PVC `db-data` (512Mi, RWO) using it and a pod `db` (image `redis:7-alpine`) mounting it at `/data`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cka-storage-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: cka-storage
spec:
  storageClassName: cka-storage-retain
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 512Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: db
  namespace: cka-storage
spec:
  containers:
  - name: redis
    image: redis:7-alpine
    volumeMounts:
    - {name: data, mountPath: /data}
  volumes:
  - name: data
    persistentVolumeClaim: {claimName: db-data}
```

**Verify:**

```bash
kubectl get sc                         # only "standard" marked (default)
kubectl get pvc db-data                # Bound
kubectl get pv $(kubectl get pvc db-data -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'   # Retain
```

**Why:** Dynamically provisioned PVs inherit their reclaim policy from the class. local-path doesn't actually resize volumes, so here you practise the field only; an expansion request would not complete on kind.

</details>

### Exercise 9 — One file from a ConfigMap without hiding the directory · Medium

Create a pod `web` (image `nginx:1.27`) that serves the `index.html` key of ConfigMap `web-config` as `/usr/share/nginx/html/index.html`. nginx's other files in that directory (e.g. `50x.html`) must remain, and `maintenance.html` must not appear in the container.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: cka-storage
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html/index.html
      subPath: index.html
  volumes:
  - name: html
    configMap:
      name: web-config
      items:
      - {key: index.html, path: index.html}
```

**Verify:**

```bash
kubectl exec web -- ls /usr/share/nginx/html     # 50x.html  index.html
kubectl exec web -- curl -s localhost            # <h1>Served from a ConfigMap</h1>
```

**Why:** Mounting a volume on a directory hides what was there; `subPath` mounts a single file. The cost: `subPath` mounts don't receive ConfigMap updates (compare with exercise 5).

</details>

### Exercise 10 — One directory, three sources · Medium

Create a pod `bundle` (image `busybox:1.36`, command `sleep 1d`, labels `app=bundle`, `tier=demo`) with a single volume mounted at `/etc/bundle` that contains:

- `db/username` and `db/password` from Secret `db-cred`
- `app/mode` from key `MODE` of ConfigMap `app-settings`
- `meta/labels` containing the pod's own labels

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bundle
  namespace: cka-storage
  labels: {app: bundle, tier: demo}
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "1d"]
    volumeMounts:
    - {name: bundle, mountPath: /etc/bundle, readOnly: true}
  volumes:
  - name: bundle
    projected:
      sources:
      - secret:
          name: db-cred
          items:
          - {key: username, path: db/username}
          - {key: password, path: db/password}
      - configMap:
          name: app-settings
          items:
          - {key: MODE, path: app/mode}
      - downwardAPI:
          items:
          - path: meta/labels
            fieldRef: {fieldPath: metadata.labels}
```

**Verify:**

```bash
kubectl exec bundle -- find /etc/bundle -type f -not -path '*/..*'
kubectl exec bundle -- cat /etc/bundle/db/username /etc/bundle/app/mode /etc/bundle/meta/labels
# admin / batch / app="bundle" tier="demo"
```

**Why:** A `projected` volume merges secret, configMap, downwardAPI and serviceAccountToken sources into one directory. It's in the docs under "Projected Volumes", so search for that rather than writing it from memory.

</details>

### Exercise 11 — Bind to a specific tier · Medium

Two Available volumes share StorageClass `manual-tiered`: `cka-storage-pv-silver` and `cka-storage-pv-gold`. Create a PVC `gold-claim` (1Gi, RWO, class `manual-tiered`) that is guaranteed to bind to a volume labelled `tier=gold`, without naming the PV in the claim.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gold-claim
  namespace: cka-storage
spec:
  storageClassName: manual-tiered
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 1Gi}
  selector:
    matchLabels:
      tier: gold
```

**Verify:** `kubectl get pvc gold-claim` shows `Bound` with volume `cka-storage-pv-gold`.

**Why:** Without the selector, the binder picks the smallest volume that fits, which here would be silver (2Gi). A label `selector` limits the candidates; `volumeName` pins a single PV (used in exercise 15).

</details>

<details>
<summary>Fault for exercise 12 — apply without reading</summary>

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests: {storage: 128Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: shared-user
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo ok > /shared/status; sleep 1d"]
    volumeMounts:
    - {name: shared, mountPath: /shared}
  volumes:
  - name: shared
    persistentVolumeClaim: {claimName: shared-data}
EOF
```

</details>

### Exercise 12 — Claim that never provisions · Medium

Pod `shared-user` stays Pending. Find out why and get it running with a 128Mi volume from the default StorageClass mounted at `/shared`. Only this one pod uses the volume. Keep the names `shared-user` and `shared-data`.

<details>
<summary>Solution</summary>

```bash
kubectl describe pod shared-user | tail -5        # waiting for volume / unbound claim
kubectl describe pvc shared-data | tail -5        # ProvisioningFailed ... access mode not supported
kubectl get pvc shared-data -o jsonpath='{.spec.accessModes}'   # ["ReadWriteMany"]
```

A claim's access modes can't be edited, so recreate both objects:

```bash
kubectl get pod shared-user -o yaml > /tmp/shared-user.yaml
kubectl delete pod shared-user
kubectl delete pvc shared-data
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 128Mi}
EOF
kubectl apply -f /tmp/shared-user.yaml
```

**Verify:** `kubectl get pod shared-user` is Running, and `kubectl exec shared-user -- cat /shared/status` prints `ok`.

**Why:** local-path only supports node-local access modes, so an RWX request fails at provisioning. Both the PVC and the pod spec are largely immutable, so recreating them is the normal fix. Delete the pod first, otherwise PVC protection can hold the claim in `Terminating`.

</details>

<details>
<summary>Fault for exercise 13 — apply without reading</summary>

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audit-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 128Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: auditor
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "date >> /audit/audit.log && sleep 1d"]
    volumeMounts:
    - {name: audit, mountPath: /audit, readOnly: true}
  volumes:
  - name: audit
    persistentVolumeClaim: {claimName: audit-data}
EOF
```

</details>

### Exercise 13 — Pod crashes on start · Medium

Pod `auditor` keeps restarting. Find the cause and fix it so it runs, keeping its existing claim, mount path and command.

<details>
<summary>Solution</summary>

```bash
kubectl get pod auditor                      # CrashLoopBackOff
kubectl logs auditor --previous              # can't create /audit/audit.log: Read-only file system
kubectl get pod auditor -o yaml > /tmp/auditor.yaml
# delete the line "readOnly: true" under volumeMounts (or set it to false)
kubectl replace --force -f /tmp/auditor.yaml
```

**Verify:** `kubectl get pod auditor` is Running with 0 new restarts, and `kubectl exec auditor -- cat /audit/audit.log` shows a date.

**Why:** Volume mounts can't be changed on a running pod. `kubectl replace --force` deletes and recreates it in one step, which is the fastest exam-day pattern for "fix this pod". Check `--previous` logs first, because the current container may not have failed yet.

</details>

### Exercise 14 — Populate content before the app starts · Medium

Create a pod `site` whose `nginx:1.27` container serves an `index.html` containing `<h1>Built by init</h1>`. The file must be generated at pod start by an init container `build` (image `busybox:1.36`) and handed to nginx through a volume. Don't use a ConfigMap.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: site
  namespace: cka-storage
spec:
  initContainers:
  - name: build
    image: busybox:1.36
    command: ["sh", "-c", "echo '<h1>Built by init</h1>' > /work/index.html"]
    volumeMounts:
    - {name: html, mountPath: /work}
  containers:
  - name: nginx
    image: nginx:1.27
    volumeMounts:
    - {name: html, mountPath: /usr/share/nginx/html}
  volumes:
  - name: html
    emptyDir: {}
```

**Verify:** `kubectl exec site -- curl -s localhost` returns `<h1>Built by init</h1>`.

**Why:** Init containers run to completion before app containers start, and an `emptyDir` persists across containers within the pod. Mount paths may differ between containers; the volume is the same.

</details>

### Exercise 15 — Reuse a retained volume · Advanced

The claim `archive-old` is no longer needed, but the data on its volume `cka-storage-pv-archive` must be kept. Delete `archive-old`, then make the **same** PV bind to a new claim `archive-new` (1Gi, RWO). Don't delete or recreate the PV.

<details>
<summary>Solution</summary>

```bash
kubectl delete pvc archive-old
kubectl get pv cka-storage-pv-archive      # Released
kubectl patch pv cka-storage-pv-archive --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
kubectl get pv cka-storage-pv-archive      # Available
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: archive-new
  namespace: cka-storage
spec:
  storageClassName: manual-archive
  volumeName: cka-storage-pv-archive
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 1Gi}
```

**Verify:** `kubectl get pvc archive-new` shows `Bound` to `cka-storage-pv-archive`.

**Why:** A `Retain` PV whose claim is deleted becomes `Released`, and its `claimRef` still carries the old claim's UID, so nothing else can bind. Removing the `claimRef` frees it; `volumeName` makes sure the new claim gets this exact PV.

</details>

### Exercise 16 — Node-pinned local storage · Advanced

The directory `/mnt/cka-local` exists on the first worker node (setup step 7). Make it available as persistent storage:

- StorageClass `cka-storage-local`: no dynamic provisioning; binding waits until a pod uses the claim
- PersistentVolume `cka-storage-pv-local`: 1Gi, RWO, `Retain`, using a **local** volume (not `hostPath`) on that directory
- PVC `local-claim` (1Gi) and pod `local-user` (image `busybox:1.36`) that writes `hello` to `/data/hello.txt` and sleeps

The pod must always land on the node that holds the directory.

<details>
<summary>Solution</summary>

```bash
W1=$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cka-storage-local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cka-storage-pv-local
spec:
  capacity: {storage: 1Gi}
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cka-storage-local
  local:
    path: /mnt/cka-local
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values: ["$W1"]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-claim
  namespace: cka-storage
spec:
  storageClassName: cka-storage-local
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 1Gi}
---
apiVersion: v1
kind: Pod
metadata:
  name: local-user
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo hello > /data/hello.txt; sleep 1d"]
    volumeMounts:
    - {name: data, mountPath: /data}
  volumes:
  - name: data
    persistentVolumeClaim: {claimName: local-claim}
EOF
```

(The heredoc delimiter is unquoted on purpose so the shell substitutes `$W1`.)

**Verify:**

```bash
kubectl get pod local-user -o wide                      # NODE = first worker
docker exec "$W1" cat /mnt/cka-local/hello.txt         # hello
```

**Why:** `local` volumes require `nodeAffinity` on the PV; the scheduler then places consuming pods on that node. `WaitForFirstConsumer` lets the scheduler weigh the pod's other constraints before binding. `no-provisioner` means you create the PVs yourself.

</details>

### Exercise 17 — Per-replica storage, cleaned up on scale-down · Advanced

Create a StatefulSet `cache` using the existing headless Service `cache`:

- 3 replicas, image `redis:7-alpine`, pod label `app=cache`, container port 6379
- each replica gets its own 128Mi RWO volume from the default class, mounted at `/data`
- claims of replicas removed by a scale-down are deleted automatically; deleting the StatefulSet keeps all claims

Then scale to 1 replica and confirm the result.

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cache
  namespace: cka-storage
spec:
  serviceName: cache
  replicas: 3
  selector:
    matchLabels: {app: cache}
  persistentVolumeClaimRetentionPolicy:
    whenScaled: Delete
    whenDeleted: Retain
  template:
    metadata:
      labels: {app: cache}
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - {name: data, mountPath: /data}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests: {storage: 128Mi}
```

```bash
kubectl rollout status sts/cache
kubectl get pvc | grep data-cache      # data-cache-0, -1, -2
kubectl scale sts cache --replicas=1
```

**Verify:** After a few seconds, `kubectl get pvc | grep data-cache` shows only `data-cache-0`.

**Why:** Claim names follow `<template>-<statefulset>-<ordinal>`. StatefulSets keep claims forever by default; `persistentVolumeClaimRetentionPolicy` (GA in 1.32) changes that.

</details>

<details>
<summary>Fault for exercise 18 — apply without reading</summary>

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: old-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 128Mi}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: old-app
  namespace: cka-storage
spec:
  replicas: 1
  selector:
    matchLabels: {app: old-app}
  template:
    metadata:
      labels: {app: old-app}
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sleep", "1d"]
        volumeMounts:
        - {name: data, mountPath: /data}
      volumes:
      - name: data
        persistentVolumeClaim: {claimName: old-data}
EOF
kubectl rollout status deploy/old-app
kubectl delete pvc old-data --wait=false
```

</details>

### Exercise 18 — Claim stuck in Terminating · Advanced

PVC `old-data` has been stuck in `Terminating`. Find out why. Then move Deployment `old-app` onto a new claim `new-data` (128Mi, default class, same mount path) so that `old-data` finishes deleting on its own. Don't edit finalizers.

<details>
<summary>Solution</summary>

```bash
kubectl get pvc old-data -o jsonpath='{.metadata.finalizers}'   # ["kubernetes.io/pvc-protection"]
kubectl describe pvc old-data | grep -i "used by"                # Used By: old-app-...
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: new-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 128Mi}
EOF
kubectl patch deployment old-app --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/0/persistentVolumeClaim/claimName","value":"new-data"}]'
kubectl rollout status deploy/old-app
```

**Verify:**

```bash
kubectl get pvc old-data         # NotFound (after the old pod terminates)
kubectl get pvc new-data         # Bound
```

**Why:** The `pvc-protection` finalizer blocks deletion while any pod uses the claim. Removing the finalizer by hand would yank storage from a running pod. The right fix is to stop the usage, after which the claim is deleted, along with its PV, because the default class uses `Delete`.

</details>

<details>
<summary>Fault for exercise 19 — apply without reading</summary>

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pipeline-data
  namespace: cka-storage
spec:
  storageClassName: fast-ssd
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 200Mi}
---
apiVersion: v1
kind: Pod
metadata:
  name: pipeline
  namespace: cka-storage
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "cat /etc/pipeline/mode && sleep 1d"]
    volumeMounts:
    - {name: work, mountPath: /work}
    - {name: config, mountPath: /etc/pipeline}
  volumes:
  - name: work
    persistentVolumeClaim: {claimName: pipeline-data}
  - name: config
    configMap: {name: pipeline-config}
EOF
```

</details>

### Exercise 19 — Pod that won't start, two reasons · Advanced

Pod `pipeline` isn't running. Get it to `Running` with:

- a 200Mi volume from the **default** StorageClass at `/work` (claim name `pipeline-data`)
- a file `/etc/pipeline/mode` containing `batch`

Keep the pod's name, command and mount paths.

<details>
<summary>Solution</summary>

```bash
kubectl describe pod pipeline | tail -5        # unbound PersistentVolumeClaim
kubectl describe pvc pipeline-data | tail -3   # storageclass.storage.k8s.io "fast-ssd" not found
kubectl get pod pipeline -o yaml > /tmp/pipeline.yaml
kubectl delete pod pipeline
kubectl delete pvc pipeline-data
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pipeline-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 200Mi}
EOF
kubectl apply -f /tmp/pipeline.yaml
kubectl describe pod pipeline | tail -3        # MountVolume.SetUp failed: configmap "pipeline-config" not found
kubectl create configmap pipeline-config --from-literal=mode=batch
```

**Verify:** `kubectl get pod pipeline` shows Running (the kubelet retries the mount; allow up to a minute), and `kubectl logs pipeline` prints `batch`.

**Why:** Faults hide behind each other. The second one only appears once the pod gets scheduled, so after each fix, re-run `describe` instead of assuming you're done. A bound or provisioning PVC's `storageClassName` can't be changed, so recreating it is the fix.

</details>

### Exercise 20 — Migrate data to a new claim · Advanced

Pod `legacy` has written records to `/data/records.txt` on PVC `legacy-data`. Copy that data to a new PVC `modern-data` (512Mi, default class) using a Job `migrate` (image `busybox:1.36`) that runs once and completes. Leave `legacy` running. Then prove that `modern-data` holds all 50 records.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: modern-data
  namespace: cka-storage
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 512Mi}
---
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate
  namespace: cka-storage
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: copy
        image: busybox:1.36
        command: ["sh", "-c", "cp -a /src/. /dst/ && ls -l /dst"]
        volumeMounts:
        - {name: src, mountPath: /src, readOnly: true}
        - {name: dst, mountPath: /dst}
      volumes:
      - name: src
        persistentVolumeClaim: {claimName: legacy-data}
      - name: dst
        persistentVolumeClaim: {claimName: modern-data}
```

**Verify:**

```bash
kubectl wait --for=condition=complete job/migrate --timeout=120s
kubectl run check --rm -it --restart=Never --image=busybox:1.36 \
  --overrides='{"spec":{"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"modern-data"}}],"containers":[{"name":"check","image":"busybox:1.36","command":["wc","-l","/d/records.txt"],"volumeMounts":[{"name":"d","mountPath":"/d"}]}]}}'
# 50 /d/records.txt
```

**Why:** RWO restricts a volume to one **node**, not one pod, so the Job can mount `legacy-data` alongside the running `legacy` pod. The scheduler puts the Job on that node because of the local-path PV's node affinity. Since `modern-data` uses `WaitForFirstConsumer`, it's provisioned on the same node, so both mounts work.

</details>
