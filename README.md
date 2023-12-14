# kc  ... kvm controller 

## Description

kc environment contains a couple of bash scripts to install and manage kvm  
from a central host and directory (master).

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 22.04
 * bare metal with VT support

## Installation
### minimal ubuntu-22.04 server
```
apt-get purge -y snapd
apt-get update
apt-get dist-upgrade -y
apt-get autoremove -y
apt-get install -y vim

apt-get install -y locales iputils-ping net-tools rsync vim
dpkg-reconfigure locales
locale-gen en_US.UTF-8
```
### fix ip, hostname,...

https://www.linuxtechi.com/static-ip-address-on-ubuntu-server/

```
vi /etc/hosts
vi /etc/hostname
chmod 600  /etc/netplan/00-installer-config.yaml 
# apply ssh-keys
vi /etc/netplan/00-installer-config.yaml
netplan generate ; netplan apply
vi /root/.ssh/authorized_keys
netplan generate
netplan apply   # WARNING:root:Cannot call Open vSwitch: ovsdb-server.service is not running.
```

### kvm/libvirt

https://www.linuxtechi.com/how-to-install-kvm-on-ubuntu-22-04/

```
egrep -c '(vmx|svm)' /proc/cpuinfo
apt install -y cpu-checker
kvm-ok
apt-get install -y qemu-kvm libvirt-daemon-system virtinst libvirt-clients bridge-utils
#
apt-get install -y libguestfs-tools  # virt-customize
apt-get install -y cloud-image-utils # cloud-localds
systemctl enable --now libvirtd
systemctl start        libvirtd
systemctl status       libvirtd
ADMIN_USER=madmin 
usermod -aG kvm     $ADMIN_USER
usermod -aG libvirt $ADMIN_USER

```

### glusterfs

https://www.howtoforge.com/how-to-install-and-configure-glusterfs-on-ubuntu-22-04/

```
vi /etc/hosts # "10.10.10.1 node1 10.10.10.2 node2"
vi /etc/netplan/00-installer-config.yaml
netplan apply

apt-get install glusterfs-server -y
systemctl start glusterd
systemctl enable glusterd
systemctl status glusterd
lsblk
fdisk /dev/sdb
mkfs.xfs /dev/sdb1
mkdir /srv/.bricks
vi /etc/fstab # "/dev/sdb1 /srv/.bricks xfs defaults 0 0"
mount -a
df -h
gluster pool list
#
#
#
gluster peer probe node2
gluster pool list
mkdir /tsp0
gluster volume create cust replica 2 node1://srv/.bricks/cust node2://srv/.bricks/cust
# Replica 2 volumes are prone to split-brain. Use Arbiter or Replica 3 to avoid this.
# See: http://docs.gluster.org/en/latest/Administrator-Guide/Split-brain-and-ways-to-deal-with-it/.

gluster volume start
gluster volume status
gluster volume info cust
vi /etc/fstab
# localhost:cust /tsp0 glusterfs defaults,_netdev 0 0
# #localhost:cust /tsp0  glusterfs log-file=/var/log/mirror.vol,defaults,_netdev,noauto,x-systemd.automount 0 0
mount -a
```

### bridges and bonds

https://fabianlee.org/2019/04/01/kvm-creating-a-bridged-network-with-netplan-on-ubuntu-bionic/

```
vi /etc/netplan/00-installer-config.yaml
netplan generate ; netplan apply

# 

vi /tmp/host-bridge.xml
<network>
  <name>guest-net</name>
  <forward mode="bridge"/>
  <bridge name="guest-net"/>
</network>

# create libvirt network using existing host bridge
virsh net-define x
virsh net-start guest-net
virsh net-autostart guest-net

# state should be active, autostart, and persistent
virsh net-list --all
```

### virtnbdbackup

https://github.com/abbbi/virtnbdbackup

