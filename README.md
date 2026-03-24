## CKA 2025 – Practice Questions (Summary & Answers)

This document reformats `CKA_Exam_2026.txt` into a clear, readable guide you can quickly revise from.

---

## Question 1 – MariaDB with PersistentVolume

**Scenario**  
A MariaDB `Deployment` in namespace `mariadb` was accidentally deleted. A `PersistentVolume` (PV) still exists and must be reused so that the data is preserved.

**Tasks**
- **Create PVC** `mariadb` (namespace `mariadb`)  
  - `accessModes: ReadWriteOnce`  
  - `storage: 250Mi`
- **Update Deployment manifest** `~/mariadb-deploy.yaml` to use this PVC.
- **Apply Deployment** and ensure the MariaDB `Deployment` is running.

**Example PVC YAML**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 250Mi
```

**Deployment volume section (simplified example)**

```yaml
spec:
  containers:
  - name: mariadb
    image: <mariadb-image>
    volumeMounts:
    - name: mariadb-storage
      mountPath: /var/lib/data

  volumes:
  - name: mariadb-storage
    persistentVolumeClaim:
      claimName: mariadb
```

---

## Question 2 – Install Argo CD with Helm

**Goal**  
Install Argo CD using Helm (chart from the official Argo Helm repo), version `7.7.3`, in namespace `argocd`, **without installing CRDs** (CRDs already exist).

**Steps**
1. **Add Helm repo**
   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   ```
2. **Generate template (no CRDs)**
   ```bash
   helm template argocd argo/argo-cd \
     --set crds.install=false \
     --version 7.7.3 \
     -n argocd > ./argo-helm.yaml
   ```
3. **Install Argo CD**
   ```bash
   helm install argocd argo/argo-cd \
     --set crds.install=false \
     --version 7.7.3 \
     -n argocd
   ```

---

## Question 3 – Sidecar Log Tailer

There are two variants; both are the same concept.

**Goal**  
Add a **sidecar container** to an existing `Deployment` so it tails a log file from a **shared volume**.

### Variant A – WordPress
- Deployment: existing WordPress Deployment.
- Add sidecar:
  - name: `sidecar`
  - image: `busybox:stable`
  - command: `/bin/sh -c 'tail -f /var/log/wordpress.log'`
- Use a shared volume mounted at `/var/log` in **both** the main and sidecar containers.

### Variant B – synergy-deployment (example YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergy-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergy
  template:
    metadata:
      labels:
        app: synergy
    spec:
      containers:
      - name: main-container
        image: <existing-image>
        volumeMounts:
        - name: log-volume
          mountPath: /var/log

      - name: sidecar
        image: busybox:stable
        command: ["/bin/sh","-c","tail -n+1 -f /var/log/synergy-deployment.log"]
        volumeMounts:
        - name: log-volume
          mountPath: /var/log

      volumes:
      - name: log-volume
        emptyDir: {}
```

---

## Question 4 – WordPress Resource Requests & Limits

**Scenario**  
WordPress `Deployment` should use fair, equal CPU/memory across 3 Pods, including `initContainers`, and add a bit of safety overhead.

**Given example node resources**
- CPU: 2 cores  
- Memory: 4Gi

**Rough calculation**
- Divide by 3 Pods → ~600m CPU, ~1.2Gi RAM each.  
- Add overhead → choose `500m` CPU and `1Gi` RAM.

**Steps**
1. Scale to 0:
   ```bash
   kubectl scale deployment wordpress --replicas=0
   ```
2. Check node allocatable resources:
   ```bash
   kubectl describe node | grep -A5 "Allocatable"
   ```
3. Edit Deployment:
   ```bash
   kubectl edit deployment wordpress
   ```
   Add **same** `resources` block to:
   - each container
   - each initContainer

   ```yaml
   resources:
     requests:
       cpu: "500m"
       memory: "1Gi"
     limits:
       cpu: "500m"
       memory: "1Gi"
   ```
4. Scale back to 3:
   ```bash
   kubectl scale deployment wordpress --replicas=3
   ```

---

## Question 5 – cert-manager CRDs & kubectl explain

**Tasks**
- List all cert-manager CRDs to `/root/resources.yaml`.
- Save `kubectl explain` output for `certificate.spec.subject` to `/root/subject.yaml`.

**Commands**
```bash
kubectl get crds | grep cert-manager > /root/resources.yaml
kubectl explain certificate.spec.subject > /root/subject.yaml
```

---

## Question 6 – PriorityClass & Deployment Patch

**Goal**
- Create `PriorityClass` `high-priority` for user workloads with a value one below the *highest* existing user-defined priority.
- Patch `busybox-logger` Deployment (namespace `priority`) to use it.

**Example PriorityClass YAML**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 99999
globalDefault: false
description: "High priority for user workloads"
```

