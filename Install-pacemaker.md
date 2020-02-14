## Mô hình Lab

|Server|controller1|controller2|controller3|
|------|-----------|-----------|-----------|
|eth1|192.168.20.11|192.168.20.12|192.168.20.13|
|eth3|192.168.40.11|192.168.40.12|192.168.40.13|

IP VIP: 192.168.20.20

## 1. Cài đặt MariaDB

Thực hiện trên tất cả các node

Khai báo repo
```sh
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo
yum update -y
```
Cài đặt MariaDB và các gói hỗ trợ
```sh
yum install mariadb-server -y
yum install -y galera rsync
```
Tắt MariaDB
```sh
systemctl stop mariadb
```
***Lưu ý: Không khởi động dịch vụ mariadb sau khi cài (Liên quan tới cấu hình Galera Mariadb)***
## Cấu hình Galera Cluster
**Thực hiện trên Node controller1**
```sh
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=192.168.20.11

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://192.168.40.11,192.168.40.12,192.168.40.13"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="portal_cluster"
# Allow server to accept connections on all interfaces.
bind-address=192.168.20.11
# this server ip, change for each server
wsrep_node_address="192.168.40.11"
# this server name, change for each server
wsrep_node_name="controller1"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]' > /etc/my.cnf.d/server.cnf
```
**Thực hiện trên Node controller2**
```sh
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=192.168.20.12

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://192.168.40.11,192.168.40.12,192.168.40.13"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="portal_cluster"
# Allow server to accept connections on all interfaces.
bind-address=192.168.20.12
# this server ip, change for each server
wsrep_node_address="192.168.40.12"
# this server name, change for each server
wsrep_node_name="controller2"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]' > /etc/my.cnf.d/server.cnf
```
**Thực hiện trên Node controller3**
```sh
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=192.168.20.13

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#add your node ips here
wsrep_cluster_address="gcomm://192.168.40.11,192.168.40.12,192.168.40.13"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#Cluster name
wsrep_cluster_name="portal_cluster"
# Allow server to accept connections on all interfaces.
bind-address=192.168.20.13
# this server ip, change for each server
wsrep_node_address="192.168.40.13"
# this server name, change for each server
wsrep_node_name="controller3"
wsrep_sst_method=rsync
[embedded]
[mariadb]
[mariadb-10.2]' > /etc/my.cnf.d/server.cnf
```
**Thực hiên trên Node controller1**
```sh
galera_new_cluster
systemctl start mariadb
systemctl enable mariadb
```
**Thực hiện trên 2 Node còn lại**
```sh
systemctl start mariadb
systemctl enable mariadb
```
Kiểm tra trạng thái Cluster
```sh
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```
Tạo User phục vụ HAProxy
```sh
mysql -u root -p
CREATE USER 'haproxy'@'controller1';
CREATE USER 'haproxy'@'controller2';
CREATE USER 'haproxy'@'controller3';
CREATE USER 'haproxy'@'%';
exit;
```
## 2. Cài đặt HAProxy

**Thực hiện trên tất cả các Node:**
```sh
yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y
```
Backup cấu hình:
```sh
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```
Cầu hình Haproxy
```sh
echo 'global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats
    bind :8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics

listen galera
    bind 192.168.20.20:3306
    balance source
    mode tcp
    option tcpka
    option tcplog
    option clitcpka
    option srvtcpka
    timeout client 28801s
    timeout server 28801s
    option mysql-check user haproxy
    server controller1 192.168.20.21:3306 check inter 5s fastinter 2s rise 3 fall 3
    server controller2 192.168.20.22:3306 check inter 5s fastinter 2s rise 3 fall 3 backup
    server controller3 192.168.20.23:3306 check inter 5s fastinter 2s rise 3 fall 3 backup' > /etc/haproxy/haproxy.cfg
```
Cấu hình Log cho HAProxy
```sh
sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf
echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf
```
Restart 
```sh
systemctl restart rsyslog
```
Bổ sung cấu hình cho phép kernel có thể binding tới IP VIP
```sh
echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf
```
Kiểm tra
```sh
sysctl -p
```
Tắt dịch vụ HAProxy
```sh
systemctl stop haproxy
systemctl disable haproxy
```
Tạo user haproxy, phục vụ plugin health check của HAProxy (option mysql-check user haproxy)
```sh
mysql -u root -p
CREATE USER 'haproxy'@'controller1';
CREATE USER 'haproxy'@'controller2';
CREATE USER 'haproxy'@'controller3';
CREATE USER 'haproxy'@'%';
```
## 3. Cài đặt Pacemaker corosync

