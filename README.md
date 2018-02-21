# Portable Development Environment for TurnKey GNU/Linux
# Using LXC/LXD on Ubuntu 16.04 LTS

LXD is designed to work alongside LXC and provide a simpler user interface for managing LXC containers.  Currently LXD is not available in the Debian repositories and it is not likely to be included in Debian 'stretch.  This guide will document the setup of LXC/LXD on a workstation running Ubuntu 16.04 LTS.  The procedure should work for later versions of Ubuntu but it has only been tested on Xenial.

The goal of this project is to setup a local development environment for TurnKey GNU/Linux using containers on an Ubuntu 16.04 workstation.  The project makes use of the lxc-turnkey template to download and install a TurnKey appliance into an LXC (version 1) container.  Patches are applied and then the lxc-to-lxd script is used to convert to an LXD (version 2) container.  Finally, metadata and template files are added and the container is converted to an LXD image.

## Installation:

Make a user directory for development work, or use one you already have.
```bash
$ mkdir -p ~/devops
$ cd ~/devops
```
Make a directory for commonly used files.
```bash
$ mkdir -p ~/devops/files
```
Clone the TurnKey PDE from github.
```bash
$ git clone https://github.com/Dude4Linux/turnkey-pde.git
$ cd turnkey-pde
```
Run the PDE installation script.
```bash
$ ./pde-setup
```
### Testing LXD:
```bash
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
### Importing TurnKey Images:

lxc-turnkey is a template script used by lxc-create to download a TurnKey appliance image in proxmox format and create a v.1 lxc container.  LXD no longer uses template scripts but instead manages and publishes images in a new format.  A new script, lxd-turnkey, has been created for the purpose of downloading the TurnKey images and converting them to the LXD image format.  To accomplish this, it must first use the lxc-turnkey script to create a v.1 lxc container.  It then converts to a v.2 lxd container, applies metadata and templates, and then publishes the container as an LXD image.
```bash
$ lxd-turnkey --help
TurnKey LXD Image Manager Syntax: lxd-turnkey [options] -- appname

Arguments::

    appname          Appliance name (e.g., core)

Options::

    -h --help        Display this message
    -a --arch=       Appliance architecture (default: amd64)
    -v --version=    Appliance version (default: 14.2-jessie)
    -i --inithooks=  Path to inithooks.conf (default: /etc/inithooks.conf)
                     Reference: https://www.turnkeylinux.org/docs/inithooks

Example usage::

    lxd-turnkey -i /etc/inithooks.conf -- core
```
Create images for core and lamp.  Note: *sudo* may prompt for your user password midway through the script.
```bash
$ lxd-turnkey -- core
. . .
$ lxd-turnkey -- lamp
...
$ lxc image list
+-------------------------------------+--------------+--------+-----------------------------------------------+--------+----------+------------------------------+
|                ALIAS                | FINGERPRINT  | PUBLIC |                  DESCRIPTION                  |  ARCH  |   SIZE   |         UPLOAD DATE          |
+-------------------------------------+--------------+--------+-----------------------------------------------+--------+----------+------------------------------+
| turnkey-core-14.2-jessie-amd64      | 369db37c0ba3 | yes    | TurnKey core (jessie) (x86_64) (1491213770)   | x86_64 | 175.12MB | Feb 8, 2018 at 4:44am (UTC)  |
+-------------------------------------+--------------+--------+-----------------------------------------------+--------+----------+------------------------------+
| turnkey-lamp-14.2-jessie-amd64      | 4a3ea1b1269e | yes    | TurnKey lamp (jessie) (x86_64) (1493272560)   | x86_64 | 225.03MB | Feb 8, 2018 at 5:18am (UTC)  |
+-------------------------------------+--------------+--------+-----------------------------------------------+--------+----------+------------------------------+
```
### Working with LXD:

Create a new appliance based on LAMP
```bash
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
```bash
$ sudo cp /etc/inithooks.conf files/my_inithooks.conf
$ sudo chown USER:USER files/my_inithooks.conf && chmod 0700 files/my_inithooks.conf
```
Next edit my_inithooks.conf and replace the default passwords and settings with your custom values.

Push the custom inithooks file into the container.
```bash
$ lxc file push files/my_inithooks.conf mautic/etc/inithooks.conf --uid=0 --gid=0
```
Now start the container.
```bash
$ lxc start mautic
```
The turnkey-init script will run without prompts using the information supplied in inithooks.conf. When it finishes, it will delete the information in inithooks.conf leaving a single character (line-feed).  To determine when initialization is complete, you can monitor the length of inithooks.conf in the container, waiting until it is reduced to a single byte.
```bash
$ until [ $( lxc file pull mautic/etc/inithooks.conf - | wc -c ) -eq 1 ]; do sleep 10; done
```
Verify that the container hostname, mautic, can be resolved in the lxd domain.
```bash
$ host mautic.lxd
mautic.lxd has address 10.76.85.136
```
Verify that the container is running and that you can login via ssh using the password you specified in my_inithooks.conf.
```bash
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

```bash
$ lxc exec mautic /bin/bash
root@mautic ~# turnkey-version
turnkey-lamp-14.2-jessie-amd64
root@mautic ~#
...
```

Now customize your new LAMP container installing your desired application, e.g. Mautic.

### Installing TKLdev

The purpose of the TurnKey Portable Development Environment is to enable running the TKLdev appliance in a container.  This will require additional settings in the container config.  Start by downloading the TKLdev image, then create a container as described above.  Lastly, add security.privileged and security.nesting to the container.  Currently TKLdev must run in a privileged container which presents a security risk.  Make sure to follow security recommendations and protect the use of the TKLdev container.
```bash
$ lxd-turnkey -- tkldev
$ lxc init turnkey-tkldev-14.2-jessie-amd64 tkldev -c security.privileged=true -c security.nesting=true
```
Push the custom inithooks file into the container.
```bash
$ lxc file push files/my_inithooks.conf tkldev/etc/inithooks.conf --uid=0 --gid=0
```
Now start the container, and wait for initialization to complete.
```bash
$ lxc start tkldev
$ until [ $( lxc file pull tkldev/etc/inithooks.conf - | wc -c ) -eq 1 ]; do sleep 10; done
```
Enter the container and setup TKLdev.
```bash
$ lxc exec tkldev -- /bin/bash

root@tkldev ~# apt-get update --quiet
root@tkldev ~# apt-get dist-upgrade --quiet --yes
root@tkldev ~# tkldev-setup
```
Test the functionality by building the core appliance.
```bash
root@tkldev ~# cd products/core/
root@tkldev ~# make
```
Customize the TKLdev container by adding your GitHub credentials and any other development tools you desire.
Enjoy your new development environment.

### TODO

1. Currently, the output of the TKLdev build process is a product.iso.  There is no method of testing the product.iso without having a virtual machine manager such as VirtualBox.  Theoretically, it should be possible to create a v.2 lxd image directly from build/root.patched by applying changes from https://github.com/turnkeylinux/buildtasks, lxc-turnkey, lxc-to-lxd, and lxd-turnkey.  Once we can create and publish a v.2 lxd image from TKLdev, we can then create and test a container using that image.
2. LXD recommends the use of ZFS for container storage.  Update the pde-setup script to recognize if ZFS is available and if so, configure LXD to use it for default container storage.

