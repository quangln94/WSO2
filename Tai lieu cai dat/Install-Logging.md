Cài đặt Logging

**Cài đặt Java**
```sh
apt install openjdk-8-jdk -y
java -version
```
**Install nginx**
```sh
apt install nginx -y
ufw allow 'Nginx Full'
ufw reload
ufw app list
```
**Install Elasticsearch**
```sh
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
apt update
apt install elasticsearch
```
**Chỉnh sửa file cấu hình**
```sh
vim /etc/elasticsearch/elasticsearch.yml
==>
cluster.name: idp
node.name: node-1
network.host: [_local_, _ens160_]
http.port: 9200
```
```sh
systemctl start elasticsearch
systemctl enable elasticsearch
```
```sh
curl -X GET "localhost:9200"
```
**Install Kibana**
```sh
apt install kibana
```
```sh
systemctl enable kibana
systemctl start kibana
```
echo "admin:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users
==> IdpSI@!2019

**Cấu hình nginx:**
```sh
vim /etc/nginx/sites-available/idp.vnpt.vn
==>
server {
    listen 80;

    server_name 10.1.38.139;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
**Khai báo enable site**
```sh
ln -s /etc/nginx/sites-available/idp.vnpt.vn /etc/nginx/sites-enabled/idp.vnpt.vn
```
**check config nginx**
```sh
nginx -t
systemctl restart nginx
```
**đăng nhập kibana: `http://10.1.38.139/status`

**install Fluentd**
Thực hiện trên master1. Đầu tiên ta tạo file khai báo
```sh
cd
mkdir install_log
cd install_log
```
```sh
cat << EOF > fluentd.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "10.1.38.139"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
EOF
```
**Chạy lệnh cài đặt**
```sh
kubectl apply -f fluentd.yaml
```
**Trên kibana ta tạo index logstash-timestamp**

==> Management bên trái

phần create index pattern điền: logstash-* rồi next

phần configure settings chọn @timestamp
