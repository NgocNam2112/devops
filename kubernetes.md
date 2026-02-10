# Kubernetes Commands & YAML Samples

Tài liệu này tổng hợp **các lệnh Kubernetes cơ bản** và **ví dụ file YAML** để dễ học, dễ nhớ và dễ dùng lại.

---

## 1. Pods

### Tạo Pod từ manifest (`podmanifest.yml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f podmanifest.yml
```

---

### Lấy tất cả Pod trong namespace `default`

```bash
kubectl get pods
```

---

### Describe Pod

```bash
kubectl describe pod nginx
```

> `kubectl describe RESOURCE RESOURCE_NAME` hiển thị thông tin chi tiết của resource.

---

### Delete Pod

```bash
kubectl delete pod nginx
# hoặc
kubectl delete -f podmanifest.yml
```

---

## 2. Namespace

### Lấy tất cả namespace

```bash
kubectl get namespace
```

---

### Lấy Pod trong namespace cụ thể (`kube-system`)

```bash
kubectl get pods -n kube-system
```

---

### Tạo namespace mới

```bash
kubectl create ns demo
```

---

### Namespace manifest (`namespace.yml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

Apply:

```bash
kubectl apply -f namespace.yml
```

---

## 3. Metrics Server (Updated - Production Knowledge)

### Metrics Server la gi?

Metrics Server la component trong Kubernetes dung de thu thap CPU va Memory usage cua Pods va Nodes.

Metrics Server lay du lieu tu kubelet cua moi node va cung cap qua Kubernetes API.

Kien truc:

```text
Pods / Nodes
      |
      v
   Kubelet
      |
      v
Metrics Server
      |
      v
Kubernetes API
      |
      v
kubectl top / HPA
```

### Neu khong co Metrics Server

Các lệnh sau khong hoat dong:

```bash
kubectl top pod
kubectl top nodes
```

Error thuong gap:

```text
error: Metrics API not available
```

### Tai sao production cluster can Metrics Server?

1️⃣ Monitoring resource usage

DevOps can biet:

* Pod nao dung CPU cao
* Pod nao dung Memory cao
* Node nao dang qua tai

Vi du:

```bash
kubectl top pods
```

Output:

```text
NAME              CPU(cores)   MEMORY(bytes)
api-gateway       120m         210Mi
user-service      80m          140Mi
```

2️⃣ Horizontal Pod Autoscaler (HPA)

Autoscaling phu thuoc Metrics Server.

Vi du:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Flow:

```text
Traffic tang
   ↓
CPU usage tang
   ↓
Metrics Server detect
   ↓
HPA scale Pods
```

3️⃣ Capacity planning

Metrics giup:

* xac dinh Pod can bao nhieu CPU
* dieu chinh resource limits
* xac dinh khi nao can them node

### Cai Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Kiem tra Metrics Server

```bash
kubectl get pods -n kube-system
```

Expected:

```text
metrics-server-xxxxx Running
```

### Kiem tra metrics

```bash
kubectl top nodes
kubectl top pods
```

### Metrics Server vs Prometheus

| Tool | Purpose |
| ---- | ------- |
| Metrics Server | real-time CPU / Memory |
| Prometheus | metrics history |
| Grafana | dashboard |

Production stack:

* Metrics Server -> Autoscaling
* Prometheus -> Monitoring
* Grafana -> Visualization

---

## 4. Chạy Pod không cần manifest

```bash
kubectl run demopod --image=nginx
```

Kiểm tra:

```bash
kubectl get pods
```

---

## 5. Port Forward

### Forward port từ local vào Pod

```bash
kubectl port-forward demopod 2224:80
```

Test:

```bash
curl http://localhost:2224
```

> Port mặc định của NGINX là `80`.

---

## 6. Exec vào Pod

```bash
kubectl exec -it demopod -- sh
```

⚠️ **Lưu ý quan trọng**:

> Containers là **stateless**. Mọi thay đổi bên trong container sẽ **mất hoàn toàn** khi Pod bị restart hoặc delete.

---

## 6. Resource Limits & Requests

Kubernetes cho phép giới hạn và yêu cầu tài nguyên (CPU, Memory) cho mỗi container thông qua `resources` trong `spec`.

### Ý nghĩa

