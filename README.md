# Secure OpenLDAP server on Debian 10 Buster
- [X] Build and configure a basic OpenLDAP server
- [X] Configure logging
- [X] Configure TLS (LDAPs)
- [X] Disable Anonymous access
- [ ] Create a Base DN for Users and Groups  >> in progress
- [ ] Set ACL, including Read Only user for LDAP binding
- [ ] Implement self service portal based on [Self Service Password](https://ltb-project.org/documentation/self-service-password)
- [ ] Build and configure Read Only slave directory replication

* [Build and configure a basic OpenLDAP server](#build-and-configure-a-basic-openldap-server)
* [Configure Logging](#configure-logging)
* [Configure TLS encryption](#configure-tls-encryption)
* [Disable Anonymous access](#disable-anonymous-access)

## Build and configure a basic OpenLDAP server
According to your security hardening policy, install a fresh debian 10 server then install and configure sudo.

Don't forget to update your server:
```
$ sudo apt update && apt upgrade -y
```
Install OpenLDAP
```
$ sudo apt install slapd ldap-utils rsyslog gnutls-bin --no-install-recommends -y
```
OpenLDAP package configuration screen is automaticaly started: set then confirm the password for admin.

OpenLDAP is configured with the defaut system configuration: it must be reconfigured:
```
$ sudo dpkg-reconfigure slapd
```
For the first screen: to the question “omit OpenLDAP server configuration”: select "No".

Next screen; set a FQDN as the base DN: in our case "test.local".

Next screen: set the name of the organization: in our case, base DN is used.

Next screen: set then confirm the password for admin.

Next screen: select OpenLDAP database backend: MDB is recommended type.

Next screen: to the question “Do you want the database to be removed when slapd is purged?”: select “Yes”.

Next screen: to the question “Move old database?": select “Yes”.

Test the configuration:
```
$ sudo slapcat
dn: dc=test,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: test.local
dc: test
....
dn: cn=admin,dc=test,dc=local
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
....
```

## Configure Logging

Configure OpenLDAP:
```
$ sudo ldapmodify -Y external -H ldapi:/// << OEF
dn: cn=config
changeType: modify
replace: olcLogLevel
olcLogLevel: stats
OEF
```
Update Rsyslog configuration:
```
$ echo "local4.* /var/log/slapd.log" | sudo tee -a /etc/rsyslog.conf
```
Configure log rotation:
```
$ sudo cat /etc/logrotate.d/slapd
/var/log/slapd.log
{ 
        rotate 10
        daily
        size=50M
        missingok
        notifempty
        delaycompress
        compress
        postrotate
                /usr/lib/rsyslog/rsyslog-rotate
        endscript
}
```
Reboot the server for activating logging:
```
$ sudo /sbin/reboot
```
## Configure TLS encryption
This configuration activate LDAP with StartTLS extension (port 389) and LDAPs (port 636)

Create directories to store certificates:
```
$ sudo mkdir -p /etc/openldap/tls/
$ sudo mkdir -p /etc/openldap/ca/
```

Create the CA key file:
```
$ sudo certtool --generate-privkey --sec-param high --outfile /etc/openldap/ca/ca.key
```

Create the CA certificate file:
```
$ cat /etc/openldap/ca/ca.info
cn=CA Secure OpenLDAP 
ca
cert_signing_key

$ sudo certtool --generate-self-signed --load-privkey /etc/openldap/ca/ca.key --template /etc/openldap/ca/ca.info --outfile /etc/openldap/ca/ca-cert.pem
```

Create the OpenLDAP server key file:
```
$ sudo certtool --generate-privkey --sec-param high --outfile /etc/openldap/tls/ldap.test.local.key
```
Create the OpenLDAP server certificate file:
```
$ cat /etc/openldap/tls/ldap.test.local.info
organization = Test
cn = ldap.test.local
tls_www_server
encryption_key
signing_key
expiration_days = 365

$ sudo certtool --generate-certificate --load-privkey /etc/openldap/tls/ldap.test.local.key --load-ca-certificate /etc/openldap/ca/ca-cert.pem --load-ca-privkey /etc/openldap/ca/ca.key --template /etc/openldap/tls/ldap.test.local.info --outfile /etc/openldap/tls/ldap.test.local.pem
```
Activate TLS:

In /etc/default/slapd

replace the following line 

SLAPD_SERVICES="ldap:/// ldapi:///"

with

SLAPD_SERVICES="ldap:/// ldaps:///  ldapi:///"

Then add the following lines

TLS_CACERTDIR="/etc/openldap/tls/"                                                                                                                                                                                                          

TLS_CACERT="/etc/openldap/ca/ca-cert.pem" 

Configure certificates:
```
$ cat tls.ldif
dn: cn=config
changetype: modify
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/tls/ldap.test.local.pem

add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/tls/ldap.test.local.key

add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/ca/ca-cert.pem

$ sudo ldapmodify -Y EXTERNAL -H ldapi:// -f tls.ldif
$ sudo systemctl restart slapd
```
Force TLS only:
```
$ cat forcetls.ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSecurity
olcSecurity: tls=1

$  sudo ldapmodify -H ldapi:// -Y EXTERNAL -f forcetls.ldif
```
## Disable Anonymous access
```
$ cat disable-anon.ldif
dn: cn=config
changetype: modify
add: olcDisallows
olcDisallows: bind_anon

dn: cn=config
changetype: modify
add: olcRequires
olcRequires: authc

dn: olcDatabase={-1}frontend,cn=config
changetype: modify
add: olcRequires
olcRequires: authc

$ sudo ldapadd -Y EXTERNAL -H ldapi:/// -f disable-anon.ldif
```
