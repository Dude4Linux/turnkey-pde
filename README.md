# Portable Development Environment for TurnKey GNU/Linux
# Using LXC/LXD on Ubuntu 16.04 LTS

LXD is designed to work alongside LXC and provide a simpler user interface for managing LXC containers.  Currently LXD is not available in the Debian repositories and it is not likely to be included in Debian 'stretch.  This guide will document the setup of LXC/LXD on a workstation running Ubuntu 16.04 LTS.

The goal of this project is to setup a local development environment for TurnKey GNU/Linux using containers on an Ubuntu 16.04 workstation.  The project makes use of the lxc-turnkey template to download and install a TurnKey appliance into an LXC (version 1) container.  Patches are applied and then the lxc-to-lxd script is used to convert to an LXD (version 2) container.  Finally, metadata and template files are added and the container is converted to an LXD image.

## Installation:

Make a user directory for development work, or use one you already have. 
```
$ mkdir -p ~/devops
$ cd ~/devops
```
Clone the TurnKey PDE from github.
```
$ git clone 
$ cd turnkey-pde
```
Run the PDE installation script.
```
$ ./pde-setup
```
### Testing LXD:
```
$ lxc image list
+-------+-------------+--------+-------------+------+------+-------------+
| ALIAS | FINGERPRINT | PUBLIC | DESCRIPTION | ARCH | SIZE | UPLOAD DATE |
+-------+-------------+--------+-------------+------+------+-------------+

$ lxc image copy lxc-org:/debian/jessie/amd64 local: --alias=jessie-amd64
Image copied successfully!

$ lxc image list
+--------------+--------------+--------+----------------------------------------+--------+---------+-----------------------------+
|    ALIAS     | FINGERPRINT  | PUBLIC |              DESCRIPTION               |  ARCH  |  SIZE   |         UPLOAD DATE         |
+--------------+--------------+--------+----------------------------------------+--------+---------+-----------------------------+
| jessie-amd64 | 06b11f7b270f | no     | Debian jessie (amd64) (20170104_22:42) | x86_64 | 94.05MB | Jan 5, 2017 at 5:40pm (UTC) |
+--------------+--------------+--------+----------------------------------------+--------+---------+-----------------------------+

$ lxc launch jessie-amd64 jessie-01
Creating jessie-01
Starting jessie-01

$ lxc list
+-----------+---------+---------------------+------+------------+-----------+
|   NAME    |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+-----------+---------+---------------------+------+------------+-----------+
| jessie-01 | RUNNING |                     |      | PERSISTENT | 0         |
+-----------+---------+---------------------+------+------------+-----------+

$ lxc stop jessie-01
$ lxc list
+-----------+---------+---------------------+------+------------+-----------+
|   NAME    |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+-----------+---------+---------------------+------+------------+-----------+
| jessie-01 | STOPPED |                     |      | PERSISTENT | 0         |
+-----------+---------+---------------------+------+------------+-----------+

$ lxc delete jessie-01
$ lxc list
+-----------+---------+---------------------+------+------------+-----------+
|   NAME    |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+-----------+---------+---------------------+------+------------+-----------+
```
### Importing TurnKey Image:


### Working with LXD:

Create a new appliance based on LAMP
```
$ lxc init turnkey-lamp-14.2-jessie-amd64 mautic
Creating mautic

$ lxc list
+--------------+---------+---------------------+------+------------+-----------+
|     NAME     |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+--------------+---------+---------------------+------+------------+-----------+
| mautic       | STOPPED |                     |      | PERSISTENT | 0         |
+--------------+---------+---------------------+------+------------+-----------+
```
Notice that 'init' creates the new container leaving it stopped, where 'launch' both creates and starts the container. In this case, we want to insert a new inithooks.conf file before starting so we used 'init'.

Make a customized local copy of inithooks.conf in your working directory. We'll call it my_inithooks.conf. Make sure it is owned and executable only by you.
```
$ sudo cp /etc/inithooks.conf my_inithooks.conf
$ sudo chown USER:USER my_inithooks.conf && chmod 0700 my_inithooks.conf
```
Next edit my_inithooks.conf and replace the default passwords and settings with your custom values.

Push the custom inithooks file into the container.
```
$ lxc file push my_inithooks.conf mautic/etc/inithooks.conf
```
Now start the container, and run turnkey-init.
```
$ lxc start mautic
$ lxc exec mautic turnkey-init
```
The turnkey-init script will ask if you want warning messages forwarded to your admin account. When it finishes, it will display a summary of services, addresses and ports. Press enter to Quit and you will be returned to the host prompt.

Verify that the container hostname, mautic, can be resolved in the lxd domain.
```
$ host mautic.lxd
mautic.lxd has address 10.76.85.136
```
Verify that the container is running and that you can login via ssh using the password you specified in my_inithooks.conf.
```
$ lxc list
+--------------+---------+---------------------+------+------------+-----------+
|     NAME     |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+--------------+---------+---------------------+------+------------+-----------+
| mautic       | RUNNING | 10.76.85.136 (eth0) |      | PERSISTENT | 0         |
+--------------+---------+---------------------+------+------------+-----------+

$ slogin root@mautic.lxd
The authenticity of host 'mautic.lxd (10.76.85.136)' can't be established.
ECDSA key fingerprint is SHA256:qz0jyckG3iwHYchzONMPQdNxvlvohCmWRzCu8Kt9/Pw.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'mautic.lxd' (ECDSA) to the list of known hosts.

root@mautic.lxd's password: 
Linux lamp 4.4.0-78-generic #99-Ubuntu SMP Thu Apr 27 15:29:09 UTC 2017 x86_64
Welcome to Lamp, TurnKey GNU/Linux 14.2 / Debian 8.7 Jessie
```
Connect to the container using the 'exec' command.

```
$ lxc exec mautic /bin/bash
root@mautic ~# turnkey-version 
turnkey-lamp-14.2-jessie-amd64
root@mautic ~# 
...
```

Now customize your new LAMP container installing your desired application, e.g. Mautic.