* **requests**: tài nguyên *tối thiểu* Pod cần để được schedule
* **limits**: tài nguyên *tối đa* container được phép sử dụng

Nếu container vượt quá:

* **CPU limit** → bị throttling
* **Memory limit** → Pod bị OOMKilled

---

### Ví dụ Pod với Resource Limits (`pod-resources.yml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-resources
spec:
  containers:
    - name: nginx
      image: nginx:latest
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f pod-resources.yml
```

---

### Kiểm tra resource usage

```bash
kubectl top pod nginx-resources
kubectl describe pod nginx-resources
```

---

### Đơn vị tài nguyên phổ biến

| Resource    | Ý nghĩa  |
| ----------- | -------- |
| `1000m` CPU | 1 core   |
| `500m` CPU  | 0.5 core |
| `128Mi`     | 128 MB   |
| `1Gi`       | 1024 MB  |

---

### Best Practices

* Luôn set **requests & limits** cho production
* Tránh để limits quá thấp → dễ OOM
* Tránh để limits quá cao → chiếm tài nguyên node
* Dùng `kubectl top` để tinh chỉnh dần

---

## 7. ConfigMap (Updated - Production Usage)

### 1. What ConfigMap Is (Real Meaning)

A ConfigMap is used to store non-sensitive configuration data.

Examples:

* application environment variables
* feature flags
* config files
* service URLs
* log levels
* application settings

Example real backend config:

```text
NODE_ENV=production
DB_HOST=mysql
REDIS_HOST=redis
RABBITMQ_URL=amqp://rabbitmq
LOG_LEVEL=info
```

These should not be hardcoded in the container image.

### 2. Real Architecture Example

Typical production setup:

```text
Docker Image (NestJS App)
        |
        v
Kubernetes Deployment
        |
        v
   ConfigMap
        |
        v
Environment variables inside container
```

Example:

```text
NestJS Container
   |
   |---- process.env.DB_HOST
   |---- process.env.REDIS_HOST
   |---- process.env.RABBITMQ_URL
```

All values come from ConfigMap.

### 3. Example Real ConfigMap

`configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  NODE_ENV: "production"
  DB_HOST: "mysql-service"
  REDIS_HOST: "redis-service"
  RABBITMQ_URL: "amqp://rabbitmq-service"
  LOG_LEVEL: "info"
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

Check:

```bash
kubectl get configmap
```

### 4. Using ConfigMap in Deployment (Most Common)

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: my-backend:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: backend-config
```

This loads all keys from ConfigMap into environment variables.

Inside NestJS:

* `process.env.DB_HOST`
* `process.env.REDIS_HOST`

### 5. Using Specific Keys (More Secure)

Sometimes you want only specific variables.

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: backend-config
        key: DB_HOST
```

### 6. Using ConfigMap as File (Very Common)

Many apps require config files, not environment variables.

Example `config.yaml` in ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    logLevel: info
    maxConnections: 100
    featureFlags:
      enableLogin: true
```

Mount as volume:

```yaml
containers:
  - name: backend
    image: my-backend
    volumeMounts:
      - name: config-volume
        mountPath: /app/config

volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Inside container:

`/app/config/config.yaml`

### 7. Real Production Example (Microservices)

Example Nx NestJS microservices architecture.

Services:

* `auth-service`
* `checkout-service`
* `notification-service`

Each service has its own ConfigMap.

`auth-config`:

```yaml
data:
  JWT_SECRET: "my-secret"
  TOKEN_EXPIRE: "3600"
  REDIS_HOST: "redis-service"
```

`checkout-config`:

```yaml
data:
  PAYMENT_PROVIDER: "stripe"
  RABBITMQ_URL: "amqp://rabbitmq-service"
```

`notification-config`:

```yaml
data:
  SMTP_HOST: "smtp.gmail.com"
  SMTP_PORT: "587"
```

### 8. Updating ConfigMap in Production

Update config:

```bash
kubectl edit configmap backend-config
```

Pods will not automatically reload config.

You must restart deployment:

```bash
kubectl rollout restart deployment backend
```

### 9. Best Practice in Real Systems

1️⃣ ConfigMap for non-secret data

* `DB_HOST`
* `REDIS_HOST`
* `LOG_LEVEL`
* `SERVICE_URL`

2️⃣ Secret for sensitive data