```
cd /tmp
wget https://github.com/abbbi/virtnbdbackup/releases/download/v1.9.51/virtnbdbackup_1.9.51-1_all.deb
dpkg --force-depends -i /tmp/virtnbdbackup_1.9.51-1_all.deb
apt --fix-broken install
echo -e '/var/tmp/virtnbdbackup.* rw,\n/var/tmp/backup.* rw,'   > /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
echo -e '/var/tmp/virtnbdbackup.* rw,\n/var/tmp/backup.* rw,'   > /etc/apparmor.d/local/abstractions/libvirt-qemu
echo -e '/var/tmp/virtnbdbackup.* rw,\n/var/tmp/backup.* rw,'   > /etc/apparmor.d/local/usr.sbin.libvirtd

#
mkdir /backup
DOMAIN=test
virtnbdbackup  -d $DOMAIN -l auto -o /backup
```

### install netdata

https://wiki.crowncloud.net/?how_to_Install_netdata_monitoring_tool_ubuntu_22_04

```
apt install netdata -y
vi /etc/netdata/netdata.conf # bind socket ip
# firefox <ip>:19999

```

### utils and kc environment

```
apt-get install -y git tree screen swaks
git clone https://github.com/unimock/kc.git /opt/kc
/opt/kc/bin/kc-install init     # initialize kc environment
. /etc/profile                  # or re-login
```

# TODO



### create and deploy ssh keys
```
 mkdir /root/.ssh
 #ssh-keygen -b4096
 #cat id_rsa.pub >> authorized_keys
 echo "StrictHostKeyChecking=no"     >  /root/.ssh/config
 echo "UserKnownHostsFile=/dev/null" >> /root/.ssh/config
 scp /root/.ssh/* root@kvm2:/root/.ssh
 sed -i "s|#PasswordAuthentication yes|PasswordAuthentication no|" /etc/ssh/sshd_config
 /etc/init.d/ssh reload

```
### bonds and bridges 
```
 # https://www.cyberciti.biz/faq/ubuntu-linux-bridging-and-bonding-setup/
 # (/etc/network/interfaces)
 brctl show
 cat /proc/net/bonding/bond0
 kc-status bond
```
### install glusterfs
```
 # https://support.rackspace.com/how-to/set-up-a-two-server-glusterfs-array/
 # https://github.com/gluster/glusterfs-specs/blob/master/done/GlusterFS%203.5/Virt%20store%20usecase.md
 # http://www.admin-magazine.com/Articles/Build-storage-pools-with-GlusterFS/(offset)/3

 # resolve qemu port conflict:
 vi /etc/glusterfs/glusterd.vol
 option base-port 50152
 echo "10.10.10.1      gfs1" >> /etc/hosts
 echo "10.10.10.2      gfs2" >> /etc/hosts

 lsblk
 vgcreate data  /dev/sdb
 #lvcreate -n tsp0 -L 1T data
 lvcreate -n tsp0 -l100%FREE data
 umount /dev/data/tsp0
 mkfs.xfs /dev/data/tsp0
 mkdir -p /srv/.bricks
 echo "/dev/data/tsp0 /srv/.bricks xfs defaults 0 0" >> /etc/fstab
 mount /srv/.bricks
 gluster peer probe gfs1
 gluster peer probe gfs2

 GLUSTER_VOL="cust"
 gluster volume create ${GLUSTER_VOL} replica 2 gfs1://srv/.bricks/${GLUSTER_VOL} gfs2://srv/.bricks/${GLUSTER_VOL}
 gluster volume start  ${GLUSTER_VOL}
 gluster volume info

 echo "localhost:${GLUSTER_VOL} /tsp0  glusterfs log-file=/var/log/mirror.vol,defaults,_netdev,noauto,x-systemd.automount 0 0" >> /etc/fstab
 mount -a
```


## Administration

### glusterd

```
 kc-status gluster

 glusterd -V
 gluster volume status
 gluster volume info
 gluster peer probe gfs2
 gluster peer probe gfs1
 gluster peer status

 tail -f /var/log/glusterfs/glusterd.log
 
 systemctl status glusterd
 systemctl stop   glusterd
 systemctl start  glusterd
```

### dist-upgrade

```
 # @kvm1: (non productive)
 apt-get update ; apt-get autoremove ; apt-get dist-upgrade
 init 6
 kc-status bond
 kc-status gluster
 # change the productive node. migrate VMs from kvm2 to kvm1
 kc-tool mig kvm2 kvm1
```

