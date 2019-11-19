# Managing security 

* Every daemon that provides a network service increases the risk of a successful remote attack, so you should not run unnecessary services.
* You should not allow root to directly log in to the system using ssh. Instead, require initial login to an unprivileged account that can use sudo or su to become root.
* You should consider turning off password-based SSH access and require either key-based authentication or Kerberos for remote logins.

## CVE and updates

> yum updateinfo --security

> yum updateinfo list updates | grep Critical

> yum updateinfo RHSA-2018:1453

> yum updateinfo list --cve CVE-2018-1111

> yum update --cve CVE-2018-1111


## SECURING SERVICES

```
[root@machine ~]$ ss -tlw
Netid  State   Recv-Q  Send-Q    Local Address:Port          Peer Address:Port  
icmp6  UNCONN  0       0                     *:ipv6-icmp                *:*     
tcp    LISTEN  0       32        192.168.122.1:domain             0.0.0.0:*     
tcp    LISTEN  0       5             127.0.0.1:ipp                0.0.0.0:*     
tcp    LISTEN  0       128           127.0.0.1:postgres           0.0.0.0:*     
tcp    LISTEN  0       5                 [::1]:ipp                   [::]:*     
tcp    LISTEN  0       128               [::1]:postgres              [::]:*     
tcp    LISTEN  0       128                   *:websm                    *:*  
```

> ssh-keygen

> ssh-copy-id user@remote


### ssh config change

Modify the /etc/ssh/sshd_config file

```
PermitRootLogin no
PasswordAuthentication no
```

> [root@demo ~]# systemctl reload sshd

### sudoers

> visudo

> user ALL=(ALL) ALL

> user ALL=(ALL) NOPASSWD:ALL

### Pass 

Force immediate password expiration by running the following command as root:

> chage -d 0 username


# Selinux users and roles

Create a new user that is mapped to guest_u (i.e., no internet, no sudo/su or most other setuid/setgid apps, no X)

> useradd -Z guest_u newuserbob

> userdel -Z -r newuserbob

Make it so guest_u & xguest_u won't be allowed to execute anything in /tmp or $HOME

> setsebool -P guest_exec_content=off xguest_exec_content=off

```sh
semanage boolean -l | grep exec_content
auditadm_exec_content          (on   ,   on)  Allow auditadm to exec content
guest_exec_content             (off  ,  off)  Allow guest to exec content
dbadm_exec_content             (on   ,   on)  Allow dbadm to exec content
xguest_exec_content            (off  ,  off)  Allow xguest to exec content
secadm_exec_content            (on   ,   on)  Allow secadm to exec content
logadm_exec_content            (on   ,   on)  Allow logadm to exec content
user_exec_content              (on   ,   on)  Allow user to exec content
staff_exec_content             (on   ,   on)  Allow staff to exec content
sysadm_exec_content            (on   ,   on)  Allow sysadm to exec content
```

Confine an existing user, mapping to user_u (i.e., no su/sudo or most other setuid/setgid apps)

> semanage login -a -s user_u existinguseralice



## Folders for services

```sh
[root@serverd ~]# ls -Zd /var/log/httpd
drwx------. root root system_u:object_r:httpd_log_t:s0 /var/log/httpd
```
Use the semanage fcontext command to add the new rule for /custom/httpd_log .
```sh
[root@serverd ~]# semanage fcontext -a -t httpd_log_t \> '/custom/httpd_logs(/.*)?'[root@serverd ~]# 
```
Remember to restore the context of the /custom/httpd_logs directory.
```sh
[root@serverd ~]# restorecon -Rv /custom/httpd_logs
restorecon reset /custom/httpd_logs context unconfined_u:object_r:default_t:s0->unconfined_u:object_r:httpd_log_t:s0
```

 Change the default mapping between the Linux and the SELinux users. Map the Linux users to the user_u SELinux user.

[root@serverd ~]# semanage login -m -s user_u -r s0 __default__[root@serverd ~]# 

To prevent users from running programs in /tmp or their home directory, set the SELinux user_exec_content Boolean to off .

[root@serverd ~]# setsebool -P user_exec_content off


## Summary

> semanage  (port/fcontext/login) -l

Note that system_u is a special user identity for system processes and objects. It must never be associated to a Linux user. Also, unconfined_u and root are unconfined users. For these reasons, they are not included in the aforementioned table of SELinux user capabilities.
Alongside with the already mentioned SELinux users, there are special roles, that can be mapped to those users. These roles determine what SELinux allows the user to do:

* webadm_r can only administrate SELinux types related to the Apache HTTP Server. See Section 14.2, “Types” for further information.
* dbadm_r can only administrate SELinux types related to the MariaDB database and the PostgreSQL database management system. See Section 21.2, “Types” and Section 22.2, “Types” for further information.
* logadm_r can only administrate SELinux types related to the syslog and auditlog processes.
* secadm_r can only administrate SELinux.
* auditadm_r can only administrate processes related to the audit subsystem. 

# LUKS and NBDE

When performing automated installations, Kickstart can create encrypted block devices. For automated partitioning, you can specify:

> autopart --type=lvm --encrypted --passphrase=PASSPHRASE

If you are configuring specific disk partitions, you must specify the --encrypted and --
passphrase options for each partition to be encrypted. 