* `DB_PASSWORD`
* `JWT_SECRET`
* `API_KEYS`

Use `Secret` + `ConfigMap` together.

3️⃣ Versioned ConfigMap

Instead of updating one name repeatedly, use versioning:

* `backend-config-v1`
* `backend-config-v2`

Then update deployment.

4️⃣ Use Helm for ConfigMap

In production Kubernetes systems, ConfigMaps are commonly managed by Helm charts.

Example `values.yaml`:

```yaml
config:
  redisHost: redis
  logLevel: info
```

### 10. Real Flow in Production

Production CI/CD pipeline:

```text
Developer changes config
        |
        v
Git push
        |
        v
CI/CD pipeline
        |
        v
Update ConfigMap
        |
        v
Rollout Deployment
        |
        v
Pods restart with new config
```

### 11. Debugging ConfigMap

Check ConfigMap:

```bash
kubectl get configmap backend-config -o yaml
```

Check inside container:

```bash
kubectl exec -it pod-name -- env
```

or

```bash
kubectl exec -it pod-name -- cat /app/config/config.yaml
```

### 12. Most Common Interview Answer (System Design)

When asked: How do you manage configuration in Kubernetes?

Answer:

* Application configuration is stored in ConfigMaps
* Secrets store sensitive credentials
* Deployments inject these values as environment variables or mounted configuration files
* CI/CD updates ConfigMaps and triggers rolling deployment

### 13. Real World Example (Netflix Style)

Large companies store the following in ConfigMaps:

* feature flags
* rate limits
* service endpoints
* logging configuration

Production structure often follows Kustomize-style layout:

```text
k8s/
 ├── base
 │    ├── deployment.yaml
 │    ├── service.yaml
 │    ├── configmap.yaml
 │    └── secret.yaml
 │
 └── overlays
      ├── dev
      ├── staging
      └── production
```

---

## 8️⃣ Logs (Ví dụ CỤ THỂ từ Pod → Deployment → Service)

Phần này là **ví dụ thực hành end‑to‑end**, đi từ **tạo từng loại Pod** → **xem logs** → **hiểu sự khác nhau khi scale và expose Service**.

---

# 🧪 VÍ DỤ 1: Pod đơn (Single Pod)

### 1️⃣ Tạo Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-single-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - |
          i=0;
          while true; do
            echo "[Single Pod] count=$i $(date)";
            i=$((i+1));
            sleep 2;
          done
```

Apply:

```bash
kubectl apply -f single-pod.yml
```

### 2️⃣ Xem logs

```bash
kubectl logs log-single-pod
kubectl logs -f log-single-pod
```

➡️ **Dễ nhất**, 1 Pod = 1 luồng log.

---

# 🧪 VÍ DỤ 2: Pod nhiều container

### 1️⃣ Tạo Pod multi-container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-multi-pod
spec:
  containers:
    - name: app-1
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - while true; do echo "[app-1] $(date)"; sleep 3; done
    - name: app-2
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - while true; do echo "[app-2] $(date)"; sleep 5; done
```

### 2️⃣ Xem logs

```bash
kubectl logs log-multi-pod -c app-1
kubectl logs log-multi-pod -c app-2
kubectl logs log-multi-pod --all-containers
```

➡️ **Phải chỉ rõ container**, nếu không Kubernetes không biết lấy log nào.

---

# 🧪 VÍ DỤ 3: Deployment (nhiều Pod)

### 1️⃣ Tạo Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: log-demo
  template:
    metadata:
      labels:
        app: log-demo
    spec:
      containers:
        - name: app
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              echo "Pod: $(hostname) started";
              while true; do
                echo "[$(hostname)] $(date)";
                sleep 4;
              done
```

Apply:

```bash
kubectl apply -f log-deploy.yml
```

### 2️⃣ Xem logs từng Pod

```bash
kubectl get pods -l app=log-demo
kubectl logs <pod-name>
```

### 3️⃣ Xem logs TOÀN BỘ Deployment

```bash
kubectl logs -l app=log-demo
```

➡️ Đây là **cách phổ biến nhất khi debug Service**.

---

# 🧪 VÍ DỤ 4: Deployment + Service + Logs (load balancing)

### 1️⃣ Service cho Deployment

```yaml
apiVersion: v1
kind: Service
metadata:
  name: log-service
