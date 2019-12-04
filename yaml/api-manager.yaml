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
