***Virtual IP: 10.1.38.150***
## 1. Cài đặt, cấu hình HAproxy+keepalived

Thực hiện trên 2 Node: 10.1.38.137, 10.1.38.138

**Cài đặt HAProxy trên Ubuntu 16.04**
```sh
sudo apt-get -y install haproxy
```
Thêm dòng sau vào file `/etc/default/haproxy`
```sh
$ vim /etc/default/haproxy
ENABLE=1
```
**Cài đặt Keepalived trên Ubuntu 16.04**
```sh
sudo apt-get install keepalived
service keepalived start
```
**Cấu hình gắn IP lên carđ mạng:**
```sh
vim /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind=1
sysctl -p
```
**Install Haproxy, keepalived:**
```sh
yum install haproxy keepalived -y
```
**Setup keepalived configuration:**
```sh
$ mkdir -p /etc/keepalived
$ cat > /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_haproxy {
script "killall -0 haproxy"
interval 3
weight 3
}

vrrp_instance LAN_68 {
interface ens160
virtual_router_id 68
priority 100
advert_int 2

authentication {
  auth_type PASS
  auth_pass tan124
}

track_script {
  check_haproxy
}

virtual_ipaddress {
  10.1.38.150/24
  }
}
EOF
```
**Setup Haproxy**
```sh
cat > /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # Default ciphers to use on SSL-enabled listening sockets.
    # For more information, see ciphers(1SSL). This list is from:
    #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3

defaults
    log global
    mode http
    option httplog
    option dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend k8s
bind 10.1.38.150:6443
option tcplog
mode tcp
default_backend k8s-master-nodes


backend k8s-master-nodes
mode tcp
balance roundrobin
option tcp-check
server k8s-master-1 10.1.38.128:6443 check fall 3 rise 2
server k8s-master-2 10.1.38.111:6443 check fall 3 rise 2
server k8s-master-3 10.1.38.149:6443 check fall 3 rise 2
EOF
```
**Thực hiện restart haproxy và keepalived:**
```sh
systemctl restart haproxy
systemctl restart keepalived
```
**Kiểm tra:**
```sh
root@haproxy1:~# ip a | grep ens160
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 10.1.38.137/24 brd 10.1.38.255 scope global ens160
    inet 10.1.38.150/24 scope global secondary ens160
```
```sh
$ systemctl status haproxy
$ systemctl status keepalived
```
