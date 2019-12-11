- cài prometheus luôn trên cụm kubernetes
- Ta sử dụng helm để cài đặt prometheus + grafana giám sát cho cụm kubernetes:
wget https://get.helm.sh/helm-v2.14.2-linux-amd64.tar.gz
# wget https://get.helm.sh/helm-v2.15.2-linux-amd64.tar.gz
tar -zxvf helm-v2.14.2-linux-amd64.tar.gz
cd linux-amd64/
cp helm /usr/local/bin/

- Tạo file role base access để cài tiller tương tác với cụm kubernetes:
cd
mkdir prometheus
cd prometheus
cat << EOF > rbac-config.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

- Apply file role base access vào hệ thống:
kubectl apply -f rbac-config.yaml

- Giờ sử dụng helm để tạo ra tiller dựa vào rbac được khai báo ở trên
helm init --service-account tiller

- Update repo cho helm bằng lệnh:
helm repo update

- Tạo một storageclass sử dụng NFS để lưu trữ dữ liệu cho pod prometheus. các dữ liệu này sẽ tồn tại bên ngoài container để đảm bảo dữ liệu ko bị mất. Yêu cầu thay đổi đúng tham số IP và thư mục lưu trữ dữ liệu. Tạo thư mục lưu trữ /data/prometheus trên nfs server
cat << EOF > storageclass-nfs-prometheus.yaml
replicaCount: 1
nfs:
  server: 10.1.38.129
  path: /data/prometheus
  mountOptions:
storageClass:
  create: true
  archiveOnDelete: false
  name: nfs-prometheus
  allowVolumeExpansion: true
EOF

- Sử dụng helm để tạo storage class:
helm install --name nfs-prometheus -f storageclass-nfs-prometheus.yaml stable/nfs-client-provisioner --namespace monitoring

- Check storage class vừa tạo:
kubectl get storageclass

- Thiết lập các tham số cần thiết cho các pod prometheus và grafana:
cat << EOF > custom-values.yaml
# Define Prometheus
prometheus:
  prometheusSpec:
    retention: 365d
    resources:
      limits:
        cpu: 400m
        memory: 1Gi
      requests:
        cpu: 400m
        memory: 1Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: nfs-prometheus
          resources:
            requests:
              storage: 50Gi
  
# Define Grafana
grafana:
  # Set password for Grafana admin user
  adminPassword: Tan@6789

EOF

- Thực hiện cài đặt prometheus bằng helm. các thông số lấy theo file vừa khai báo ở trên. Lệnh sau sẽ cài các pod của prometheus và grafana
# helm install --name prometheus-operator -f custom-values.yaml stable/prometheus-operator --namespace monitoring 
helm install --name prometheus-operator -f custom-values.yaml stable/prometheus-operator --namespace monitoring --version 6.7.3

3 helm upgrade prometheus-operator -f custom-values1.yaml stable/prometheus-operator --namespace monitoring --version 6.7.3

helm search stable/prometheus-operator
helm search prometheus-operator -l

- Thực hiện cài đặt service cho prometheus dashboard để truy cập từ bên ngoài vào dashboard. Ta tạo file sau
cat << EOF > prom-service.yml
apiVersion: v1
kind: Service
metadata:
  name: prom-service
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      nodePort: 30090
  selector:
    app: prometheus
EOF

- Khởi tạo service dashboard
kubectl apply -f prom-service.yml -n monitoring

- Sau khi khởi tạo vào service dashboard, ta có thể truy cập vào dashboard bằng đường dẫn sau, thay bằng IP bất kỳ của master hoặc worker:
http://{IP cluster}:30090/targets
http://10.1.38.128:30090/targets

- Cài đặt Grafana dashboard bằng cách tạo service expose port để kết nối từ bên ngoài vào:
cat << EOF > grafana-service.yml
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  labels:
    app: grafana
spec:
  type: NodePort
  ports:
    - port: 3000
##    nodePort: 30300
      nodePort: 30303
  selector:
    app: grafana
EOF

- Chạy lệnh sau để tạo service:
kubectl apply -f grafana-service.yml -n monitoring

- Truy cập vào địa chỉ sau để vào dashboard, thay bằng IP của master hoặc worker:
http://{grafana IP}:30303/
http://10.1.38.129:30303/

- Truy cập bằng user/pass: admin/Tan@6789

++++++++++++++++++++++++++++ END install prometheus +++++++++++++++++++++++++++++++++++
