# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the Raspberry Pis required for running a secure Kubernetes cluster. Additional Raspberry Pis can be added to ensure the cluster is highly available.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

All network traffic between compute resources is balanced using IPVS.

### Private Network

This cluster will be deployed to a local network behind a physical router. Commodity hardware is sufficient for this tutorial. The subnet used by the router should be large to assign a private IP address to each physical node in the Kubernetes cluster. Ideally, a static address is assigned to each node in the router configuration, matching the MAC address of the Raspberry Pi network adapters to an IP.

> The `192.168.1.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

All traffic within the local network should allow internal communication across all protocols. In order to access the cluster from outside the network, the router firewall must be configured to forward traffic to the cluster.

> Setting up the router firewall is out of scope for this tutorial.

### Load Balancing the Kubernetes API

Traffic to the API server can be load balanced across multiple control plane hosts to achieve high availability.

> Setting up a load balancer for a highly available control plane is out of scope for this tutorial.

### Kubernetes Public IP Address

The IP address allocated by an ISP to your local network would be used as the public facing IP. All traffic to this IP would be routed to the load balancer to then distribute requests across the control plane hosts in a highly available control plane configuration.

> Setting up a public facing IP address for the Kubernetes API is out of scope for this tutorial.

## Compute Instances

The compute instances in this lab will be provisioned using [Debian 11 (Bullseye) Images](https://raspi.debian.net/tested-images/), which have been built for the Raspberry Pi. 

Note: Debian 11 has support for `cgroups v2` since it supports `systemd version 244` or later. Older `systemd` versions do not support delegation of the `cpuset` controller which is important when imposing resource limitations on rootless containers.

> This tutorial does not leverage rootless containers.

### Download a Debian Raspberry Pi Image

```
curl -O -L "https://raspi.debian.net/verified/20210823_raspi_4_bullseye.img.xz"
```

### Flash the SD Card

Once the SD card is plugged into the laptop, confirm the disk number assigned to this device (e.g. `/dev/disk2`).

```
xzcat 20210823_raspi_4_bullseye.img.xz | sudo dd of=/dev/disk2 bs=64k
```

### Update Image Configuration

```
vim /Volumes/RASPIFIRM/sysconf.txt
# Uncomment and update the root_autherized_key value (e.g. pbcopy < ~/.ssh/id_ed25519.pub).
# Uncomment and update the hostname value (e.g. controller-0).
```

### Unmount the SD Card and Repeat

```
sudo diskutil unmount /Volumes/RASPIFIRM
```

Repeat this process for all Raspberry Pis.

## Configuring SSH Access

SSH access as the root user should be available since the image configuration contains the public SSH key of the laptop. Updating `/etc/hosts` might be required to access the Raspberry Pis depending on your network configuration.

Test SSH access to the `controller-0` Raspberry Pi:

```
ssh root@controller-0
```

Type `exit` at the prompt to exit the `controller-0` Raspberry Pi:

```
root@controller-0:~$ exit
```

> Output

```
logout
Connection to controller-0 closed.
```

## Preparing the Base OS

The Raspberry Pi images used are not fully equipped to run Kubernetes out of the box. The following steps will prepare the OS by cleaning up the `dmesg` logs, setup a regular user and ensure configuration that applies to all hosts is applied. Unfortunately, this is a manual process, for now!

The following is assumed to be done as the root user on all machines.

### Update the OS Packages

```
apt update
apt upgrade
```

### Check `dmesg` for Errors

```
dmesg
```

### Hardware Device Drivers

Since `git` and `curl` are not available on the base Debian image, grab the necessary files using your laptop,

#### Regulatory Database for Wireless Adapters

```
git clone https://kernel.googlesource.com/pub/scm/linux/kernel/git/sforshee/wireless-regdb
cd wireless-regdb
git checkout <latest-release-tag>  # e.g. master-2020-11-20
```

Copy out the files we need for convenience,

```
cp regulatory.db* ..
```

#### Bluetooth Firmware

```
curl -O -L "https://github.com/armbian/firmware/raw/master/BCM4345C5.hcd"
curl -O -L "https://github.com/armbian/firmware/raw/master/BCM4345C0.hcd"
curl -O -L "https://github.com/armbian/firmware/raw/master/brcm/brcmfmac43455-sdio.clm_blob"
```

#### Copy to the Pis

```
for host in controller-0 node-0 node-1 node-2
  do scp regulatory.db regulatory.db.p7s BCM4345C5.hcd BCM4345C0.hcd brcmfmac43455-sdio.clm_blob root@$host:/root