**Example commands**

```bash
kubectl create priorityclass high-priority \
  --value=99999 \
  --global-default=false \
  --description="High priority for user workloads"

kubectl -n priority patch deployment busybox-logger \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/priorityClassName","value":"high-priority"}]'

kubectl -n priority rollout status deployment busybox-logger
```

---

## Question 7 – CNI Choice (Flannel vs Calico)

**Requirement**
- CNI must:
  - Allow Pod-to-Pod communication.
  - **Support NetworkPolicy**.
  - Be installed from a manifest.

**Decision**
- **Use Calico** (`v3.28.2`) because:
  - Flannel (v0.26.1) **does not support NetworkPolicy**.

**Command**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
```

---

## Question 8 – cri-dockerd Setup

**Tasks**
- Install `cri-dockerd` from `.deb` package.
- Enable and start the service.
- Configure kernel parameters via `sysctl`.

**Commands**

```bash
dpkg -i ~/cri-dockerd*.deb
# Optional if dependencies are broken:
apt-get install -f -y

systemctl daemon-reload
systemctl enable --now cri-docker
```

**sysctl configuration (`/etc/sysctl.d/99-kubernetes-cri.conf`)**

```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

Apply:

```bash
sysctl --system
```

---

## Question 9 – Taints & Tolerations

**Tasks**
- Taint `node01` so normal Pods cannot be scheduled:
  - key: `PERMISSION`
  - value: `granted`
  - effect: `NoSchedule`
- Create a Pod that tolerates this taint and runs **on `node01`**.

**Commands**

```bash
kubectl taint nodes node01 PERMISSION=granted:NoSchedule
kubectl describe node node01 | grep -i taint
```

**Example Pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerate-pod
spec:
  nodeName: node01
  tolerations:
  - key: "PERMISSION"
    operator: "Equal"
    value: "granted"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```

Apply and verify:

```bash
kubectl apply -f pod.yaml
kubectl get pods -o wide
```

---

## Question 10 – Migrate Ingress to Gateway API

**Scenario**
- Existing `Ingress` named `web` exposes an HTTPS site.
- Need to migrate to Gateway API:
  - `Gateway` named `web-gateway`
  - `HTTPRoute` named `web-route`
  - Hostname: `gateway.web.k8s.local`
  - Existing `GatewayClass`: `nginx-class`

**Steps**
1. Inspect existing Ingress:
   ```bash
   kubectl get ingress web -o yaml
   ```
   Assume:
   - host: `gateway.web.k8s.local`
   - paths: `/`
   - service: `web-service:80`
   - TLS secret: `web-secret`

2. Create `Gateway`:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
     name: web-gateway
   spec:
     gatewayClassName: nginx-class
     listeners:
     - name: https
       protocol: HTTPS
       port: 443
       hostname: gateway.web.k8s.local
       tls:
         mode: Terminate
         certificateRefs:
         - kind: Secret
           name: web-secret
   ```

3. Create `HTTPRoute`:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: web-route
   spec:
     parentRefs:
     - name: web-gateway
     hostnames:
     - gateway.web.k8s.local
     rules:
     - matches:
       - path:
           type: PathPrefix
           value: /
       backendRefs:
       - name: web-service
         port: 80
   ```

---

## Question 11 – Ingress for echo-service

> Note: In the original text there is a small mistake: the Service is on port 80, but the Ingress backend uses port 8080. Both must match. The fixed answer uses port 80 consistently.

**Scenario**
- Namespace: `echo-sound`
- Deployment: `echo`
- Create:
  - `Service` `echo-service` (type `NodePort`, port 80).
  - `Ingress` `echo` exposing `http://example.org/echo`.

### 1. Service

