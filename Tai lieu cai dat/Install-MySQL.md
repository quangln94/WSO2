# Install MySQL Master-Master
## 1. Cài đặt MySQL trên 2 Node 
```sh
sudo apt-get update
sudo apt-get install mysql-server
mysql_secure_installation
```
## 2. Câu hình Master-Master
#### 1.2.1 Thực hiện trên Node 1

**Tạo databases**
```
mysql -uroot -p ( nhập password root của MySQL )
mysql>create database itlabvn character set utf8 collate utf8_general_ci;
mysql>use itlabvn;
mysql>create table users(id int, username varchar(20),password varchar(20));
mysql>insert into users values(1001,'admin','password');
```
**dump databases**

Lock database để ngăn người dùng thay đổi dữ liệu mới trong quá trình chúng ta replication MySQL
```sh
mysql> use itlabvn;
mysql> flush tables with read lock;
```
#### 1.2.2 Thực hiện trên Node 2

**Export dữ liệu MySQL database sử dụng câu lệnh mysqldump**
```sh
$ mysqldump --default-character-set=utf8 --opt --databases itlabvn --user=root --password > itlabvn.sql
```
**Unlock database để có thể đọc và ghi vào database bằng câu lệnh sau**
```sh
mysql> unlock tables;
```
**Copy file itlabvn.sql từ Node 1 sang Node 2**
```sh
scp itlabvn.sql root@172.16.68.102:~
```
**Tạo database `itlabvn` trên Node 2 sau đó import dữ liệu từ file `.sql` đã export trên Node 1**
```sh
mysql> create database itlabvn character set utf8 collate utf8_general_ci;
$ mysql -u root -p itlabvn < /root/itlabvn.sql
```
## 2. Cấu hình MySQL Master-Master

### 2.1 Cài đặt và cấu hình MySQL Master Master Replication trên Node 1 - 10.1.38.147
Thêm vào file `/etc/mysql/mysql.conf.d/mysqld.cnf` nội dung sau: 
```sh
$ vim /etc/mysql/mysql.conf.d/mysqld.cnf
bind-address            = 10.1.38.147
server-id=1
log-bin="mysql-bin"
relay-log="mysql-relay-log"
auto-increment-offset = 1
```
Restart lại mysql
```sh
systemctl restart mysqld
```
**Tạo user replicator trên mỗi server**
```sh
$ mysql -u root -p
GRANT REPLICATION SLAVE ON *.* TO 'replicas'@'%' IDENTIFIED BY '12345aA@';
FLUSH PRIVILEGES;
```
**Lấy thông tin file log để sử dụng trên mỗi server**
```sh
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      443 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
***Note lại `File` và `Position` để dùng cho Node 2***

**Thực hiện tiếp lệnh sau:
```sh
mysql> STOP SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST = '172.16.68.102', MASTER_USER = 'replicas', MASTER_PASSWORD = '12345aA@', MASTER_LOG_FILE = 'mysql-bin.000002', MASTER_LOG_POS = 449;
mysql> START SLAVE;
```
Trong đó:
- `MASTER_LOG_FILE = 'mysql-bin.000002'`: Lấy ở Node 2
- `MASTER_LOG_POS = 449`: lấy ở Node 2

Sau khi thực hiện xong, reboot lại Node 1
```sh
systemctl reboot
```

### 2.2 Cài đặt và cấu hình MySQL Master Master Replication trên Node 2 - 172.16.68.102
Thêm vào file `/etc/my.cnf` nội dung sau: 
```sh
$ vim /etc/my.cnf
server-id=2
log-bin="mysql-bin"
relay-log="mysql-relay-log"
auto-increment-offset = 2
```
Restart lại mysql
```sh
systemctl restart mysqld
```
**Tạo user replicator trên mỗi server**
```sh
$ mysql -u root -p
GRANT REPLICATION SLAVE ON *.* TO 'replicas'@'%' IDENTIFIED BY '12345aA@';
FLUSH PRIVILEGES;
```
**Lấy thông tin file log để sử dụng trên mỗi server**
```sh
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      449 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
***Note lại `File` và `Position` để dùng cho Node 1***

**Thực hiện tiếp lệnh sau:
```sh
mysql> STOP SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST = '172.16.68.101', MASTER_USER = 'replicas', MASTER_PASSWORD = '12345aA@', MASTER_LOG_FILE = 'mysql-bin.000002', MASTER_LOG_POS = 443;
mysql> START SLAVE;
```
Trong đó:
- `MASTER_LOG_FILE = 'mysql-bin.000002'`: Lấy ở Node 1
- `MASTER_LOG_POS = 443`: lấy ở Node 1

Sau khi thực hiện xong, reboot lại Node 2
```sh
systemctl reboot
```
## 3. Kiểm tra
**Thực hiện trên Node 1:**
```sh
mysql> create database test01;
```
**Kiểm tra trên Node2:**
```sh
mysql> show databases;
```
