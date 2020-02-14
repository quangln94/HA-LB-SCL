sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y

cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

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
	
sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf

echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf

systemctl restart rsyslog

echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf

sysctl -p

systemctl stop haproxy
systemctl disable haproxy

yum -y install pacemaker pcs
systemctl start pcsd 
systemctl enable pcsd

passwd hacluster

pcs cluster auth controller1 controller2 controller3
pcs cluster setup --name ha_cluster controller1 controller2 controller3

pcs cluster start --all

pcs cluster enable --all

pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs property set default-resource-stickiness="INFINITY"
pcs property list
pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=192.168.20.20 cidr_netmask=24 op monitor interval=30s
pcs resource create Loadbalancer_HaProxy systemd:haproxy op monitor timeout="5s" interval="5s"
pcs constraint order start Virtual_IP then Loadbalancer_HaProxy kind=Optional
pcs constraint colocation add Virtual_IP Loadbalancer_HaProxy INFINITY
pcs status
pcs resource show --full
pcs constraint

http://192.168.20.20:8080/stats
https://192.168.20.20:2224/login

mysql -h 192.168.20.20 -u root

https://blog.cloud365.vn/linux/pacemaker-haproxy-galera/
https://github.com/hocchudong/ghichep-pacemaker-corosync
