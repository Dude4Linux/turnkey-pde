# Installing TKLdev

The purpose of the TurnKey Portable Development Environment is to enable running the TKLdev appliance in a container.  This will require additional settings in the container config.  Start by downloading the TKLdev image, then create a container as described in README.md.  Lastly, add security.privileged and security.nesting to the container.  Currently TKLdev must run in a privileged container which presents a security risk.  Make sure to follow security recommendations and protect the use of the TKLdev container.
```bash
  $ lxd-turnkey -- tkldev
  $ lxc init turnkey-tkldev-14.2-jessie-amd64 tkldev -c security.privileged=true -c security.nesting=true
```
Push the custom inithooks and *proxy files into the container.
```bash
  $ lxc file push files/my_inithooks.conf tkldev/etc/inithooks.conf --uid=0 --gid=0
  $ lxc file push files/01proxy tkldev/etc/apt/apt.conf.d/01proxy --uid=0 --gid=0
```
 \* assuming you have installed [apt-cacher-ng](apt-cacher-ng.md) on the host.

Now start the container, and wait for initialization to complete.
```bash
  $ lxc start tkldev
  $ until [ $( lxc file pull $1/etc/inithooks.conf - | wc -c ) -eq 1 ]; do sleep 10; echo -n "."; done
```
Enter the container and setup TKLdev.
```bash
  $ lxc exec tkldev -- /bin/bash

  root@tkldev ~# apt-get update --quiet
  root@tkldev ~# apt-get dist-upgrade --quiet --yes
  root@tkldev ~# tkldev-setup
```
Change the FAB_APT_PROXY in the container's /root/.bashrc.d/fab to use [apt-cacher-ng](apt-cacher.ng.md)
```bash
  # the proxy address is the same as the default gateway
  PROXY="http://$(ip route | awk '/default/ { print $3 }'):3142"
  sed -i "/FAB_APT_PROXY/ s|=.*$|=${PROXY}|" /root/.bashrc.d/fab
```
Test the functionality by building the core appliance.
```bash
  root@tkldev ~# cd products/core/
  root@tkldev products/core# make
```
At the conclusion of the make build process, a product.iso file and several directories will exist in the build directory.
```bash
  root@tkldev products/core# ls build
  bootstrap    root.build    root.spec
  cdroot       root.patched  stamps
  product.iso  root.sandbox
```
### Installing lxd-make-image
In our portable development environment, we've been able to build a product.iso for core, or any other TurnKey appliance.  Unfortunately testing the iso file would require a virtual machine to install and run the product.iso.  We would prefer to be able to test our new appliance in another container using an LXD format image.  Until now, we would have had to wait until the appliance was accepted by the TurnKey admins, and released as a 'proxmox' formatted image.  Rapid development cycles require that we have a means of immediately testing a newly built appliance, preferably one that can be automated by an orchestration engine such as Ansible.  Fortunately there is now lxd-make-image, a script which uses the result of the make build process contained in build/root.patched to directly create an LXD formatted image.  This image file can be imported into the host's image store and then used to create containers running the TurnKey appliance.  The power of Ansible can then be used to run a series of pre-programmed tests on the target container.

On the host, run
```bash
  $ lxc file push turnkey-pde/lxd-make-image tkldev/usr/bin/ --uid=0 --gid=0
```
### Creating an LXD container image
After completing the make build in the TKLdev container, run
```bash
  root@tkldev products/core# lxd-make-image
```
At the completion of the process, you should find the LXD image in the build directory e.g. build/turnkey-core-14.2-jessie-amd64.tar.xz.  From there the image must be moved to the host and imported into the LXD image storage.

On the host, run
```bash
  $ lxc file pull tkldev/turnkey/fab/products/core/build/turnkey-core-14.2-jessie-amd64.tar.xz /tmp
  $ lxc image import /tmp/turnkey-core-14.2-jessie-amd64.tar.xz --alias turnkey-core-14.2-jessie-amd64 --public=true
  $ rm -f /tmp/turnkey-core-14.2-jessie-amd64.tar.xz
```
### Creating a container from LXD image
Once your appliance image has been created and imported, you can then launch a running container.  I recommend using init instead of launch which allows pre-seeding the container with your custom inithooks and proxy settings.
```bash
  $ lxc init turnkey-core-14.2-jessie-amd64 core-test
  $ lxc file push files/my_inithooks.conf core-test/etc/inithooks.conf --uid=0 --gid=0
  $ lxc file push files/01proxy core-test/etc/apt/apt.conf.d/01proxy --uid=0 --gid=0
  
  $ lxc start core-test
  $ until [ $( lxc file pull $1/etc/inithooks.conf - | wc -c ) -eq 1 ]; do sleep 10; echo -n "."; done # wait for initialization
```
### Caveats
There a couple of problems you should be aware of.  Hopefully they will be corrected soon.

1. Containers created with SystemD enabled will not launch the TurnKey initialization automatically on start. For now, you will have to run turnkey-init in the container.
2. LXD images created on the stretch version of TKLdev cannot be imported into the image store. More on that later.

### Finis
Customize the TKLdev container by adding your GitHub credentials and any other development tools you desire.
Enjoy your new development environment.