Cài đặt pacemaker, start và enable pacemaker
```sh
yum -y install pacemaker pcs
systemctl start pcsd 
systemctl enable pcsd
```
Thiết lập mật khẩu user hacluster
```sh
passwd hacluster
```
***Lưu ý: Nhập chính xác và nhớ mật khẩu user hacluster, đồng bộ mật khẩu trên tất cả các node***

Chứng thực cluster (Chỉ thực thiện trên cấu hình trên một node duy nhất, trong bài sẽ thực hiện trên controller1), nhập chính xác tài khoản user hacluster
```sh
pcs cluster auth controller1 controller2 controller3
```
Khởi tạo cấu hình cluster ban đầu
```sh
pcs cluster setup --name ha_cluster controller1 controller2 controller3
```
***Lưu ý: controller1, controller2, controller3: Hostname các node thuộc cluster, yêu cầu khai báo trong `/etc/hosts`***

Khởi động Cluster và cho phép start cùng hệ thống
```sh
pcs cluster start --all
pcs cluster enable --all
```
**Thiết lập Cluster**

Bỏ qua cơ chế STONITH
```sh
pcs property set stonith-enabled=false
```
Cho phép Cluster chạy kể cả khi mất quorum
```sh
pcs property set no-quorum-policy=ignore
```
Hạn chế Resource trong cluster chuyển node sau khi Cluster khởi động lại
```sh
pcs property set default-resource-stickiness="INFINITY"
```
Kiểm tra thiết lập cluster
```sh
pcs property list
```
Tạo Resource IP VIP Cluster
```sh
pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=192.168.20.20 cidr_netmask=24 op monitor interval=30s
```
Tạo Resource quản trị dịch vụ HAProxy
```sh
pcs resource create Loadbalancer_HaProxy systemd:haproxy op monitor timeout="5s" interval="5s"
```
Ràng buộc thứ tự khởi động dịch vụ, khởi động dịch vụ Virtual_IP sau đó khởi động dịch vụ Loadbalancer_HaProxy
```sh
pcs constraint order start Virtual_IP then Loadbalancer_HaProxy kind=Optional
```
Ràng buộc resource Virtual_IP phải khởi động cùng node với resource Loadbalancer_HaProxy
```sh
pcs constraint colocation add Virtual_IP Loadbalancer_HaProxy INFINITY
```
Kiểm tra trạng thái Cluster
```sh
pcs status
```
Kiêm tra cấu hình Resource
```sh
pcs resource show --full
```
Kiểm tra ràng buộc trên resource
```sh
pcs constraint
```
## 4. Kiểm tra trạng thái dịch vụ
- haproxy: http://192.168.20.20:8080/stats
- pacemaker: https://192.168.20.20:2224/login
- Truy cập MariaDB qua IP VIP: 
```sh
mysql -h 192.168.20.20 -u root
```
## Tài liệu tham khảo: 
- https://blog.cloud365.vn/linux/pacemaker-haproxy-galera/
- https://github.com/hocchudong/ghichep-pacemaker-corosync
- https://blog.cloud365.vn/linux/cai-dat-galera-mariadb/
- https://alvinalexander.com/blog/post/mysql/show-users-i-ve-created-in-mysql-database