> part /home --fstype=ext4 --size=10000 --onpart=vda2 --encrypted --passphrase=PASSPHRASE

Similar syntax works for LVM physical volume:

> part pv.01 --size=10000 --encrypted --passphrase=PASSPHRASE

Note that the passphrase, PASSPHRASE, is stored in the Kickstart profile in plain text, and so
the Kickstart profile must be secured. If you omit the --passphrase option then the installer
prompts for the passphrase during installation.


> cryptsetup luksFormat /dev/vdb1
> cryptsetup luksDump /dev/vdb1

The following example decrypts the /dev/vdb1 device and maps it to the example logical device-
mapper device. To decrypt the partition, the cryptsetup luksOpen command prompts for the
passphrase used to encrypt it.

> [root@demo ~]# cryptsetup luksOpen /dev/vdb1 example

> cryptsetup luksClose example


## Tang server

Associate the LUKS-encrypted partition available on /dev/vdb1 with the Tang servers on serverc and serverd . Configure SSS encryption so that at least two Tang servers must be available to decrypt the partition.

Install the packages required to configure serverb as a Clevis client.

> [root@serverb ~]# yum install clevis clevis-luks clevis-dracut
> ...output omitted...

Associate the LUKS-encrypted partition available on /dev/vdb1 with the Tang servers on serverc , and serverd . Configure the SSS encryption so that the two Tang servers must be available to decrypt the partition.

> [root@serverb ~]# cfg=$'{"t":2,"pins":{"tang":[\n> {"url":"http://serverc.lab.example.com"},\n> {"url":"http://serverd.lab.example.com"}]}}'

> [root@serverb ~]# clevis luks bind -d /dev/vdb1 sss "$cfg"

 Enable clevis-luks-askpass.path to support non-root LUKS-encrypted partitions.

```sh
[root@serverb ~]# systemctl enable clevis-luks-askpass.path
Created symlink from /etc/systemd/system/remote-fs.target.wants/clevis-luks-askpass.path to /usr/lib/systemd/system/clevis-luks-askpass.path.
```

# USBGuard

> sudo yum install usbguard

> sudo yum install usbutils udisks2

Rule Targets
The target of a rule specifies whether the device will be authorized for use or not. Three types of
target are recognized:
allow
Authorize the device. The device and its interfaces will be allowed to communicate with the
system.
block
Do not authorize the device. The device is visible to the system but will remain in a blocked
state until it is authorized.
reject
Deauthorize and remove the device from the system. The device will have to be re-inserted to
become visible to the system again.


> usbguard generate-policy > /etc/usbguard/rules.conf
> systemctl restart usbguard

```
usbguard list-devices
usbguard allow-device 6
usbguard list-devices

usbguard allow-device -p 6
systemctl restart usbguard

usbguard block-device ID
usbguard reject-device ID

usbguard add-user -g usbguard \
> --devices=modify,list,listen --policy=list --exceptions=listen
```


# PAM

It is highly recommended to configure PAMs using the authconfig tool instead of manually editing the PAM configuration files. 

The man -k pam_ command returns a long list of related man pages.

## switch to SSSD

> yum -y install sssd
...output omitted...
> authconfig --enablesssd --enablesssdauth --update

## Summary 

• PAM stores most of its configuration files in /etc/pam.d/.
• A PAM-enabled application invokes the rules in each management group, auth, account,
password, and session, at different times during the user authentication and authorization
process.
• The authconfig command is the recommended way of updating the PAM configuration.
• Before any modification, back up the PAM configuration with authconfig --
savebackup=backupdir and open an extra root session to recover from errors.
• The pam_pwquality module uses the /etc/security/pwquality.conf configuration file
to enforce your organization password complexity requirements.
• The pam_faillock module locks accounts after too many consecutive failed attempts. You
use the authconfig --enablefaillock --faillockargs="parameters" command to
configure it.


# Auditd

> systemctl status auditd

```
vi /etc/audit/auditd.conf
...output omitted...
log_format = ENRICHED
flush = INCREMENTAL_ASYNC
freq = 50
name_format = HOSTNAME
...output omitted...
```

> service auditd restart

> tail /var/log/audit/audit.log

## Remote client

Configure the Audit service on servera to send audit messages to the Audit service on serverb.

> yum install audispd-plugins

In the /etc/audisp/plugins.d/au-remote.conf file, set the value for the active option to yes to enable remote logging.

```
vi /etc/audisp/plugins.d/au-remote.conf
...output omitted...
active = yes
...output omitted...
```

In the /etc/audisp/audisp-remote.conf file, set the remote_server option to the IP address of the remote logging server in our environment

serverb.lab.example.com. Set the port to be used on the remote logging server, which is 60 by default.

```
vi /etc/audisp/audisp-remote.conf
...output omitted...
remote_server = 172.25.250.11
port = 60
...output omitted...
```

Restart the auditd service to update its configuration. When done, log off from servera.

> service auditd restart

## Remote server

```
vi /etc/audit/auditd.conf
...output omitted...
tcp_listen_port = 60
...output omitted...
```

Open TCP port 60 to enable access to the Audit server.

> firewall-cmd --zone=public --add-port=60/tcp --permanent


> firewall-cmd --reload


Restart the auditd service to update its configuration. When done, log off from serverb.

> service auditd restart

## Reporting

