# Cấu hình HAProxy cho MySQL(Master-Master) Ubuntu


Mô hình LAB

|Server|Haproxy 01|MySql 01|MySQL 02|
|------|----------|--------|--------|
|IP|192.168.40.100|

## 1. Cài đặt MySQL Master-Master
Tham khảo (tại đây)[]

Tạo User MySQL cho HAProxy để check trạng thái của server.

mysql -u root -p -e "INSERT INTO mysql.user (Host,User) values ('192.168.40.100','haproxy_check'); FLUSH PRIVILEGES;"

User cần quyền root để access từ HAProxy. Mặc định user root có thể đăng nhập cục bộ. Mặc dù điều này có thể được khắc phục bằng cách cấp quyền bổ sung cho User root, tuy nhiên nên có 1 user riêng với quyền root.

mysql -u root -p -e "GRANT ALL PRIVILEGES ON *.* TO 'haproxy_root'@'192.168.40.100' IDENTIFIED BY '123123' WITH GRANT OPTION; FLUSH PRIVILEGES"


## 2. Cài đặt HAProxy

Cài đạt MySQL Client trên HAProxy để test kết nối.

apt-get install mysql-client


Thử query lên 1 trong các Node từ user haproxy_root.

mysql -h 192.168.40.1 -u haproxy_root -p -e "SHOW DATABASES"

## Cài đặt HAProxy

apt-get install haproxy

Enable HAPrpxy bằng init script.

sed -i "s/ENABLED=0/ENABLED=1/" /etc/default/haproxy

Kiểm tra bằng lệnh sau

service haproxy
Usage: /etc/init.d/haproxy {start|stop|reload|restart|status}
Configuring HAProxy

Backup cấu hình
mv /etc/haproxy/haproxy.cfg{,.original}

Tạo file config với nội dung sau:

cat << EOF > /etc/haproxy/haproxy.cfg

#The first block is the global and defaults configuration block.
global
    log 127.0.0.1 local0 notice
    user haproxy
    group haproxy

defaults
    log global
    retries 2
    timeout connect 3000
    timeout server 5000
    timeout client 5000
listen mysql-cluster
    bind 127.0.0.1:3306
    mode tcp
    option mysql-check user haproxy_check
    balance roundrobin
    server mysql-1 192.168.40.1:3306 check
    server mysql-2 192.168.40.2:3306 check
listen 0.0.0.0:8080
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth haproxy1:haproxy1
    stats auth hâproxy2:haproxy2


Start HAProxy

service haproxy start

## 3, Kiểm tra

Sử dụng MySQL client để query HAProxy.

mysql -h 127.0.0.1 -u haproxy_root -p -e "SHOW DATABASES"

# Tài liệu tham khảo
- https://www.digitalocean.com/community/tutorials/how-to-use-haproxy-to-set-up-mysql-load-balancing--3
- https://towardsdatascience.com/high-availability-mysql-cluster-with-load-balancing-using-haproxy-and-heartbeat-40a16e134691
- https://viblo.asia/p/xay-dung-high-available-cho-mysql-server-voi-haproxy-va-keepalived-tren-ubuntu-naQZRg6Xlvx
