# kc  ... kvm controller 

## Description

kc environment contains a couple of bash scripts to install and manage kvm  
from a central host and directory (master).

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 16.04
 * bare metal with VT support

## Installation

```
git clone https://github.com/unimock/kc.git /opt/kc
/opt/kc/bin/kc-install init     # initialize kc environment
/opt/kc/bin/kc-install packages # install additional packages
. /etc/profile                  # or re-login
```
## Utilities

```
 kc-visrh ls               # list VMs
 kc-virsh mig kvm1 kvm2    # migrate VMs from kvm1 to kvm2 
 
 kc-status gluster         # display gluster status
 kc-status bond            # display bond status

 kc-syncxml complete       # sync VMs definition files between hosts

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
  lsblk
  fdisk /dev/sdc  # create partition
  cryptsetup -y -v luksFormat /tsp0/config/backup-passwd.conf
  blkid
  NAME="kvm-bup"
  cryptsetup luksOpen $DEV $NAME < /tsp0/config/backup-passwd.conf
  mount /dev/mapper/$NAME /backup
  echo "INFO_HD_NAME=name_of_hd" > /backup/INFO.cfg
```