spec:
  selector:
    app: log-demo
  ports:
    - port: 80
      targetPort: 80
```

### 2️⃣ Gọi Service (port-forward)

```bash
kubectl port-forward svc/log-service 8080:80
curl localhost:8080
```

### 3️⃣ Quan sát logs

```bash
kubectl logs -l app=log-demo
```

➡️ Mỗi request có thể vào **Pod khác nhau**.

---

# 🧪 VÍ DỤ 5: Pod restart & logs cũ

### 1️⃣ Pod crash

```yaml
args:
  - |
    echo "starting";
    sleep 5;
    exit 1
```

### 2️⃣ Xem logs

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

➡️ `--previous` cực kỳ quan trọng khi debug CrashLoopBackOff.

---

# 🧠 TỔNG KẾT LOGS

| Trường hợp      | Cách xem logs    |
| --------------- | ---------------- |
| Pod đơn         | kubectl logs pod |
| Multi container | -c container     |
| Deployment      | -l label         |
| Service         | logs backend Pod |
| Pod crash       | --previous       |

> **Service KHÔNG có logs** – logs luôn nằm ở Pod / container.

1. Pod có log không?
2. Log có cập nhật khi request vào?
3. Có Pod nào không nhận request?
4. Pod nào lỗi liên tục?

---

## 🔑 Ghi nhớ

* Logs = công cụ debug số 1
* Kết hợp logs + labels để debug Service
* Logs giúp bạn **thấy được load balancing thực sự**

## 9. Labels & Selectors

Labels là **key-value metadata** dùng để phân loại, lọc và quản lý resource trong Kubernetes.

---

### Xem labels của Pod

```bash
kubectl get pod demo-pod --show-labels
```

---

### Gán label cho Pod

```bash
kubectl label pod demo-pod awesome=training
```

Nếu label đã tồn tại và muốn ghi đè:

```bash
kubectl label pod demo-pod awesome=training --overwrite
```

---

### Xóa label khỏi Pod

```bash
kubectl label pod demo-pod awesome-
```

---

### Hiển thị Pod kèm theo một label cụ thể

```bash
kubectl get pods -L awesome
```

> `-L` giúp hiển thị thêm cột label trong output.

---

### Tạo Pod mới và gán label

```bash
kubectl run another-pod --image=nginx
kubectl run another-pod2 --image=nginx
kubectl label pod another-pod alta3=awesome
```

---

### Lọc Pod bằng label selector

```bash
kubectl get pods --selector=alta3=awesome
# hoặc
kubectl get pods -l alta3=awesome
```

---

### Best Practices với Labels

* Dùng label cho **phân loại**, không dùng để lưu dữ liệu quan trọng
* Đặt tên label có ý nghĩa (vd: `app`, `env`, `version`)
* Labels là nền tảng cho **Service, Deployment, ReplicaSet selectors**

---

## 10. Deployment (Very Detailed)

Deployment là **resource quan trọng nhất trong Kubernetes** để chạy application lâu dài, có khả năng **scale, self-healing và rolling update**.

---

## Deployment là gì?

Deployment quản lý:

* ReplicaSet (số lượng Pod mong muốn)
* Vòng đời Pod
* Rolling update / rollback

> ❗ Trong thực tế **không deploy Pod trực tiếp**, mà luôn deploy thông qua **Deployment**.

---

## Ví dụ Deployment (`deployment.yml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

Apply Deployment:

```bash
kubectl apply -f deployment.yml
```

---

## Giải thích chi tiết từng phần

### 1️⃣ apiVersion & kind

```yaml
apiVersion: apps/v1
kind: Deployment
```

* `apps/v1`: API ổn định cho Deployment
* `Deployment`: loại resource

---

### 2️⃣ metadata

```yaml
metadata:
  name: nginx-deployment
  labels:
    app: nginx
```

* `name`: tên Deployment
* `labels`: metadata dùng cho quản lý & selector

---

### 3️⃣ spec.replicas

```yaml
replicas: 2
```

* Kubernetes **luôn đảm bảo có đúng 2 Pods đang chạy**
* Nếu 1 Pod chết → tự tạo Pod mới

---

### 4️⃣ spec.selector (CỰC KỲ QUAN TRỌNG)

```yaml
selector:
  matchLabels:
    app: nginx
```

