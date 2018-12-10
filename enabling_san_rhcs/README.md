# ENABLING SAN EXTENSION - RHCS 

## __1. INTRODUCTION__

Subject Alternative Name or SAN provides alternative DNS name for single certificate. During SSL handshake, SNI or server name indication will be checked against the server certificate to make sure its match each other. This is to cater the use of single certificate over multiple SNI name.

RHCS has certificate signer profiles when user submit their CSR (Certificate Signing Request) to the respective signing request. This profile has informations on what to be included into final certificate after signing.

By default, RHCS serverCert profile does not include SAN as part of the certificate X509 extension. We need to enable this into the profile.

## __2. CONFIGURATION STEPS__

1. Edit /var/lib/pki/pki-tomcat/ca/profiles/ca/caServerCert.cfg and add below lines and add the ```<number>``` into policyset.serverCertSet.list:

```
policyset.<profile>.<number>.constraint.class_id=noConstraintImpl
policyset.<profile>.<number>.constraint.name=No Constraint
policyset.<profile>.<number>.constraint.subjAltNameExtCritical=false
policyset.<profile>.<number>.default.class_id=userExtensionDefaultImpl
policyset.<profile>.<number>.default.name=User Supplied Extension Default
policyset.<profile>.<number>.default.params.userExtOID=2.5.29.17
```


In my case, it looks like this:
```
policyset.serverCertSet.9.constraint.class_id=noConstraintImpl
policyset.serverCertSet.9.constraint.name=No Constraint
policyset.serverCertSet.9.constraint.subjAltNameExtCritical=false
policyset.serverCertSet.9.default.class_id=userExtensionDefaultImpl
policyset.serverCertSet.9.default.name=User Supplied Extension Default
policyset.serverCertSet.9.default.params.userExtOID=2.5.29.17
policyset.serverCertSet.list=1,2,3,4,5,6,7,8,9
```

2. Restart pki.
```
systemctl restart pki-tomcatd.target
```

3. Create a new CSR with below configurations:
```
[req] distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no
[req_distinguished_name]
C = MY
ST = Selangor
L = Sepang
O = Red Hat
OU = Consulting
CN = ocpmaster01.xxx.net
[v3_req]
basicConstraints = CA:FALSE
keyUsage = keyEncipherment, dataEncipherment,digitalSignature
extendedKeyUsage = serverAuth, 1.3.6.1.5.5.8.2.2
subjectAltName = @alt_name

[alt_name]
DNS.1=ocpmaster.xxx.net
DNS.2=ocpmaster-int.xxx.net
```

4. Create a CSR from above config:
```
openssl req -out master.csr -newkey rsa:2048 -nodes -keyout master.key -config master.conf
```

5.Submit the CSR for signing into RHCS and your final certificate should have SAN entry after signing.
