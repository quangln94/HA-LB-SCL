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


## 3, Kiểm tra
# Tài liệu tham khảo
- digitalocean.com/community/tutorials/how-to-use-haproxy-to-set-up-mysql-load-balancing--3