done
```

Now on the Raspberry Pis, we can put the firmware in the correct location,

```
mv /root/regulatory.db* /lib/firmware
```

```
mv /root/BCM4345C* /lib/firmware/brcm
mv /root/brcmfmac43455-sdio.clm_blob /lib/firmware/brcm
```

#### Validate

Reboot all nodes and then check the `dmesg` output again to verify all errors have been resolved.

### Setup Locales

```
apt install locales
dpkg-reconfigure locales
```

### Install `sudo`

```
apt install sudo
```

### Set `vi` as the Default Editor

```
update-alternatives --config editor
```

### Update `sudo` Configuration to Allow No Password

```
visudo
# Update the following line with "NOPASSWD"
- %sudo   ALL=(ALL:ALL) ALL
+ %sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

### Update `/etc/hosts`

#### Update the Hostname

```
# Example entries to update.
127.0.0.1       controller-0.localdomain controller-0
::1             controller-0.localdomain controller-0 ip6-localhost ip6-loopback
```

#### Add the Cluster IPs

An example list of hosts,

```
# Kubernetes Cluster
192.168.1.110   controller-0
192.168.1.120   node-0
192.168.1.121   node-1
192.168.1.122   node-2
```

### Disable `swap`

By default, the Debian image used for the Raspberry Pis doesn't use swap. To confirm,

```
cat /proc/swaps
```

### Enable cgroups

By default, the Debian image used for the Raspberry Pis has all required cgroups enabled. To confirm,

```
cat /proc/cgroups
```

### Enable Bridge Networking in the Kernel

Update the loaded kernel modules configuration,

```
vi /etc/modules-load.d/modules.conf
```

Add the following line to this file,

```
br_netfilter
```

### Enable IPVS in the Kernel

Install the support programs to interface with IPVS,

```
apt install ipvsadm ipset conntrack
```

Update the loaded kernel modules configuration,

```
vi /etc/modules-load.d/modules.conf
```

Add the following lines to this file,

```
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
```

### WIP: Possibly only needed on Worker nodes.

#### Enable Overlay Networking in the Kernel

Update the loaded kernel modules configuration,

```
vi /etc/modules-load.d/modules.conf
```

Add the following lines to this file,

```
overlay
```

#### Enable IP Forwarding in the Kernel

Update the kernel module parameters configuration,

```
vi /etc/sysctl.d/local.conf
```

Add the following to this file,

```
net.ipv4.ip_forward = 1
```

#### Install the Container Runtime (crun)

```
(
  export VERSION="1.0"
  curl -O -L "https://github.com/containers/crun/releases/download/1.0/crun-${VERSION}-linux-arm64"
  chmod +x "crun-${VERSION}-linux-arm64"
  mv "crun-${VERSION}-linux-arm64" /usr/local/bin/crun
)
```

#### Install the Container Networking Interface Plugins 
 
Setup the installation directory,

``` 
mkdir -p /opt/cni/bin 
``` 

Download and install the plugins,

``` 
(
  export VERSION="1.0.1" 
  curl -O -L "https://github.com/containernetworking/plugins/releases/download/v${VERSION}/cni-plugins-linux-arm64-v${VERSION}.tgz"
  tar -xvf "cni-plugins-linux-arm64-v${VERSION}.tgz" -C /opt/cni/bin/
  rm "cni-plugins-linux-arm64-v${VERSION}.tgz"
)
``` 
 
#### Install the Container Runtime Interface (cri-o)

##### Add the cri-o Package Repositories

```
(
  export VERSION=1.21
  export OS=Debian_Testing
  
  echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
  
  curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/${OS}/Release.key | apt-key add -
)
```

**Note:** Since cri-o for `aarch64` is not published to the Debian_11 repository, xUbuntu_20.04 is used instead.
**Note:** xUbuntu_20.10 uses a newer version of `glibc` and thus is incompatible with Debian 11.

```
(
  export VERSION=1.21
  export OS=xUbuntu_20.04
  
  echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/${VERSION}/${OS}/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.list
  
  curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/${OS}/Release.key | apt-key add -
  curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/Release.key | apt-key add -
)
```

Update the repositories,

```
apt update
```

##### Install the Container Runtime Interface

```
apt install cri-o
```

##### Enable and Start the Container Runtime Interface

```
systemctl daemon-reload
systemctl enable cri-o.service
systemctl start cri-o.service
```

### Configure a Regular User

#### Add the User

```
adduser nerditup
usermod –a –G sudo nerditup
```

**Note:** Login as the regular user for the following,

#### Generate an SSH Key

```
ssh-keygen -t ed25519
```

#### Distribute SSH Keys to Each Host

```
# Example loop for controller-0.
for i in node-0 node-1 node-2; do ssh-copy-id nerditup@$i; done
```

### WIP: Possibly able to remove the following configuration.

#### Enable `iptables` Filtering on the Bridge Network

TODO: Try and remove this requirement: https://wiki.libvirt.org/page/Net.bridge.bridge-nf-call_and_sysctl.conf

> the bridge module in the kernel has the default for all three of these values set to "1"

Update the relevant kernel parameters,

```
vi /etc/sysctl.d/local.conf
```

Add the following lines to this filem

```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

---

Finally, reboot all machines.

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
