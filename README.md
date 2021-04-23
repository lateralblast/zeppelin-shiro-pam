![alt tag](https://raw.githubusercontent.com/lateralblast/zeppelin-shiro-pam/master/cat_zeppelin.png)

Zeppelin Shiro PAM
==================

Zeppelin authentication via PAM - Shiro configuration

Introduction
------------

The instructions for configuring PAM authentication for Zeppelin via Shiro are not very clear.
These instructions are designed as an attempt to make the process easier.

These instructions are for the default OOTB Zeppelin install using HTTP on port 8080.
You should enable and configure HTTPS. Details for doing this are not included here.

I have not been able to find a method for mapping alternate groups to user and admin like can be done with LDAP.

Using PAM requires the user Zeppelin is running as to have read access to the shadow file,
therefore I would recommend using OpenLDAP as solution, to avoid a potential XSS.

If users don't need system access or to upload files manually for processing,
then you could set the user shell to /bin/false to restrict access file read.
Having said that, you should lock down top level access to valid users anyway.

Requirements
------------

Software:

- Zeppelin
- PAM

Configuration:

- A user for the zeppelin server to run as e.g. zepserver
- A group for Zeppelin users
  - a group name user with a group id (GID) e.g. 2001
- A group for Zeppelin admins
  - a group name admin with a group id (GID) e.g. 2002
- Zeppelin users and ids
  - A user name e.g. zepuser with a user id (UID) e.g. 2001
- Zeppelin admins and ids
  - A user name e.g. zepadmin with a user id (UID) e.g. 2002

Instructions
------------

Create users and groups:

```
$ sudo groupadd user
$ sudo groupadd admin
$ sudo useradd -s /bin/bash -d /home/zepserver -m -G user zepserver
$ sudo useradd -s /bin/bash -d /home/zepuser -m -G user zepuser
$ sudo useradd -s /bin/bash -d /home/zepadmin -m -G admin zepadmin
$ sudo passwd zepuser
$ sudo passwd zepadmin
```

Example group file entries:

```
user:x:2001:zepuser
admin:x:2002:zepadmin
zepuser:x:1001:
zepadmin:x:1002:
```

Example passwd file entries:

```
zepuser:x:1001:1001::/home/zepuser:/bin/bash
zepadmin:x:1002:1002::/home/zepadmin:/bin/bash
```

Install Zeppelin:

```
$ wget https://downloads.apache.org/zeppelin/zeppelin-0.9.0/zeppelin-0.9.0-bin-all.tgz 
$ tar -xpf zeppelin-0.9.0-bin-all.tgz
```

Copy shiro config template and edit it:

```
$ cp ./zeppelin-0.9.0-bin-all/conf/shiro.ini.template ./zeppelin-0.9.0-bin-all/conf/shiro.ini
$ vi ./zeppelin-0.9.0-bin-all/conf/shiro.ini
```

Comment out user entries in shiro.ini, e.g.:

```
[users]
# List of users with their password allowed to access Zeppelin.
# To use a different strategy (LDAP / Database / ...) check the shiro doc at http://shiro.apache.org/configuration.html#Configuration-INISections
# To enable admin user, uncomment the following line and set an appropriate password.
#admin = password1, admin
#user1 = password2, role1, role2
#user2 = password3, role3
#user3 = password4, role2
```

Allow zeppelin server user to read shadow file (required for PAM authentication to work):

```
$ sudo apt install -y acl
$ sudo setfacl -m user:zepserver:r /etc/shadow
```

Add PAM entries to main section in shiro.ini, e.g.:

```
[main]
### A sample PAM configuration
pamRealm = org.apache.zeppelin.realm.PamRealm
pamRealm.service = sshd
securityManager.realms = $pamRealm
```

Have admin entry in roles section in shiro.ini, e.g.:

```
[roles]
zepadmins = *
```

Make sure you include groups in roles in urls section in shiro.ini, e.g.:

```
[urls]
/api/version = anon
/api/cluster/address = anon
# Allow all authenticated users to restart interpreters on a notebook page.
# Comment out the following line if you would like to authorize only admin users to restart interpreters.
#/api/interpreter/setting/restart/** = authc
/api/interpreter/** = authc, roles[zepadmins]
/api/notebook-repositories/** = authc, roles[zepadmins]
/api/configurations/** = authc, roles[zepadmins]
/api/credential/** = authc, roles[zepadmins]
/api/admin/** = authc, roles[zepadmins]
#/** = anon
/** = authc
```

The authc entry above allows anyone with a valid user account and password on the system to have user access via PAM.
The roles entry allows only people in the zepadmins group to have administrative access to administrative URLS.

License
-------

This documentation is licensed as CC-BA (Creative Commons By Attrbution)

http://creativecommons.org/licenses/by/4.0/legalcode
