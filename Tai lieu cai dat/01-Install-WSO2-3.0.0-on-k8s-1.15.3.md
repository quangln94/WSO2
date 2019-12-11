**Mô hình LAB**

|Server|IP|
|------|--|
|master1|10.1.38.128|
|master2|10.1.38.111|
|master3|10.1.38.149|
|worker1|10.1.38.144|
|worker2|10.1.38.146|
|haproxy1|10.1.38.137|
|haproxy2|10.1.38.138|
|mysql|10.1.38.147|
|NFS|10.1.38.129|
|Registry|10.1.38.42|
|Logging|10.1.38.139|


## 1. Thực hiện trên NFS
### 1.1 Tạo thư mục chia sẻ và phân quyền trên NFS
```sh
mkdir -p /data/wso2/worker
mkdir -p /data/wso2/dashboard
mkdir -p /data/wso2/apim
chown -R wso2carbon:wso2 /data/wso2
```
### 1.2 Chỉnh sửa Config
**Clone repository wso2:**
```sh
git clone https://github.com/wso2/docker-apim.git
```
**Copy các file cấu hình vào thư mục chia sẻ tương ứng:**
```sh
cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim/config /data/wso2/apim
cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim/artifact /data/wso2/apim 

cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim-analytics-worker/config /data/wso2/worker

cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim-analytics-dashboard/config /data/wso2/dashboard
cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim-analytics-dashboard/artifact /data/wso2/dashboard
```

**Sửa file cấu hình của `apim` như sau:**

```sh
vim /data/wso2/apim/config/repository/conf/deployment.toml
[server]
#hostname = "10.1.38.128"
hostname = "api-manager"
node_ip = "10.1.38.128"
offset=22057
-----------------------
[database.apim_db]
type = "mysql"
url = "jdbc:mysql://10.1.38.147:3306/WSO2AM_DB?autoReconnect=true&amp;useSSL=false"
-----------------------

[database.shared_db]
type = "mysql"
url = "jdbc:mysql://10.1.38.147:3306/WSO2AM_SHARED_DB?autoReconnect=true&amp;useSSL=false"
-----------------------
[[apim.gateway.environment]]
service_url = "https://10.1.38.128:${mgt.transport.https.port}/services/"
ws_endpoint = "ws://10.1.38.128:9099"
wss_endpoint = "wss://10.1.38.128:8099"
http_endpoint = "http://10.1.38.128:${http.nio.port}"
https_endpoint = "https://10.1.38.128:${https.nio.port}"
-----------------------
[apim.oauth_config]
revoke_endpoint = "https://10.1.38.128:${https.nio.port}/revoke"
```
Trong đó cần lưu ý các trường sau:
- `10.1.38.128`: Thay đổi toàn bộ IP này bằng IP của máy mình
- `10.1.38.147`: Thay đổi toàn bộ IP này bằng IP của máy MYSQL
- `offset=22057`: Giá trị này thay đổi the Port mà bạn expose. Ở đây expose `port 9443` của `api-manager` ra `port 31500` nên giá trị `offset=31500-9443=22057`

**Sửa file cấu hình của `worker` như sau:**

```sh
vim /data/wso2/worker/conf/worker/deployment.yaml
--------------------------------------------------------------------------------
- name: WSO2_PERMISSIONS_DB
      jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_PERMISSIONS_DB?useSSL=false'
--------------------------------------------------------------------------------     
- name: APIM_ANALYTICS_DB
      jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_STATS_DB?useSSL=false'
```
Trong đó lưu ý 1 số trường sau:
- `10.1.38.147`: Thay toàn bộ IP này bằng IP của máy MYSQL

**Sửa file cấu hình của `dashbroad` như sau:**

```sh
vim /data/wso2/worker/conf/worker/deployment.yaml
------------------------------------------------------------------------------------
- name: BUSINESS_RULES_DB
      jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_BUSINESS_RULES_DB?useSSL=false'
------------------------------------------------------------------------------------
- name: WSO2_PERMISSIONS_DB
      jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_PERMISSIONS_DB?useSSL=false'
```
Trong đó lưu ý 1 số trường sau:
- `10.1.38.147`: Thay toàn bộ IP này bằng IP của máy MYSQL của mình.

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
        - containerPort: 8280
          name: port1
        - containerPort: 30300                  # Port mặc định + giá trị offset
          name: port2
        - containerPort: 9763
          name: port3
        - containerPort: 31500                  # Port mặc định + giá trị offset
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
Trong đó lưu ý 1 số trường sau:
- `containerPort: 30300`: Giá trị mặc định là 8243
- `containerPort: 31500`: Giá trị mặc định là 9443

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
  - port: 8280
    name: port1
    nodePort: 31124
  - port: 30300
    name: port2
    nodePort: 30300
  - port: 9763
    name: port3
    nodePort: 31126
  - port: 31500
    name: port4
    nodePort: 31500
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
        - containerPort: 8280
          name: port1
        - containerPort: 30300
          name: port2
        - containerPort: 9763
          name: port3
        - containerPort: 31500
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
  - port: 8280
    name: port1
    nodePort: 31124
  - port: 30300
    name: port2
    nodePort: 30300
  - port: 9763
    name: port3
    nodePort: 31126
  - port: 31500
    name: port4
    nodePort: 31500
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
        - containerPort: 9764
          name: port1
        - containerPort: 9444
          name: port2
        - containerPort: 7612
          name: port3
        - containerPort: 7712
          name: port4
        - containerPort: 9091
          name: port5
        - containerPort: 7071
          name: port6
        - containerPort: 7444
          name: port7
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
  - port: 9764
    name: port1
    nodePort: 31122
  - port: 9444
    name: port2
    nodePort: 31123
  - port: 7612
    name: port3
    nodePort: 31121
  - port: 7712
    name: port4
    nodePort: 31120
  - port: 9091
    name: port5
    nodePort: 31119
  - port: 7071
    name: port6
    nodePort: 31118
  - port: 7444
    name: port7
    nodePort: 31117
  selector:
    app: am-analytics-worker
    role: am-analytics-worker
EOF
```
### 2.3 Tạo file `am-analytics-dashboard.yaml` với nội dung sau:
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
        - containerPort: 9713
          name: port1
        - containerPort: 9643
          name: port2
        - containerPort: 9613
          name: port3
        - containerPort: 7713
          name: port4
        - containerPort: 9091
          name: port5
        - containerPort: 7613
          name: port6
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
  - port: 9713
    name: port1
    nodePort: 31128
  - port: 9643
    name: port2
    nodePort: 31129
  - port: 9613
    name: port3
    nodePort: 31130
  - port: 7713
    name: port4
    nodePort: 31131
  - port: 9091
    name: port5
    nodePort: 31132
  - port: 7613
    name: port6
    nodePort: 31133
  selector:
    app: am-analytics-dashboard
    role: am-analytics-dashboard
EOF
```
## 3. Thực hiện trên MYSQL 

**Import database vào MYSQL**

Clone repository trên Github:
```sh
git clone https://github.com/wso2/docker-apim.git
```
```sh
cd /root/docker-apim/docker-compose/apim-with-analytics/mysql/scripts

mysql -u username -p  < mysql_analytics.sql
mysql -u username -p  < mysql_apim.sql
mysql -u username -p  < mysql_shared.sql
```
Set `max_connections` trên mysql:
```sh
mysql -u username -p

set global max_connections = 2000;
```
## 4. Thực hiện deploy các `service`, `deployment`, `hpa`, `pv`, `pvc` vừa tạo như sau:
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
