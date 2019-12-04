**Mô hình LAB**

|Server|IP|
|------|--|
|master1|10.1.38.128|
|master2|10.1.38.111|
|master3|10.1.38.149|
|worker1|10.1.38.144|
|worker2|10.1.38.146|
|mysql|10.1.38.147|
|NFS|10.1.38.129|

## 1. Thực hiện trên NFS
**Tạo thư mục chia sẻ trên NFS**
```sh
mkdir -p /data/wso2/worker
mkdir -p /data/wso2/dashboard
mkdir -p /data/wso2/apim
```
## 2. Thực hiện trên Node Master
Tạo 3 file cho 3 serice: `am-analytics-worker.yaml`, `api-manager`, `am-analytics-dashboard`.**
### 2.1 Tạo file `api-manager.yaml` với nội dung gồm các phần như sau:
**Tạo 2 `PersistentVolume` và 2 `PersistentVolumeClaim` tương ứng như sau:**
```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-config
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/config"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-config
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-artifact
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-artifact
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/artifact"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-artifact
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-artifact
```
Cần lưu ý 1 số trường như sau: 
- `spec.nfs`: IP của NFS server và đường dẫn. Thay đổi với Topologi của mình cho phù hợp, ở dây là `10.1.38.129` và path: `/data/wso2/apim/artifact`
- Các trường khác có thể giữ nguyên hoặc thay đổi nhưng cần chú ý label cho khớp.

**Tạo Deployment với nội dung như sau:**
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-manager-deployment
  namespace: wso2
  labels:
    app: api-manager
    role: api-manager
spec:
  selector:
    matchLabels:
      app: api-manager
      role: api-manager
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: api-manager
        role: api-manager
    spec:
      containers:
      - name: api-manager
        image: wso2/wso2am:3.0.0
        ports:
        - containerPort: 9763
          name: port1
        - containerPort: 9443
          name: port2
        - containerPort: 8280
          name: port3
        - containerPort: 8243
          name: port4
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: api-manager-config
        - mountPath: /home/wso2carbon/wso2-artifact-volume
          name: api-manager-artifact
      volumes:
      - name: api-manager-config
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-config
      - name: api-manager-artifact
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-artifact
```
**Tạo file Service với nội dung sau:**
```sh
apiVersion: v1
kind: Service
metadata:
  name: api-manager
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9763
    name: port1
    nodePort: 31124
  - port: 9443
    name: port2
    nodePort: 31125
  - port: 8280
    name: port3
    nodePort: 31126
  - port: 8243
    name: port4
    nodePort: 31127
  selector:
    app: api-manager
    role: api-manager
```
Tất cả các File được gom lại thành 1 file `api-manager.yaml` như sau:
```sh
cat << EOF > api-manager.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-config
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/config"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-config
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-artifact
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-artifact
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/artifact"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-artifact
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-artifact
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-manager-deployment
  namespace: wso2
  labels:
    app: api-manager
    role: api-manager
spec:
  selector:
    matchLabels:
      app: api-manager
      role: api-manager
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: api-manager
        role: api-manager
    spec:
      containers:
      - name: api-manager
        image: wso2/wso2am:3.0.0
        ports:
        - containerPort: 9763
          name: port1
        - containerPort: 9443
          name: port2
        - containerPort: 8280
          name: port3
        - containerPort: 8243
          name: port4
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: api-manager-config
        - mountPath: /home/wso2carbon/wso2-artifact-volume
          name: api-manager-artifact
      volumes:
      - name: api-manager-config
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-config
      - name: api-manager-artifact
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-artifact
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-api-manager
  namespace: wso2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-manager-deployment
  minReplicas: 1
  maxReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-manager
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9763
    name: port1
    nodePort: 31124
  - port: 9443
    name: port2
    nodePort: 31125
  - port: 8280
    name: port3
    nodePort: 31126
  - port: 8243
    name: port4
    nodePort: 31127
  selector:
    app: api-manager
    role: api-manager