* Deployment **quản lý các Pod có label khớp selector**
* Selector **immutable** (không đổi được sau khi tạo)

> ❗ Selector **phải khớp 100%** với labels trong template

---

### 5️⃣ spec.template (Pod template)

```yaml
template:
  metadata:
    labels:
      app: nginx
```

* Đây là **khuôn mẫu để tạo Pod**
* Mỗi Pod sinh ra đều có label `app=nginx`

---

### 6️⃣ Containers

```yaml
containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
      - containerPort: 80
```

* `image`: image chạy trong Pod
* `containerPort`: port ứng dụng (chỉ mang tính mô tả)

---

## Quan hệ giữa Deployment → ReplicaSet → Pod

```text
Deployment
   └── ReplicaSet
          ├── Pod
          ├── Pod
```

* Deployment tạo ReplicaSet
* ReplicaSet tạo & quản lý Pods

---

## Kiểm tra Deployment

```bash
kubectl get deployment
kubectl get replicaset
kubectl get pods
```

Xem chi tiết:

```bash
kubectl describe deployment nginx-deployment
```

---

## Scale Deployment

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Kubernetes sẽ:

* Tạo thêm Pod (nếu scale up)
* Xóa Pod (nếu scale down)

---

## Rolling Update

Thay đổi image:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16
```

Theo dõi rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

---

## Rollback Deployment

Xem lịch sử:

```bash
kubectl rollout history deployment/nginx-deployment
```

Rollback:

```bash
kubectl rollout undo deployment/nginx-deployment
```

---

## Best Practices (Rất quan trọng)

* Không dùng `latest` tag cho image
* Luôn set `replicas >= 2` cho production
* Selector **không được đổi** sau khi tạo
* Deployment phù hợp cho **stateless app**

---

## Tóm tắt nhanh

* Deployment = chuẩn để chạy app
* Tự healing, tự scale
* Quản lý ReplicaSet & Pods
* Hỗ trợ rolling update & rollback

---

## 11. Persistent Volume (PV) & Persistent Volume Claim (PVC)

Persistent Volume dùng để **lưu trữ dữ liệu lâu dài**, giúp dữ liệu **không bị mất khi Pod bị restart / recreate**.

> ❗ Deployment, Pod là *ephemeral* → **storage phải là Persistent** nếu muốn giữ data.

---

## 1️⃣ Khái niệm tổng quan

```text
[ Pod ]
   │
   ▼
[ PVC ]  (request storage)
   │
   ▼
[ PV ]   (actual storage)
```

* **PV**: tài nguyên storage trong cluster
* **PVC**: yêu cầu storage từ Pod
* **Pod không mount PV trực tiếp**, mà mount qua PVC

---

## 2️⃣ Persistent Volume (PV)

### Ví dụ PV (`pv.yml`)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/nginx
```

### Giải thích

* `capacity.storage`: dung lượng
* `ReadWriteOnce`: 1 node ghi/đọc
* `Retain`: giữ data khi PVC bị xóa
* `hostPath`: **chỉ dùng cho local dev (Colima / Minikube)**

---

## 3️⃣ Persistent Volume Claim (PVC)

### Ví dụ PVC (`pvc.yml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

PVC sẽ được bind với PV có:

* Cùng `accessModes`
* Dung lượng ≥ request

---

## 4️⃣ Deployment sử dụng PVC (Cập nhật từ deployment trước)

### Deployment + Volume (`deployment.yml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: nginx-storage
          persistentVolumeClaim:
            claimName: nginx-pvc
