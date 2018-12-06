# ENABLING MODERN TLS & CIPHERS ON ROUTER & MASTER

## __INTRODUCTION__

During VA scan, some cipher considered as not safe and need to be disabled. There are two places that we need to configure:

 
1. Master

2. Router


## __CONFIGURATION STEPS__


### __MASTER.__

On all master nodes: 
1. Edit /etc/origin/master/master-origin.yaml

```servingInfo:
  ...
  minTLSVersion: VersionTLS12
  cipherSuites:
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_RSA_WITH_AES_256_CBC_SHA
  - TLS_RSA_WITH_AES_128_CBC_SHA
  ```
2. Restart master components.

### __ROUTER.__

There are several built-in router cipher suite [[1]](https://mojo.redhat.com/external-link.jspa?url=https%3A%2F%2Fdocs.openshift.com%2Fcontainer-platform%2F3.11%2Farchitecture%2Fnetworking%2Froutes.html%23ciphers). To set router cipher to modern, update the router dc environment.

```
[root@bastion 3.11]# oc set env dc/router ROUTER_CIPHERS=modern
deploymentconfig.apps.openshift.io/router updated

[root@bastion 3.11]# oc get pods
NAME                       READY     STATUS              RESTARTS   AGE
docker-registry-1-gjqfb    1/1       Running             0          18h
registry-console-1-8g64s   1/1       Running             0          1d
router-2-8ldc2             1/1       Running             0          1m
router-3-deploy            0/1       ContainerCreating   0          2s
[root@bastion 3.11]# 

[root@bastion 3.11]# oc rsh router-3-wtzxz env | grep CIPHERS
ROUTER_CIPHERS=modern
[root@bastion 3.11]# 
```

The haproxy.config of the router looks like this now:

```
[root@bastion ~]# oc rsh router-3-wtzxz cat /var/lib/haproxy/conf/haproxy.config | grep cipher
# The default cipher suite can be selected from the three sets recommended by https://wiki.mozilla.org/Security/Server_Side_TLS,
# By default when a cipher set is not provided, intermediate is used.
  # Modern cipher suite (no legacy browser support) from https://wiki.mozilla.org/Security/Server_Side_TLS
  ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256
[root@bastion ~]# 
```

## __VERIFICATION__

Command to check:
```
openssl s_client -connect  `hostname`:8443 -tls1
openssl s_client -connect  `router_hostname`:443 -tls1
```
 Example output:
 It should has the "__Secure Renegotiation IS NOT supported__" keywords.
 ```
 CONNECTED(00000003)
140586350237584:error:1409442E:SSL routines:ssl3_read_bytes:tlsv1 alert protocol version:s3_pkt.c:1493:SSL alert number 70
140586350237584:error:1409E0E5:SSL routines:ssl3_write_bytes:ssl handshake failure:s3_pkt.c:659:
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 7 bytes and written 0 bytes
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1
    Cipher    : 0000
    Session-ID: 
    Session-ID-ctx: 
    Master-Key: 
    Key-Arg   : None
    Krb5 Principal: None
    PSK identity: None
    PSK identity hint: None
    Start Time: 1539170928
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
```

Or using nmap:

```
[root@mzali-fedora ~]# nmap  --script ssl-enum-ciphers -p 8443 ocp-con-prod.bytewise.com.my
Starting Nmap 7.70 ( https://nmap.org ) at 2018-12-03 16:38 +08
Nmap scan report for ocp-con-prod.bytewise.com.my (192.168.50.50)
Host is up (0.00015s latency).
rDNS record for 192.168.50.50: master01.bytewise.com.my

PORT     STATE SERVICE
8443/tcp open  https-alt
| ssl-enum-ciphers: 
|   TLSv1.2: 
|     ciphers: 
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|     compressors: 
|       NULL
|     cipher preference: server
|_  least strength: A
MAC Address: 52:54:00:AD:E3:3D (QEMU virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.74 seconds
[root@mzali-fedora ~]# 
```  
  
    
      
  
## __REFERENCES__

1. [Configure cipher-suites in etcd daemon?](https://mojo.redhat.com/external-link.jspaurl=https%3A%2F%2Faccess.redhat.com%2Fsolutions%2F3499651)
2. [How to create a secured route using only a specific cipher? ](https://mojo.redhat.com/external-link.jspa?url=https%3A%2F%2Faccess.redhat.com%2Fsolutions%2F3301341)
3. [Change the OpenShift router's SSL Protocol and Supported Cipher list?](https://mojo.redhat.com/external-link.jspa?url=https%3A%2F%2Faccess.redhat.com%2Fsolutions%2F2989001)
4. [Set ciphers and tls version for OpenShift Container Platform?](https://mojo.redhat.com/external-link.jspa?url=https%3A%2F%2Faccess.redhat.com%2Fsolutions%2F3374601)
