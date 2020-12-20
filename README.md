# Secure OpenLDAP server on Debian 10 Buster
- [X] Build and configure a basic OpenLDAP server
- [X] Configure logging
- [ ] Configure TLS (LDAPs) >> in progress
- [ ] Create a Base DN for Users and Groups
- [ ] Disable Anonymous access
- [ ] Set ACL, including Read Only user for LDAP binding
- [ ] Implement self service portal based on [Self Service Password](https://ltb-project.org/documentation/self-service-password)
- [ ] Build and configure Read Only slave directory replication

* [Build and configure a basic OpenLDAP server](#build-and-configure-a-basic-openldap-server)
* [Configure Logging](#configure-logging)

## Build and configure a basic OpenLDAP server
According to your security hardening policy, install a fresh debian 10 server then install and configure sudo.

Don't forget to update your server :
```
$ sudo apt update && apt upgrade -y
```
Install OpenLDAP
```
$ sudo apt install slapd ldap-utils rsyslog --no-install-recommends -y
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

Test the configuration :
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

Configure OpenLDAP
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
$ sudo vim /etc/logrotate.d/slapd
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
Reboot the server for activating logging
```
$ sudo /sbin/reboot
```
