# Active-Active HAProxy Software Load Balancing (with OCP Example)

## __1. INTRODUCTION__

In rare case, where there is a need to use HAProxy for load balancing for OpenShift, this blog should able to give an idea on how to setup. This setup has been running in Alliance Bank OCP Project, MY.

## __2. PRE-REQ__

A. DNS round-robin (for active-active) record for master URLs and Apps route.

B. Virtual Floating IP - depends on the requirement this can be single IP or multi IP for each particular route.

C.NODES

__HAPROXY__  
HAPROXY01  
HAPROXY02
 

__OCP MASTER__
OCPMASTER01  
OCPMASTER02  
OCPMASTER03  
 

__OCP ROUTER__  
OCPROUTE01  
OCPROUTE02  
OCPROUTE03  

D. FLOATING-IP  
192.128.103.137/24 - ACTIVE VIP-1 - DNS ENTRY-1  
192.128.103.138/24 - ACTIVE VIP-2 - DNS ENTRY-2

 
## __3. HIGH LEVEL OVERVIEW__

![alt text](https://aizuddin85.github.io/act-act-sw-lb/images/overview.png "ha-sw-lb-overview")


## __4. CONFIGURATION STEPS__

### __A. HAPROXY__

1. On both HAPROXY nodes, install and add firewall rules.

8443, 443 (80 wont be used) haproxy, keepalived,psmisc packages.
```
# yum install haproxy keepalived psmisc -y
# firewall-cmd --add-port=8443/tcp --add-port=443/tcp --permanent
# firewall-cmd --reload
# systemctl enable haproxy keepalived psmisc
```

2. On both HAPROXY nodes:

Configure /etc/haproxy/haproxy.cfg with below content.
This config is not include the fine tune. 

```
global
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

# OCP Management FrontEnd & BackEnd
frontend  ocpmanagement_frontend *:8443
mode tcp
default_backend ocpmanagement_backend_8443

backend ocpmanagement_backend_8443
balance roundrobin
mode tcp
server OCPMASTER01 OCPMASTER01:8443 check fall 2 rise 1 
server OCPMASTER02 OCPMASTER02:8443 check fall 2 rise 1 
server OCPMASTER03 OCPMASTER03:8443 check fall 2 rise 1 

# OCP ROute FrontEnd  & BackEnd
frontend  ocprouter_frontend *:443
mode tcp
default_backend ocprouter_backend_443

backend ocprouter_backend_443
balance source
mode tcp
maxconn 5000
server OCPROUTE01 OCPROUTE01:443 check fall 2 rise 1
server OCPROUTE02 OCPROUTE02:443 check fall 2 rise 1
server OCPROUTE03 OCPROUTE03:443 check fall 2 rise 1

# Enabling stat page on localhost 
listen     stats          127.0.0.1:5000
     mode            http
        log             global
     clitimeout     100s     
        srvtimeout     100s
     contimeout     100s
        timeout queue   100s
        maxconn         10

     stats          enable
     stats          hide-version
     stats          refresh 20s
     stats          show-node
     stats     auth     admin:password
     stats     uri     /haproxy?stats
```

3. Now configure KeepAlived that will give HA trough the VRRP.

HAPROXY01 (Note on how 'priority'  being set to declare a MASTER/BACKUP, higher prio --> MASTER)

* HAPROXY01

```
global_defs {
   router_id ocp_veep
   script_user root
   enable_script_security
}

vrrp_script haproxy_check {
   script "/bin/killall -0 haproxy"
   interval 2
   weight 2
}

vrrp_instance OCP_EXT_PRIMARY {
   interface eth0
   virtual_router_id 50
   priority 110
   state MASTER
   virtual_ipaddress {
   198.128.103.137/24 dev eth0 
   }
   track_script {
   haproxy_check
   }
   authentication {
   auth_type PASS
   auth_pass 8d41070e-75fe-441e-aebf-15519738013b 
   }
vrrp_instance OCP_EXT_SECONDARY {
   interface eth0
   virtual_router_id 51
   priority 100
   state BACKUP
   virtual_ipaddress {
   198.128.103.138/24 dev eth0 
   }
   track_script {
   haproxy_check
   }
   authentication {
   auth_type PASS
   auth_pass 8d41070e-75fe-441e-aebf-15519738013b 
   }
}
```

* HAPROXY02

```
global_defs {
   router_id ocp_veep
   script_user root
   enable_script_security
}

vrrp_script haproxy_check {
   script "/bin/killall -0 haproxy"
   interval 2
   weight 2
}

vrrp_instance OCP_EXT_SECONDARY {
   interface eth0
   virtual_router_id 50
   priority 100
   state BACKUP
   virtual_ipaddress {
   198.128.103.137/24 dev eth0 
   }
   track_script {
   haproxy_check
   }
   authentication {
   auth_type PASS
   auth_pass 8d41070e-75fe-441e-aebf-15519738013b 
   }
vrrp_instance OCP_EXT_PRIMARY {
   interface eth0
   virtual_router_id 51
   priority 110
   state MASTER
   virtual_ipaddress {
   198.128.103.138/24 dev eth0 
   }
   track_script {
   haproxy_check
   }
   authentication {
   auth_type PASS
   auth_pass 8d41070e-75fe-441e-aebf-15519738013b 
   }
}
```

4. Start all services
```
systemctl start haproxy
systemctl start keepalived
systemctl start psmisc
```

5. In order to test, create index.html and allow port 8443/443 on real server.

Repeat this command on all master.
```
echo "Hello from OCPMASTER01" > /root/index.html
```


6. Run python SimpleHTTPServer  to test the load balancing.
```
cd /root
python -m SimpleHTTPServer 8443
```

 7. Browse tp any of the VIP to port 8443 and refresh each time will point to different server index.html.