```

### Giải thích

* `volumeMounts.mountPath`: nơi data được mount trong container
* `/usr/share/nginx/html`: thư mục web root của nginx
* `volumes.persistentVolumeClaim`: liên kết PVC

---

## 5️⃣ Thứ tự apply (RẤT QUAN TRỌNG)

```bash
kubectl apply -f pv.yml
kubectl apply -f pvc.yml
kubectl apply -f deployment.yml
```

---

## 6️⃣ Kiểm tra trạng thái

```bash
kubectl get pv
kubectl get pvc
kubectl get pods
```

Expected:

```
PV    STATUS   Bound
PVC   STATUS   Bound
```

---

## 7️⃣ Test persistence (thực tế)

```bash
kubectl exec -it <nginx-pod> -- sh
cd /usr/share/nginx/html
echo "Hello Persistent Volume" > index.html
```

Xóa Pod:

```bash
kubectl delete pod <nginx-pod>
```

Pod mới tạo lại → truy cập nginx → **data vẫn còn** ✅

---

## 8️⃣ Reclaim Policy (Rất quan trọng)

| Policy  | Ý nghĩa                   |
| ------- | ------------------------- |
| Retain  | Giữ data (manual cleanup) |
| Delete  | Xóa data khi PVC bị xóa   |
| Recycle | Deprecated                |

---

## 9️⃣ PV vs PVC vs Volume

| Thành phần | Vai trò         |
| ---------- | --------------- |
| PV         | Storage vật lý  |
| PVC        | Yêu cầu storage |
| Volume     | Gắn vào Pod     |

---

## 10️⃣ Best Practices

* Không dùng `hostPath` cho production
* Production dùng:

  * AWS EBS / EFS
  * GCP Persistent Disk
  * NFS / Ceph
* Mỗi DB nên có PVC riêng
* Backup data định kỳ

---

## 11️⃣ Pod sử dụng PVC trực tiếp (MountPath `/data`)

Phần này minh họa **Pod (không qua Deployment)** mount PVC trực tiếp vào container – đúng như ví dụ bạn vừa làm trong video.

---

### Pod + PVC (`storage.yml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-pv
spec:
  containers:
    - name: nginx-with-pv
      image: nginx:1.18.0
      ports:
        - name: http
          containerPort: 80
          protocol: TCP
      volumeMounts:
        - name: nginx-pv-storage
          mountPath: /data
  volumes:
    - name: nginx-pv-storage
      persistentVolumeClaim:
        claimName: nginx-pvc
```

---

### Giải thích quan trọng

* `mountPath: /data`

  * Thư mục **bên trong container**
  * Mọi file ghi vào `/data` sẽ được lưu trong PV

* `claimName: nginx-pvc`

  * Pod **không biết PV nào**
  * Kubernetes tự bind PVC → PV phù hợp

---

## 12️⃣ Kiểm tra PV & PVC real-time

```bash
kubectl get pv,pvc
```

Hoặc theo dõi liên tục:

```bash
watch kubectl get pv,pvc
```

Expected trạng thái:

```text
PV   STATUS   Bound
PVC  STATUS   Bound
```

---

## 13️⃣ Apply storage & Pod

```bash
kubectl apply -f storage.yml
```

Output mong đợi:

```text
persistentvolume/... unchanged
persistentvolumeclaim/... unchanged
pod/nginx-with-pv created
```

---

## 14️⃣ Test persistence nhanh

```bash
kubectl exec -it nginx-with-pv -- sh
cd /data
echo "Hello from PV" > test.txt
ls
```

Xóa Pod:

```bash
kubectl delete pod nginx-with-pv
```

Tạo lại Pod → file `test.txt` **vẫn còn** ✅

---

## 15️⃣ Ghi nhớ (RẤT QUAN TRỌNG)

* `mountPath` = nơi **container thấy data**
* PV/PVC tồn tại **độc lập Pod**
* Xóa Pod ≠ mất data
* Xóa PVC → tùy `ReclaimPolicy`

---

## 11. Service & Networking trong Kubernetes (Tổng hợp & Áp dụng toàn diện)

Phần này **tổng hợp đầy đủ kiến thức về Service & Network** và **áp dụng trực tiếp** toàn bộ những gì bạn đã học trước đó:

* Pod
* Deployment
* Labels & Selectors
* Logs
* Persistent Volume
* NetworkPolicy

Mục tiêu: hiểu **luồng giao tiếp thật sự** trong Kubernetes – từ Pod → Service → NetworkPolicy.

---

## 1️⃣ Bài toán thực tế

Giả sử bạn có:

* Một **Deployment nginx** (stateless)
* Nhiều Pod được scale
* IP Pod thay đổi liên tục
* Cần:

  * Truy cập ổn định
  * Load balancing
  * Kiểm soát ai được phép truy cập

➡️ **Service + NetworkPolicy** giải quyết bài toán này.

---

## 2️⃣ Deployment (đã học – dùng lại)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

Sau khi apply:

```bash
kubectl get pods --show-labels
```

➡️ Các Pod có label:

```
app=nginx
```

---

## 3️⃣ Service – kết nối ổn định tới Deployment

### 3.1 Vì sao không truy cập Pod trực tiếp?

* Pod có thể bị delete / recreate
* IP Pod thay đổi

➡️ **Service** cung cấp:

* IP ổn định (ClusterIP)
* DNS name
* Load balancing

---

### 3.2 Expose Deployment thành Service

```bash
kubectl expose deployment nginx-deployment --port=80 --target-port=80
```

Kiểm tra:

```bash
kubectl get svc
```

Kết quả:

```
nginx-deployment   ClusterIP   172.16.x.x   80/TCP
```

---

### 3.3 Service YAML (gắn với label đã học)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

🔑 **Selector = bridge giữa Service và Pod**

---

## 4️⃣ Service hoạt động như thế nào?

```text
Client
  ↓
