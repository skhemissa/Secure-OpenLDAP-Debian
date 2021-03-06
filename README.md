# Secure OpenLDAP server on Debian 10 Buster
- [X] Build and configure a basic OpenLDAP server
- [X] Configure logging
- [X] Configure TLS (LDAPs)
- [X] Disable Anonymous access
- [X] Create a Base DN for Users and Groups
- [ ] Set ACL, including Read Only user for LDAP binding
- [ ] Implement self service portal based on [Self Service Password](https://ltb-project.org/documentation/self-service-password)
- [ ] Build and configure Read Only slave directory replication

* [Build and configure a basic OpenLDAP server](#build-and-configure-a-basic-openldap-server)
* [Configure Logging](#configure-logging)
* [Configure TLS encryption](#configure-tls-encryption)
* [Disable Anonymous access](#disable-anonymous-access)
* [Create base DN for users and groups](#create-base-dn-for-users-and-groups)

## Build and configure a basic OpenLDAP server
According to your security hardening policy, install a fresh debian 10 server then install and configure sudo.

Don't forget to update your server:
```
$ sudo apt update && sudo apt upgrade -y
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
$ sudo vi /etc/logrotate.d/slapd
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
$ sudo vi /etc/openldap/ca/ca.info
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
$ sudo vi /etc/openldap/tls/ldap.test.local.info
organization = Test
cn = ldap.test.local
tls_www_server
encryption_key
signing_key
expiration_days = 365

$ sudo certtool --generate-certificate --load-privkey /etc/openldap/tls/ldap.test.local.key --load-ca-certificate /etc/openldap/ca/ca-cert.pem --load-ca-privkey /etc/openldap/ca/ca.key --template /etc/openldap/tls/ldap.test.local.info --outfile /etc/openldap/tls/ldap.test.local.pem
```
Make certificate readable by users
```
$ sudo chmod 640 /etc/openldap/tls/ldap.test.local.pem && \
 sudo chmod 640 /etc/openldap/tls/ldap.test.local.key && \
 sudo chmod 644 /etc/openldap/ca/ca-cert.pem && \
 sudo chgrp openldap /etc/openldap/tls/ldap.test.local.pem && \
 sudo chgrp openldap /etc/openldap/tls/ldap.test.local.key && \
 sudo chgrp openldap /etc/openldap/ca/ca-cert.pem
```

Activate TLS:

```
$ sudo vi /etc/default/slapd
```
Replace the following line 
```
SLAPD_SERVICES="ldap:/// ldapi:///"
```
with
```
SLAPD_SERVICES="ldap:/// ldaps:///  ldapi:///"
```
Then add the following lines
```
TLS_CACERTDIR="/etc/openldap/tls/"
TLS_CACERT="/etc/openldap/ca/ca-cert.pem"
```
Configure certificates:
```
$ vi tls.ldif
dn: cn=config
changetype: modify
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/tls/ldap.test.local.key
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/tls/ldap.test.local.pem
-
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/ca/ca-cert.pem

$ sudo ldapmodify -Y EXTERNAL -H ldapi:// -f tls.ldif
$ sudo systemctl restart slapd
```

Force TLS only:
```
$ vi forcetls.ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSecurity
olcSecurity: tls=1

$  sudo ldapmodify -H ldapi:// -Y EXTERNAL -f forcetls.ldif
```
Configure the LDAP client to ignore self signed certificates by adding the following line in /etc/ldap/ldap.conf
```
TLS_REQCERT never
```
## Disable Anonymous access
```
$ vi disable-anon.ldif
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
## Create base DN for users and groups
```
$ vi basedn.ldif
dn: ou=people,dc=test,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=test,dc=local
objectClass: organizationalUnit
ou: groups

$ sudo ldapadd -x -H ldaps://localhost -D cn=admin,dc=test,dc=local -W -f basedn.ldif
```
Create first user

Generate a password for the user to create
```
$ /sbin/slappasswd
New password:
Re-enter new password:
{SSHA}encrypted_password

$ vi usertest.ldif
dn: uid=usertest,ou=people,dc=test,dc=local
objectClass: inetOrgPerson
objectClass: shadowAccount
cn: last_name
sn: first_name
userPassword: {SSHA}{SSHA}encrypted_password

$ sudo ldapadd -x -H ldaps://localhost -D cn=admin,dc=test,dc=local -W -f usertest.ldif
```
Create a new group:
```
$ vi groups.ldif
dn: cn=ldap-users,ou=groups,dc=test,dc=local
objectClass: top
objectClass: groupOfNames
member: uid=usertest,ou=people,dc=test,dc=local

$ sudo ldapadd -x -H ldaps://localhost -D cn=admin,dc=test,dc=local -W -f groups.ldif
```
ldif file for adding another user to existing group
```
$ vi user_group.ldif
dn: cn=webapp,ou=groups,dc=test,dc=local
changetype: modify
add: member
member: uid=other_user,ou=people,dc=test,dc=local
```
