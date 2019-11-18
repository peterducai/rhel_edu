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