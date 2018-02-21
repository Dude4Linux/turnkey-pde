# Add Apt Cache to PDE

When working remotely with a limited bandwidth available, it is important to minimize as much as possible the repeated downloading of Debian packages via Apt.  One way of doing this is to add an Apt Cache Proxy to the host running the TurnKey GNU/Linux Portable Development Environment (PDE). On the other hand, we wish to avoid multiple caches of the same file. The LXC appliance caches downloaded proxmox formated images. The TKLdev appliance, by default, uses Polipo to cache all downloaded files including deb packages.

### 1) Choosing an Apt Cache Proxy

a) squid-deb-proxy
```
  • Installs squid3 and sets up two proxies, one for HTTP and one for Apt
  • Could not get the apt proxy to accept PPA or TurnKey packages
```
b) polipo
```
  • Lightweight proxy used by TKLdev appliance
  • Support for polipo has been discontinued https://www.irif.fr/~jch//software/polipo/
```
c) apt-cacher-ng
```
  • Next generation replacing apt-cacher
  • This is the package we choose to use
```
### 2) Install apt-cacher-ng
```bash
  sudo apt-get -qy update
  sudo apt-get -qy install -t xenial-backports apt-cacher-ng
```
### 3) Create 01proxy and Install on all clients
Set the proxy for host to localhost.
```bash
  echo "Acquire::http { Proxy "http://127.0.0.1:3142"; };" | sudo tee /etc/apt/apt.conf.d/01proxy
```
Set the proxy for containers to the LXD bridge interface.
```bash
  PROXY=$(lxc network get lxdbr0 ipv4.address)
  PROXY="http://${PROXY%/[0-9]*}:3142"
  echo "Acquire::http { Proxy "${PROXY}"; };" > files/01proxy
```
For each container, push the 01proxy file and restart apt if the container is running
```bash
  for container in $(lxc list --format=csv -cn); do
    lxc file push files/01proxy ${container}/etc/apt/apt.conf.d/01proxy --uid=0 --gid=0
  done
```
### 4) Ensure clients only use http:// urls in source lists.
apt-cacher-ng cannot cache https:// urls.  We will configure apt-cacher-ng to forward all requests, but if possible you should replace any https:// urls with the equivalent http:// url.

### 5) Configure apt-cacher-ng to pass through HTTPS requests
add the following line to /etc/apt-cacher-ng/acng.conf
```bash
  PassThroughPattern: .* # this will allow CONNECT to everything including HTTPS
```
and then restart
```bash
  sudo service apt-cacher-ng restart
```
### 6) Configure firewall to allow containers to access apt-cacher-ng
```bash
  sudo ufw allow in on lxcbr0 to any port 3142 proto tcp
  sudo ufw allow in on lxdbr0 to any port 3142 proto tcp
```
### 7) The TKLdev appliance needs some additional configuration
Change the FAB_APT_PROXY in the container's /root/.bashrc.d/fab to use apt-cacher-ng
```bash
  lxc exec tkldev -- sed -i "/FAB_APT_PROXY/ s|=.*$|=${PROXY}|" /root/.bashrc.d/fab
```
where 'tkldev' is the name of the TKLdev container.