EOF
```
### 2.2 Tạo file `am-analytics-worker.yaml` với nội dung sau:**
```sh
cat << EOF > am-analytics-worker.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: am-analytics-worker-nfs-pv-config
  namespace: wso2
  labels:
    storage: am-analytics-worker-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/worker"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: am-analytics-worker-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: am-analytics-worker-nfs-pv-config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: am-analytics-worker-deployment
  namespace: wso2
  labels:
    app: am-analytics-worker
    role: am-analytics-worker
spec:
  selector:
    matchLabels:
      app: am-analytics-worker
      role: am-analytics-worker
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: am-analytics-worker
        role: am-analytics-worker
    spec:
      containers:
      - name: am-analytics-worker
        image: wso2/wso2am-analytics-worker:3.0.0
        ports:
        - containerPort: 9091
          name: port1
        - containerPort: 9444
          name: port2
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: am-analytics-worker-config
      volumes:
      - name: am-analytics-worker-config
        persistentVolumeClaim:
          claimName: am-analytics-worker-nfs-pvc-config
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-am-analytics-worker
  namespace: wso2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: am-analytics-worker-deployment
  minReplicas: 1
  maxReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
---
apiVersion: v1
kind: Service
metadata:
  name: am-analytics-worker
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9091
    name: port1
    nodePort: 31122
  - port: 9444
    name: port2
    nodePort: 31123
  selector:
    app: am-analytics-worker
    role: am-analytics-worker
EOF
```
### 2.2 Tạo file `am-analytics-worker.yaml` với nội dung sau:
```sh
cat << EOF > am-analytics-dashboard.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: am-analytics-dashboard-nfs-pv-config
  namespace: wso2
  labels:
    storage: am-analytics-dashboard-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/dashboard/config"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: am-analytics-dashboard-nfs-pv-artifact
  namespace: wso2
  labels:
    storage: am-analytics-dashboard-nfs-pv-artifact
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/dashboard/artifact"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: am-analytics-dashboard-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: am-analytics-dashboard-nfs-pv-config
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: am-analytics-dashboard-nfs-pvc-artifact
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: am-analytics-dashboard-nfs-pv-artifact
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: am-analytics-dashboard-deployment
  namespace: wso2
  labels:
    app: am-analytics-dashboard
    role: am-analytics-dashboard
spec:
  selector:
    matchLabels:
      app: am-analytics-dashboard
      role: am-analytics-dashboard
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: am-analytics-dashboard
        role: am-analytics-dashboard
    spec:
      containers:
      - name: am-analytics-dashboard
        image: wso2/wso2am-analytics-dashboard:3.0.0
        ports:
        - containerPort: 9643
          name: port1
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: am-analytics-dashboard-config
        - mountPath: /home/wso2carbon/wso2-artifact-volume
          name: am-analytics-dashboard-artifact
      volumes:
      - name: am-analytics-dashboard-config
        persistentVolumeClaim:
          claimName: am-analytics-dashboard-nfs-pvc-config
      - name: am-analytics-dashboard-artifact
        persistentVolumeClaim:
          claimName: am-analytics-dashboard-nfs-pvc-artifact
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-am-analytics-dashboard
  namespace: wso2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: am-analytics-dashboard-deployment
  minReplicas: 1
  maxReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
---
apiVersion: v1
kind: Service
metadata:
  name: am-analytics-dashboard
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9643
    name: port1
    nodePort: 31128
  selector:
    app: am-analytics-dashboard
    role: am-analytics-dashboard
EOF
```

## 3. Thực hiện deploy các `service`, `deployment`, `hpa`, `pv`, `pvc` vừa tạo như sau:
Tạo Namespace `wso2`
```sh
kubectl create namespace wso2
```
Thực hiện `deploy`, `service`, `deployment`, `hpa`, `pv`, `pvc`
```sh
kubectl apply -f am-analytics-dashboard.yaml
kubectl apply -f api-manager.yaml
kubectl apply -f am-analytics-worker.yaml
```