```bash
kubectl expose deployment echo \
  --name=echo-service \
  --port=80 \
  --target-port=80 \
  --type=NodePort \
  -n echo-sound
```

### 2. Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: echo-sound
spec:
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echo-service
            port:
              number: 80
```

### 3. Verify

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```
Expected output: `200`.

---

## Question 12 – NetworkPolicy (Least Permissive)

**Scenario**
- Deployments:
  - `frontend` in namespace `frontend`
  - `backend` in namespace `backend`
- There are three candidate NetworkPolicies (`network-policy-1.yaml`, `network-policy-2.yaml`, `network-policy-3.yaml`).  
  Choose the one that:
  - Allows frontend → backend traffic.
  - Is **least permissive** (only necessary access).

**Analysis**
- `policy-x` (network-policy-1):
  - Selects all Pods, allows all ingress.
  - Too permissive → reject.
- `policy-y` (network-policy-2):
  - Label mismatch and also allows extra `ipBlock` traffic.
  - Not least permissive → reject.
- `policy-z` (network-policy-3):
  - Selects only backend Pods with `app: backend`.
  - Allows ingress only from:
    - namespace with label `name: frontend`
    - Pods with label `app: frontend`
  - Restricts port to TCP/80.

✅ **Correct choice:** `network-policy-3.yaml` (`policy-z`).

Apply it:

```bash
kubectl apply -f network-policy-3.yaml
```

---

## Question 13 – StorageClass local-storage

**Tasks**
- Create StorageClass:
  - name: `local-storage`
  - provisioner: `rancher.io/local-path`
  - `volumeBindingMode: WaitForFirstConsumer`
  - set as **default**.
- Ensure it is the **only** default StorageClass (remove default annotation from others).

**Example StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```

**Remove default from other SCs (example)**

```bash
kubectl patch sc local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

---

## Question 14 – NodePort Service

**Scenario**
- Deployment: `nodeport-deployment`
- Namespace: `relative`

**Tasks**
- Ensure containers expose port 80/TCP.
- Create `Service` `nodeport-service`:
  - type: `NodePort`
  - `port: 80`
  - `targetPort: 80`
  - `nodePort: 30080`

**Example Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
  namespace: relative
spec:
  type: NodePort
  selector:
    app: nodeport-deployment   # adjust if label differs
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

Apply:

```bash
kubectl apply -f svc.yaml
```

---

## Question 15 – TLS v1.3 Only & Verification

**Scenario**
- Namespace: `nginx-static`
- A Deployment uses:
  - a ConfigMap (nginx TLS config supports TLSv1.2 and TLSv1.3).
  - a Secret for TLS.
- Service: `nginx-service` (ClusterIP) in same namespace.

**Tasks**
- Update ConfigMap so nginx **only supports TLSv1.3**.
- Add an `/etc/hosts` entry so that `ckaquestion.k8s.local` resolves to the service IP.
- Verify:
  - TLS 1.2 **fails**
  - TLS 1.3 **succeeds**

**Steps**
1. Find ConfigMap:
   ```bash
   kubectl get cm -n nginx-static
   ```
2. Edit ConfigMap:
   ```bash
   kubectl edit cm <configmap-name> -n nginx-static
   ```
   Example snippet:

   ```yaml
   data:
     nginx.conf: |
       ssl_protocols TLSv1.3;
   ```

   Remove any mention of `TLSv1.2`.

3. Restart Deployment:
   ```bash
   kubectl rollout restart deployment -n nginx-static
   ```
4. Get service IP:
   ```bash
   kubectl get svc nginx-service -n nginx-static
   ```
5. Update `/etc/hosts` on the client node:
   ```bash
   echo "<SERVICE-IP> ckaquestion.k8s.local" >> /etc/hosts
   ```
6. Verify TLS:
   ```bash
   curl -vk --tls-max 1.2 https://ckaquestion.k8s.local     # should FAIL
   curl -vk --tlsv1.3 https://ckaquestion.k8s.local         # should SUCCEED
   ```

---

### How to Use This File

- **Before the exam**: skim each question to understand typical scenarios (storage, networking, security, Helm, etc.).
- **During practice**: try to solve from the question statement only, then compare your solution with the example answers here.
- **After practice**: modify this README with your own notes, corrections, or cluster-specific details.

