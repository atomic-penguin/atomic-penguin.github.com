---
layout: page
title: Ohio Linuxfest 2008 - Square Pegs in Round Holes, Linux in a Windows World 
date: 2008-10-11
comments: true
categories:
- conference talk
- ohiolinux
- Windows
- Active Directory
- Samba
---

These are my presentation slides for the 2008 Ohio Linuxfest.  I gave a well recieved talk on the topic of Windows, and Active Directory, Linux integration using Samba.  My [krb5](https://atomic-penguin.github.com/cookbook-krb5) handles the krb5.conf and pam configurations outlined in the examples below.

<!-- more -->

* [Slides](/papers/Square_Pegs_in_Round_Holes_Linux_in_a_Windows.ppt)
* [DNSnarf](https://github.com/atomic-penguin/dnsnarf).  This code is deprecated now in favor of the bind::ldap2zone recipe, included in the Opscode Chef cookbook, [bind](https://atomic-penguin.github.com/cookbook-bind)

Example config files

## /etc/krb5.conf
Example Kerberos 5.0 configuration for Active Directory domain CONTOSO.COM.
```bash
#
# Example Kerberos configuration for CONTOSO.COM kerberos realm
# kdc01.contoso.com and kdc03.contoso.com are Windows domain controllers
# for this environment.

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = CONTOSO.COM
 dns_lookup_realm = false
 dns_lookup_kdc = true
 ticket_lifetime = 24h
 forwardable = yes

[realms]
 CONTOSO.COM = {
  kdc = kdc01.contoso.com
  kdc = kdc03.contoso.com
  admin_server = kdc01.contoso.com
 }

[domain_realm]
 contoso.com = CONTOSO.COM
 .contoso.com = CONTOSO.COM

[appdefaults]
 pam = {
   debug = false
   ticket_lifetime = 36000
   renew_lifetime = 36000
   forwardable = true
   krb4_convert = false
 }
```

## /etc/samba/smb.conf
Example Samba 3 configuration for Active Directory/Winbind integration
```bash
#
# This is an example of how to setup Samba as a domain member 
# of CONTOSO.COM to provide file services.
#
# The winbind method used in this example works much like a Windows
# Domain Member file server in resolving usernames and groups over RPC.
#
# idmap provides predictable UID and GID
# generation based on the user's domain SID number

[global]
unix charset = LOCALE
workgroup = CONTOSO 
realm = CONTOSO.COM
server string = Contoso File Server
security = ADS
allow trusted domains = no
log level = 1
log file = /var/log/samba/%m.log
max log size = 50
idmap backend = rid:CONTOSO=1000-1000000
idmap uid = 1000-1000000
idmap gid = 1000-1000000
winbind cache time = 3600
winbind enum groups = yes
winbind use default domain = yes
winbind normalize names = yes
admin users = @CONTOSO\domain_admins
map acl inherit = yes
disable netbios = no
```

## /etc/httpd/conf.d/auth_kerb.conf
Example mod_auth_kerb configuration file for Apache2 authentication against Active Directory.
```bash
#
# This is an example of how to setup Kerberos
# authentication in Apache against CONTOSO.COM and
# SUBDOMAIN.CONTOSO.COM.  The end result works much
# like Windows Integrated Authentication.

# Load the module into Apache
LoadModule auth_kerb_module modules/mod_auth_kerb.so

# Directory to protect with Kerberos authentication
<Location /path/to/directory>
  AuthType Kerberos
  AuthName "Contoso Login"
  KrbMethodNegotiate Off
  KrbMethodK5Passwd On
  KrbSaveCredentials On
  KrbVerifyKDC Off
  KrbAuthRealms CONTOSO.COM SUBDOMAIN.CONTOSO.COM 
  Krb5KeyTab /etc/httpd/conf.d/auth_kerb.keytab
  Require valid-user
</Location>
```

## /etc/httpd/conf.d/auth_kerb.keytab
Example mod_auth_kerb keytabl for Apache2 authentication against Active Directory.
```bash
#
# This is an example of how to setup Kerberos
# authentication in Apache against CONTOSO.COM and
# SUBDOMAIN.CONTOSO.COM.  The end result works much
# like Windows Integrated Authentication.

HTTP/webserv1.contoso.com@CONTOSO.COM
HTTP/webserv1.contoso.com@SUBDOMAIN.CONTOSO.COM
```

## /etc/pam.d/system-auth
Example PAM configuration file for Kerberos 5 authentication against Active Directory Domain.
```bash
#
# This file contains the primary system-authentication rules allowing
# Kerberos authentication with PAM integrated services such as SSH.
# Also configured in this example:
# * md5 passwords
# * shadow password file
# * local authentication, in addition to Kerberos

#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 500 quiet
auth        sufficient    pam_krb5.so use_first_pass
auth        required      pam_deny.so

account     required      pam_unix.so broken_shadow
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 500 quiet
account     [default=bad success=ok user_unknown=ignore] pam_krb5.so
account     required      pam_permit.so

password    requisite     pam_cracklib.so try_first_pass retry=3
password    sufficient    pam_unix.so md5 shadow nullok try_first_pass use_authtok
password    sufficient    pam_krb5.so use_authtok
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
session     optional      pam_krb5.so
```
