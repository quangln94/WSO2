# Install Private Registry

## 1. Tạo `certificate`

**Tạo ssl certificate cho nginx. Chúng ta sẽ sử dụng nginx để làm proxy webserver cho registry. Do registry sử dụng port khác với 80/443 để cung cấp dịch vụ.**
```sh
mkdir -p /data/nginx/certs
mkdir -p /data/nexus
chown -R 200:200 /data/nexus
cd /data/nginx/certs
```
***Chú ý: Nếu ko sử dụng lệnh chown thì sẽ bị lỗi container của nexus do không có phân quyền***

**Tạo CA**
```sh
openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt
```
```sh
Country Name (2 letter code) [AU]:VN
State or Province Name (full name) [Some-State]:Ha Noi
Locality Name (eg, city) []:HN
Organization Name (eg, company) [Internet Widgits Pty Ltd]:VNPT-IT
Organizational Unit Name (eg, section) []:TT SI
Common Name (e.g. server FQDN or YOUR name) []:idp.registry.vnpt.vn
Email Address []:nguyentrongtan@vnpt.vn
```
**Tạo Certificate Signing Request**

```sh
openssl req -newkey rsa:4096 -nodes -sha256 -keyout idp.registry.vnpt.vn.key -out idp.registry.vnpt.vn.csr
```
```sh
Country Name (2 letter code) [AU]:VN
State or Province Name (full name) [Some-State]:Ha Noi
Locality Name (eg, city) []:HN
Organization Name (eg, company) [Internet Widgits Pty Ltd]:VNPT-IT
Organizational Unit Name (eg, section) []:TT SI
Common Name (e.g. server FQDN or YOUR name) []:idp.registry.vnpt.vn
Email Address []:nguyentrongtan@vnpt.vn

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:123456
An optional company name []:VNPT-IT
```
**Tạo certificate cho registry host**
```sh
openssl x509 -req -days 3650 -in idp.registry.vnpt.vn.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out idp.registry.vnpt.vn.crt
```
**Tạo file `/data/nginx/nginx.conf` để sử dụng cho container nginx với nội dung sau**
```sh
cd
cat << EOF > /data/nginx/nginx.conf
user  nginx;
  worker_processes  1;

  error_log  /var/log/nginx/error.log warn;
  pid        /var/run/nginx.pid;

   events {
      worker_connections  1024;
    }

    http {

    log_format  main  '\$remote_addr - \$remote_user [\$time_local] "\$request" '
                      '\$status \$body_bytes_sent "\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    proxy_send_timeout 120;
    proxy_read_timeout 300;
    proxy_buffering    off;
    keepalive_timeout  5 5;
    tcp_nodelay        on;

    server {
        listen         80;
        server_name    idp.registry.vnpt.vn;

    return         301 https://\$server_name\$request_uri;
    }

    server {
        listen   *:443 ssl;
        server_name  idp.registry.vnpt.vn;

        # allow large uploads of files - refer to nginx documentation
        client_max_body_size 2048m;

        # optimize downloading files larger than 1G - refer to nginx doc before adjusting
        #proxy_max_temp_file_size 2048m

        ssl on;
        ssl_certificate      /etc/nginx/nginx-signed.crt;
        ssl_certificate_key  /etc/nginx/nginx-signed.key;

        location / {
            proxy_pass http://nexus:8083/;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto "https";
        }
    }
}
EOF
```
## 2.Cài đặt docker, docker-compose để chạy nginx và nexus

**Thực hiện cài `docker-ce`:**
```sh
apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
apt update
apt install docker-ce -y
```
**Thực hiện cài đặt `docker-compose` bằng các lệnh:**
```sh
curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
**Tạo file `docker-compose.yaml` với nội dung sau:**
```sh
cd && mkdir -p nexus3 && cd nexus3
cat << EOF > docker-compose.yaml
version: "2"
services:
  nexus:
    image: sonatype/nexus3
    ports:
    - "8081:8081" # Port quản trị UI
    - "8083:8083" # Port registry
    volumes:
    - /data/nexus:/nexus-data
  nginx:
    image: nginx
    ports:
    - "80:80"
    - "443:443"
    volumes:
    - /data/nginx/nginx.conf:/etc/nginx/nginx.conf
    - /data/nginx/certs/idp.registry.vnpt.vn.crt:/etc/nginx/nginx-signed.crt
    - /data/nginx/certs/idp.registry.vnpt.vn.key:/etc/nginx/nginx-signed.key
    links:
    - nexus:nexus
EOF
```
**Tạo các container nginx và nexus bằng lệnh sau:**
```sh
docker-compose up -d
```
***Truy cập vào đường dẫn sau để vào registry: `http://10.1.38.128:8081/`***

***Sử dụng username là admin, mật khẩu là chuỗi ký tự trong file: `/data/nexus/admin.password`***

***Sau khi đăng nhập lần đầu thì đổi mật khẩu***

Thực hiện tạo repository bằng cách vào icon bánh răng (cạnh ô search components) --> Repository (tab ở cột trái) --> repositories --> create repository --> docker (hosted)

- Trong phần create cần:
	+ đặt tên repository: private-repo
	+ port http đặt là 8083
	+ cho dấu tick vào Enable Docker v1 API

<img src=https://i.imgur.com/0xw6Xpr.png>

- Thực hiện test push và pull image từ k8s, tạo thư mục trên các máy chủ master, worker trong cụm
```sh
mkdir -p /etc/docker/certs.d/idp.registry.vnpt.vn
```
**Copy file `ca.crt` từ node registry vào thư mục `/etc/docker/certs.d/idp.registry.vnpt.vn` trên tất cả các máy master và worker**

```sh
scp /data/nginx/certs/ca.crt user@10.1.38.111:/tmp
scp /data/nginx/certs/ca.crt user@10.1.38.128:/tmp
scp /data/nginx/certs/ca.crt user@10.1.38.149:/tmp
scp /data/nginx/certs/ca.crt user@10.1.38.137:/tmp
scp /data/nginx/certs/ca.crt user@10.1.38.138:/tmp
```
**Trên các node master, worker thực hiện sao chép `ca.crt` vào trong docker**
```sh
$ cp /tmp/ca.crt /etc/docker/certs.d/idp.registry.vnpt.vn/
```
**Thực hiện tạo secret để kết nối vào repository:**
```sh
kubectl create secret docker-registry nexus-secret-registry --docker-server=idp.registry.vnpt.vn --docker-username=admin --docker-password=pass@registry
```
**Check secret vừa tạo**
```sh
kubectl get secret
```
**Thực hiện upload image lên repo bằng cách sau:**
```sh
docker tag yourimage idp.registry.vnpt.vn/image1:v1
docker login idp.registry.vnpt.vn
docker push idp.registry.vnpt.vn/image1:v1
```
***Node: Yêu cầu phải copy file `ca.crt` sang thư mục cert của host muốn up/down image.***

**Thực hiện tạo pod, lấy image từ repo của nexus bằng file `deployment-test-registry.yaml` sau:**
```sh
cat << EOF > deployment-test-registry.yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: test-registry
  labels:
    app: test-registry
spec:
  selector:
    matchLabels:
      app: test-registry
  replicas: 1
  template:
    metadata:
      labels:
        app: test-registry
    spec:
      containers:
      - name: nginx
        image: idp.registry.vnpt.vn/image:v1
        command:
        - /bin/bash
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: nexus-secret-registry
EOF
```
**Chạy lệnh:**
```sh
kubectl apply -f deployment-test-registry.yaml
```
**Check pod vừa tạo:**
```sh
kubectl get pods -o wide
```
