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