## kc utilities

```
 kc-tool ls               # list VMs
 kc-tool mig kvm1 kvm2    # migrate VMs from kvm1 to kvm2 
 
 kc-status gluster         # display gluster status
 kc-status bond            # display bond status

 kc-syncxml complete       # sync VMs definition files between hosts

```

## virsh

```
 virsh list --all --name           # list all domains
 virsh start|shutdowm <domain>     # start/stop a domain
 virsh domstate <domain>
 virsh domtime  <domain>
```

## vm stuff
```
 apt-get install -y qemu-guest-agent
```


## Backup
### Initialize a backup device
```
  NAME="kvm-bup"
  read -s -p "Password: " passw && echo -n $passw > /tsp0/config/backup-passwd.conf  && passw=''
  cryptsetup luksAddKey $DEV  /tsp0/config/backup-passwd.conf
  cryptsetup -y -v luksFormat $DEV
  cryptsetup luksOpen $DEV $NAME
  cryptsetup -v status $NAME
  mkfs.xfs /dev/mapper/$NAME
  mount /dev/mapper/$NAME /backup
  umount /backup
  cryptsetup luksClose $NAME
```

### Initialize a additional backup device
```
  blkid          # get the device name for DEV
  DEV=/dev/sdc
  INFO_HD_NAME="atk-bup-3"
  NAME="kvm-bup"

  dd if=/dev/urandom bs=1M count=8 of=$DEV
  cryptsetup luksFormat -y $DEV #pw siehe /tsp0/config/backup-passwd.conf
  cryptsetup luksOpen $DEV $NAME < /tsp0/config/backup-passwd.conf

  blkid          # get the UID of the device and append it to /tsp0/config/backup-uuids.conf
  vi /tsp0/config/backup-uuids.conf

  mkfs.xfs /dev/mapper/$NAME
  mkdir -p /backup
  mount /dev/mapper/$NAME /backup
  echo "INFO_HD_NAME=\"$INFO_HD_NAME\"" > /backup/INFO.cfg
  cat /backup/INFO.cfg
  umount /backup
  cryptsetup luksClose $NAME
  rmdir /backup
  # test
  cryptsetup luksOpen $DEV $NAME < /tsp0/config/backup-passwd.conf
  mkdir -p /backup
  mount /dev/mapper/$NAME /backup
  df -h /backup
  cat  /backup/INFO.cfg
  umount /backup
  rmdir /backup
  cryptsetup luksClose $NAME

```

## gluster administration
### remove brick of an existing replicated volume (cust)

[https://support.rackspace.com/how-to/add-and-remove-glusterfs-servers/](URL)

```
# SSH into the glusterfs machine you wish to keep and do:
# @kvm1 (gfs1):
gluster peer status
gluster volume info
gluster volume remove-brick cust replica 1 gfs2:/srv/.bricks/cust force
gluster volume info cust
gluster peer detach gfs2
```

### re-create brick lvm-partition on **kvm2**
```
# @kvm2 (gfs2):
service glusterd stop
umount /srv/.bricks
mkfs.xfs /dev/data/tsp0
mount /srv/.bricks
service glusterd start
```

### add brick to an existing replicated volume (cust)
```
# @kvm1 (gfs1):
gluster peer probe gfs2
gluster volume status
gluster volume add-brick cust replica 2 gfs2:/srv/.bricks/cust
gluster vol status
```

### testing
```
#kc-status gluster
gluster volume status cust
gluster volume info   cust
gluster volume heal   cust info
```

## Hints
* Migration zwischen verschiedenen Host-CPUs: guest-cpu wie Ziel cpu einstellen
* nach Aktivieren eines Snapshot muss vor einem Migrate erst ein Restart der vm durchgeführt werden.

### repair brocken backup

```
 ssh kvm1
 cd /tsp0/images
 ls *.backup

 DOM="vw"
 virsh domblklist  ${DOM} --details
 virsh blockcommit ${DOM} vdb  --active --wait --pivot --verbose
 virsh domblklist  ${DOM} --details
 rm /tsp0/images/xxxx.backup

```
