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

## 3. Metrics Server

### Cài Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

### Lấy APIService liên quan đến metrics

```bash
kubectl get apiservice | grep metrics
```

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

## 7. ConfigMap

### Tạo ConfigMap từ file

```bash
kubectl create cm dem-heroes --from-file=heroes.txt
```

---

### ConfigMap manifest (`configmap.yml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dem-heroes
data:
  heroes.txt: |
    ironman
    spiderman
    thor
```

Apply:

```bash
kubectl apply -f configmap.yml
```

---

### Lấy ConfigMap

```bash
kubectl get configmaps
```

---

### Describe ConfigMap

```bash
kubectl describe cm dem-heroes
```

---

## 8. Logs

Kubernetes cung cấp nhiều cách để xem log, đặc biệt quan trọng khi Pod có **nhiều containers**.

---

### Xem logs của container cụ thể trong Pod

```bash
kubectl logs counter -c countby3
```

Giải thích:

* `counter` : tên Pod
* `-c countby3` : chỉ định container cần xem log

Dùng khi Pod có **nhiều containers**.

---

### Xem logs của tất cả containers trong Pod

```bash
kubectl logs counter --all-containers
```

Kết quả sẽ bao gồm log của **mọi container** trong Pod.

---

### Follow logs (real-time)

```bash
kubectl logs -f counter -c countby3
```

---

### Xem logs container đã crash trước đó

```bash
kubectl logs counter -c count --previous
```

Rất hữu ích khi debug **CrashLoopBackOff**.

---

### Best Practices khi debug logs

* Luôn xác định **Pod có bao nhiêu containers**
* Dùng `-c` để tránh nhầm log
* Khi gặp `CrashLoopBackOff`, luôn thử `--previous`
* Kết hợp với `kubectl describe pod <pod>` để xem Events

---

### Xem logs Pod

```bash
kubectl logs demopod
```

### Follow logs

```bash
kubectl logs -f demopod
```

### Logs container cụ thể

```bash
kubectl logs demopod -c nginx
```

---

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

# 1️⃣5️⃣ VÍ DỤ TỔNG HỢP TOÀN DIỆN (END-TO-END)

Phần này **kết nối toàn bộ kiến thức đã học** thành **một hệ thống Kubernetes hoàn chỉnh**, giúp bạn có cái nhìn tổng quan từ đầu đến cuối.

---

## 🎯 Mục tiêu hệ thống

Chúng ta xây dựng hệ thống gồm:

* **Frontend (nginx)**

  * Stateless
  * Có thể scale
  * Expose qua Service

* **Backend / DB (ví dụ giả lập)**

  * Không expose public
  * Chỉ frontend được phép truy cập

* **Security**

  * NetworkPolicy kiểm soát traffic

---

## 🧱 Kiến trúc tổng thể

```text
User
  │
  ▼
Service (nginx-service)
  │  Load Balance
  ▼
Deployment nginx (2 Pods)
  │  (app=nginx)
  ▼
Service nội bộ
  │
  ▼
DB Pods (role=db)
```

---

## 1️⃣ Namespace (tùy chọn nhưng khuyến khích)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

```bash
kubectl apply -f namespace.yml
```

---

## 2️⃣ Persistent Volume & PVC (cho DB)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/db
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  namespace: demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## 3️⃣ Deployment – Frontend (nginx)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: demo
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
          image: nginx:1.18
          ports:
            - containerPort: 80
```

---

## 4️⃣ Deployment – Backend / DB (ví dụ)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-deployment
  namespace: demo
  labels:
    role: db
spec:
  replicas: 1
  selector:
    matchLabels:
      role: db
  template:
    metadata:
      labels:
        role: db
    spec:
      containers:
        - name: db
          image: busybox
          command: ["/bin/sh", "-c", "sleep 3600"]
          volumeMounts:
            - name: db-storage
              mountPath: /data
      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: db-pvc
```

---

## 5️⃣ Services

### 5.1 Service cho nginx (public)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

---

### 5.2 Service cho DB (internal only)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
  namespace: demo
spec:
  type: ClusterIP
  selector:
    role: db
  ports:
    - port: 3306
      targetPort: 3306
```

---

## 6️⃣ NetworkPolicy – bảo mật hệ thống

### 6.1 Deny all DB traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-all
  namespace: demo
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
```

---

### 6.2 Cho phép nginx truy cập DB

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-nginx
  namespace: demo
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx
```

---

## 7️⃣ Luồng traffic cuối cùng

```text
User → nginx-service → nginx Pods → db-service → db Pod
```

* Service đảm nhiệm routing
* Labels quyết định ai là backend
* NetworkPolicy quyết định ai được phép nói chuyện

---

## 8️⃣ Checklist debug hệ thống

```bash
kubectl get pods -n demo --show-labels
kubectl get svc -n demo
kubectl get endpoints -n demo
kubectl describe networkpolicy -n demo
kubectl logs <pod>
```

---

## 🧠 Tổng kết cuối cùng

* Kubernetes là **hệ sinh thái các resource kết nối bằng labels**
* Deployment → Pod lifecycle
* Service → stable networking
* NetworkPolicy → zero-trust security
* PVC → data persistence

> Nếu bạn hiểu được ví dụ này, bạn đã **nắm được 70–80% nền tảng Kubernetes thực tế**.

---

## 11. Secret & Environment Variables

* Không dùng `hostPath` cho production
* Production dùng:

  * AWS EBS / EFS
  * GCP Persistent Disk
  * NFS / Ceph
* Mỗi DB nên có PVC riêng
* Backup data định kỳ

---

## 11. Secret & Environment Variables

Phần này minh họa cách **inject Secret vào Pod thông qua biến môi trường** và kiểm tra Secret có hoạt động hay không.

---

### Pod sử dụng Secret (`mysql-pod.yml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-locked
spec:
  containers:
    - name: mysql
      image: mysql:8.0
      env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
```

Apply:

```bash
kubectl apply -f mysql-pod.yml
```

---

### Secret manifest (`secret.yml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: kubernetes.io/basic-auth
stringData:
  password: alta3
```

Apply:

```bash
kubectl apply -f secret.yml
```

---

### Kiểm tra Pod chạy thành công

```bash
kubectl get pods
```

Expected:

```
mysql-locked   1/1   Running
```

---

### Exec vào Pod và kiểm tra biến môi trường

```bash
kubectl exec -it mysql-locked -- bash
```

Bên trong container:

```bash
echo $MYSQL_ROOT_PASSWORD
```

Expected output:

```
alta3
```

---

### Ghi chú bảo mật (Important)

* Không commit file `secret.yml` thật lên Git
* `stringData` chỉ tiện cho demo / học tập
* Production nên dùng:

  * External Secret Manager (AWS Secrets Manager, Vault, GCP Secret Manager)
  * Hoặc CI/CD inject secret

---

## Tổng kết

* Pod là ephemeral & stateless
* YAML giúp tái sử dụng và quản lý hạ tầng
* `kubectl run` phù hợp test nhanh
* ConfigMap dùng cho config, **không** dùng cho secrets

---

🚀 Happy Kubernetes hacking!

