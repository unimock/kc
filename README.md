# kc  ... kvm controller 

## Description

kc environment contains a couple of bash scripts to install and manage kvm  
from a central host and directory (master).

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 16.04
 * bare metal with VT support

## Installation

### utilities, tools and  packages
```
 git clone https://github.com/unimock/kc.git /opt/kc
 /opt/kc/bin/kc-install init     # initialize kc environment
 /opt/kc/bin/kc-install packages # install additional packages
 . /etc/profile                  # or re-login
```
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
 dc-status
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
 mkfs.ext4 /dev/data/tsp0
 mkdir -p /srv/.bricks
 echo "/dev/data/tsp0 /srv/.bricks ext4 defaults 0 1" >> /etc/fstab
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

### install/update netdata (ubuntu-16.04 LTS)

see: https://guides.wp-bullet.com/install-netdata-monitoring-tool-ubuntu-16-04-lts/

```
 #
 # install:
 #
 apt-get update
 apt-get install -y zlib1g-dev uuid-dev libmnl-dev gcc make git autoconf autogen automake pkg-config curl jq nodejs
 git clone https://github.com/firehol/netdata.git --depth=1 /root/netdata
 cd /root/netdata
 ./netdata-installer.sh
 killall netdata
 systemctl enable netdata
 systemctl start  netdata
 #
 # update:
 #
 systemctl stop   netdata
 cd /root/netdata
 git pull
 ./netdata-installer.sh
 systemctl start  netdata
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
 kc-virsh mig kvm2 kvm1
```

## kc utilities

```
 kc-virsh ls               # list VMs
 kc-virsh mig kvm1 kvm2    # migrate VMs from kvm1 to kvm2 
 
 kc-status gluster         # display gluster status
 kc-status bond            # display bond status

 kc-syncxml complete       # sync VMs definition files between hosts

 kc-getiso alpine          # download images
 kc-getiso ubuntu
```

## vish

```
 virsh list --all --name           # list all domains
 virsh start|shutdowm <domain>     # start/stop a domain
 virsh domstate <domain>
 virsh domtime  <domain>
```

## create a disk from shell

```
 qemu-img create -f qcow2 /tsp0/images/bse1_temp_disk.qcow2 100G
 qemu-img info /tsp0/images/bse1_temp_disk.qcow2 100G
 chown libvirt-qemu:kvm /tsp0/images/bse1_temp_disk.qcow2
 # assign disk to the vm with virtual machine manager
 # assign and mount disk from the the vm
 hsh bse1
 fdisk -l
 fdisk /dev/vdc
 mkfs.ext4 /dev/vdc1
 mkdir /temp_disk ; chmod a+rwx /temp_disk
 echo "/dev/vdc1 /temp_disk          ext4    errors=remount-ro 0       2" >> /etc/fstab
 mount -a
 df -h
 init 6 
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
  mkfs.ext4 /dev/mapper/$NAME
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

  mkfs.ext4 /dev/mapper/$NAME
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
lsblk
lvdisplay
lvremove /dev/data/tsp0
vgremove data
vgdisplay
pvremove  /dev/sdb
pvdisplay
vgcreate data  /dev/sdb
lvcreate -n tsp0 -l100%FREE data
mkfs.ext4 /dev/data/tsp0
mount /srv/.bricks
service glusterd start
```

### add brick to an existing replicated volume (cust)
```
# @kvm1 (gfs1):
gluster peer probe gfs2
gluster volume status
#??? @kvm2: gluster volume stop   cust
#??? @kvm2: gluster volume delete cust
gluster volume add-brick cust replica 2 gfs2:/srv/.bricks/cust
gluster vol status

# @kvm2 (gfs2)
mount -a
find /tsp0
gluster volume heal cust info
```

### testing
```
#kc-status gluster
gluster volume status cust
gluster volume info   cust
gluster volume heal   cust info
```


## Hints
* nach Migrate ist Autostart der VM nicht mehr aktiv
* Migration zwischen verschiedenen Host-CPUs: guest-cpu wie Ziel cpu einstellen
* nach Aktivieren eines Snapshot muss vor einem Migrate erst ein Restart der vm durchgeführt werden.