Service (ClusterIP)
  ↓ (load balance)
Pod #1 (app=nginx)
Pod #2 (app=nginx)
```

Kiểm tra backend thực sự:

```bash
kubectl get endpoints nginx-service
```

➡️ Endpoints = danh sách Pod đang phục vụ traffic.

---

## 5️⃣ Truy cập Service (debug & test)

### 5.1 Port-forward (local debug)

```bash
kubectl port-forward svc/nginx-service 8080:80
curl http://localhost:8080
```

⚠️ Port-forward **không phải expose thật**, chỉ dùng debug.

---

## 6️⃣ Service Types (khi nào dùng?)

| Type         | Khi dùng                 |
| ------------ | ------------------------ |
| ClusterIP    | Nội bộ cluster (default) |
| NodePort     | Test nhanh / lab         |
| LoadBalancer | Cloud production         |
| ExternalName | DNS alias                |

---

## 7️⃣ NetworkPolicy – bảo mật network cho Pod

Nếu Service là **routing**, thì NetworkPolicy là **firewall**.

> Mặc định Kubernetes: **ALLOW ALL** traffic.

---

## 8️⃣ NetworkPolicy áp dụng cho Pod cụ thể

Ví dụ: chỉ kiểm soát Pod có label `role=db`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
```

➡️ Các Pod **không có label `role=db` không bị ảnh hưởng**.

---

## 9️⃣ Ingress – ai được phép truy cập DB?

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: backend
```

➡️ Chỉ Pod `app=backend` mới được phép truy cập Pod DB.

---

## 🔟 Egress – DB được phép gọi ra đâu?

```yaml
egress:
  - to:
      - podSelector:
          matchLabels:
            role: backend
```

➡️ DB chỉ được phép gọi backend.

---

## 1️⃣1️⃣ Deny All – chiến lược production

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

➡️ Chặn toàn bộ traffic → mở whitelist từng phần.

---

## 1️⃣2️⃣ Service vs NetworkPolicy (so sánh cốt lõi)

| Service                  | NetworkPolicy             |
| ------------------------ | ------------------------- |
| Routing / Load balancing | Security / Access control |
| Dựa trên labels          | Dựa trên labels           |
| Layer 4                  | Layer 3–4                 |
| Luôn hoạt động           | Phụ thuộc CNI             |

---

## 1️⃣3️⃣ Những lỗi rất hay gặp

* Service không route → **selector không khớp label**
* Có Service nhưng không truy cập được → **NetworkPolicy chặn**
* Không thấy tác dụng NetworkPolicy → **CNI không hỗ trợ**

---

## 1️⃣4️⃣ Ghi nhớ quan trọng

* Labels là **nền tảng cốt lõi** cho Deployment, Service, NetworkPolicy
* Service giải quyết **routing & load balancing**, không phải security
* NetworkPolicy giải quyết **security**, không phải expose
* Debug network luôn theo thứ tự:

  1. Pod chạy chưa?
  2. Labels đúng chưa?
  3. Service selector khớp chưa?
  4. Endpoints có Pod không?
  5. NetworkPolicy có chặn không?

---

# 1️⃣5️⃣ VÍ DỤ TỔNG HỢP TOÀN DIỆN (END-TO-END – REAL WORLD)

> 🎯 Mục tiêu phần này: **gom TOÀN BỘ kiến thức từ đầu tài liệu** thành **một hệ thống thực tế đúng kiểu production thu nhỏ**.
> Backend được mô phỏng bằng **microservice NestJS (Nx monorepo)** – rất sát thực tế doanh nghiệp.

---

## 🧠 Bài toán thực tế

Chúng ta xây dựng một hệ thống gồm:

* **Frontend**: Client (browser / curl)
* **API Gateway**: NestJS (Nx) – public entry
* **Backend Microservice**: NestJS (Nx)
* **Database**: giả lập (busybox / db pod)
* **Infrastructure**:

  * Deployment
  * Service (ClusterIP + NodePort)
  * Labels & Selectors
  * Logs
  * Persistent Volume
  * NetworkPolicy

---

## 🧱 Kiến trúc tổng thể

```text
Client
  │
  │  NODE_IP:31234
  ▼
NodePort Service (api-gateway)
  │
  ▼
Deployment: api-gateway (NestJS)
  │  ClusterIP Service
  ▼
Deployment: user-service (NestJS)
  │
  ▼
DB Pod + PVC
```

---

## 📦 1️⃣ Microservice structure (Nx – tư duy)

> **Không build Nx thật**, chỉ mô phỏng đúng cách deploy.

```text
nx-monorepo/
├─ apps/
│  ├─ api-gateway (NestJS)
│  └─ user-service (NestJS)
```

* `api-gateway`: nhận request từ client
* `user-service`: xử lý nghiệp vụ

---

## 🐳 2️⃣ Container giả lập NestJS (log-based)

Thay vì code thật, dùng image mô phỏng log:

```bash
echo "API-GATEWAY handled request";
echo "USER-SERVICE processed request";
```

➡️ Mục tiêu là **hiểu Kubernetes**, không phải code NestJS.

---

## 🚀 3️⃣ Deployment – API Gateway (public)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    app: api-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: api
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "[API-GATEWAY $(hostname)] $(date)";
                sleep 5;
              done
          ports:
            - containerPort: 3000
```

---

## 🌐 4️⃣ Service – NodePort (public entry)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-nodeport
spec:
  type: NodePort
  selector:
    app: api-gateway
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 31234
```

➡️ Client gọi:

```bash
curl NODE_IP:31234
```

---

## 🧩 5️⃣ Deployment – User Service (internal)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "[USER-SERVICE $(hostname)] $(date)";
                sleep 6;
              done
          ports:
            - containerPort: 3001
```

---

## 🔗 6️⃣ Service – ClusterIP (internal only)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - port: 3001
      targetPort: 3001
```

➡️ `api-gateway` gọi `user-service` qua DNS:

```text
http://user-service:3001
```

---

## 💾 7️⃣ Database Pod + Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    role: db
spec:
  containers:
    - name: db
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - sleep 3600
      volumeMounts:
        - name: db-storage
          mountPath: /data
  volumes:
    - name: db-storage
      persistentVolumeClaim:
        claimName: db-pvc
```

---

## 🔐 8️⃣ NetworkPolicy – Zero Trust

### 8.1 Deny all DB traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-all
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
```

### 8.2 Allow user-service → DB

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-user
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: user-service
```

---

## 📜 9️⃣ Logs – Debug toàn hệ thống

```bash
kubectl logs -l app=api-gateway
kubectl logs -l app=user-service
kubectl logs db-pod
```

➡️ Bạn thấy:

* Request vào API Gateway
* Gateway gọi User Service
* DB tồn tại data

---

## 🔎 🔟 Checklist debug thực tế

1. Pod running?
2. Labels đúng?
3. Service selector match?
4. Endpoints có backend?
5. NodePort mở đúng?
6. NetworkPolicy block?
7. Logs phản ánh đúng flow?

---

## 🧠 TỔNG KẾT CUỐI CÙNG

| Thành phần    | Vai trò             |
| ------------- | ------------------- |
| Deployment    | Lifecycle & scaling |
| Service       | Networking ổn định  |
| NodePort      | Public entry        |
| kube-proxy    | Load balancing      |
| Logs          | Debug sự thật       |
| PVC           | Data persistence    |
| NetworkPolicy | Security            |

> ✅ Nếu bạn hiểu và tự triển khai được ví dụ này, bạn đã **nắm vững nền tảng Kubernetes thực tế trong môi trường microservices**.