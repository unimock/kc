# kc  ... kvm controller 
 
## Description

kc environment contains a couple of bash scripts to install and manage a cluster of kvm nodes.

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 22.04 (amd64|arm64)
 * bare metal with VT support

## Installation

### arm: orangepi 5 plus (OPi5+)

* https://jobcespedes.dev/2023/11/running-virtual-machines-on-orange-pi-5/
* https://github.com/Joshua-Riek/ubuntu-rockchip/wiki/Orange-Pi-5-Plus
* https://github.com/unimock/kc

**known issues with RK3588:**
  1. libvirt: migration -> assign cpu in xml :
     ````
       <vcpu placement="static" cpuset="0-3">2</vcpu>
       <vcpu placement="static" cpuset="4-7">2</vcpu>
     ````
  2. libvirt: copy out large files of a virtual machine (use e1000 instead virtio network adapter) -> solved in ubuntu-24.04
  3. virt-customize -a /srv/var/tmp/ubuntu-22.04-server.qcow2 --install qemu-guest-agent

#### prepare SD card for OPi5+ with cloud-init support
```
IMAGE_VER="2.2.1"
UBUNTU_VER="22.04"
FI=/cust/images/ubuntu-${UBUNTU_VER}-preinstalled-server-arm64-orangepi-5-plus.img.xz
ls -la $FI
wget -O $FI https://github.com/Joshua-Riek/ubuntu-rockchip/releases/download/v${IMAGE_VER}/ubuntu-${UBUNTU_VER}-preinstalled-server-arm64-orangepi-5-plus.img.xz
lsblk
DEV=/dev/sda
xz -dc $FI | sudo dd of=$DEV  bs=4k
mount ${DEV}/1 /mnt
vi /mnt/user-data       # hostname, timezone, user, password, ssh-keys,...
# network configuration (https://www.linuxtechi.com/static-ip-address-on-ubuntu-server/) :
vi /mnt/network-config  # interfaces, ip addresses, ..
umount /mnt
```
#### create USB stick w/o cloud-init
```
dd ..
# boot USB stick and follow installer, ...,
# after re-boot:
vi /etc/netplan/50-cloud-init.yaml
echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
passwd root
# create/place ssh keys, authorized_keys, config, ...
sed -i "s|#PasswordAuthentication yes|PasswordAuthentication no|g" /etc/ssh/sshd_config
systemctl restart ssh
# deluser --remove-home madmin
init 6
```

### kvmc cluster node installation 

preparation

```
#
# install kvmc
#
git clone https://github.com/unimock/kc.git /opt/kc
ln -s /opt/kc/bin/kvmc       /usr/local/bin/kvmc
ln -s /opt/kc/bin/kc-backup  /usr/local/bin/kc-backup
kvmc install

#
# install md-exec helper
#
wget -O /usr/local/bin/md-exec https://raw.githubusercontent.com/unimock/addapps/main/md-exec
chmod a+x /usr/local/bin/md-exec
```

## installation

[see ./INSTALL.md](INSTALL.md)


## administration

### add a new kvm node (arm3) to existing cluster from (arm1) (admin-add-node)

```admin-add-node
PEER=arm3
#
# destroy brick @ $PEER
#
ssh $PEER kvmc ls
ssh $PEER kvmc rm complete
ssh $PEER systemctl stop libvirt-guests
ssh $PEER /tsp0/scripts/kc-virt-destroy
ssh $PEER systemctl stop libvirtd
ssh $PEER ls /tsp0
ssh $PEER "echo y | gluster volume stop gv0"
ssh $PEER "echo y | gluster volume delete gv0"
ssh $PEER systemctl stop glusterd
ssh $PEER rm -Rvf /srv/.bricks/gv0
ssh $PEER systemctl start glusterd
#
# add $PEER gluster brick
#
REPLICA=3
gluster peer probe ${PEER}
gluster peer status
gluster volume status
gluster volume add-brick gv0 replica ${REPLICA} ${PEER}:/srv/.bricks/gv0
gluster volume status
gluster volume info   gv0
gluster volume heal   gv0 info
#
# update config for now kvm and copy to new $PEER kvm
#
vi /etc/hosts
scp /etc/hosts $PEER:/etc/
scp -r /root/.ssh       $PEER:/root
scp /root/.bash_aliases $PEER:/root
vi /tsp0/config/hosts.conf
ssh $PEER cat /tsp0/scripts/kc-virt-init # wait for gluster heal
ssh $PEER /tsp0/scripts/kc-virt-init
ssh $PEER kvmc ls
```

### manage qemu images / domains

#### dist-upgrade

```
# @arm2: (hot-standhy)
kvmc status
kvmc ls
apt-get update ; apt-get autoremove ; apt-get dist-upgrade
init 6
# change the production node. migrate VMs from arm1 to arm2
kvmc --node=arm1 mig arm2
```

#### kvmc util examples

```
kvmc ls                   # list VMs
kvmc --node=arm1 mig arm2 # migrate VMs from arm1 to arm2 
kvmc status               # show gluster status
kvmc syncxml              # sync VMs definition files between hosts
```

#### virsh

```
DOM= 
virsh list --all --name       # list all domains
virsh start|shutdowm $DOM     # start/stop a domain
virsh domstate $DOM
virsh domtime  $DOM
virsh define $DOM /tsp0/sync/xml/kvm3/wws.xml
virsh domrename <old> <new>
```

#### resize partition of a cloud-init based VM

```
DOM="ubuntu-test"
kvmc --domain=$DOM list images
IMG=/tsp0/images/ubuntu-test.qcow
qemu-img resize $IMG +5G
qemu-img info   $IMG
modprobe nbd max_part=8
qemu-nbd --connect=/dev/nbd0 $IMG
fdisk -l /dev/nbd0
growpart -v /dev/nbd0 1
resize2fs -f /dev/nbd0p1
fdisk -l /dev/nbd0
qemu-nbd --disconnect /dev/nbd0
rmmod nbd
kvmc --domain=$DOM up
```

#### create a raw disk that is not taken into account in kc-backup

```
ssh arm1
kvmc --domain=vw down
IMG=/tsp0/images/vw_data-snap.img
SIZE=400G
rm $IMG
qemu-img create $IMG $SIZE
modprobe nbd max_part=8
qemu-nbd --format=raw --connect=/dev/nbd0 $IMG
mkfs.xfs -f /dev/nbd0
qemu-nbd --disconnect /dev/nbd0
sleep 1
rmmod nbd
```
virt-manager for **vm**:
    * Gerät hinzufügen -> Speicher -> select image and assign as **SCSI**
    * Boot-Optionen -> Startreihenfolge ggf. anpassen
    * power on **vw**

```
ssh vw
mkdir -p /data-snap
echo "/sda /data-snap xfs defaults,nofail 0 0" >> /etc/fstab
mount -a
df -h
# rsync -avzS --numeric-ids --delete /woher-auch-immer/ /data-snap
```

### kc-backup

#### initialize kc-backup

```
cat << EOF >/tsp0/config/kc-backup.conf
KC_BACKUP_AREA="xxx"
KC_BACKUP_NODES"arm1 arm2 amd1"
KC_BACKUP_FROM="admin@xxx.de"
KC_BACKUP_TO="yyy@xxx.de"
KC_BACKUP_SMTP="smtp.xxx.intra:25"
# exclude domains seperated with blanks
KC_BACKUP_EXCLUDE=""
KC_BACKUP_SSH_OPT=""
EOF
```

#### management

```
# backup all defined domains
kc-backup backup 
# backup specified VM-domain
DOM="xxx"
kc-backup backup $DOM
kc-backup info   $DOM
kc-backup reset  $DOM
kc-backup reset --force-restart $DOM  # stop VM and remove bitmaps within image
kc-backup restore test
```

#### Initialize first backup device

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

#### Initialize a additional backup device
```
blkid          # get the device name for DEV
DEV=/dev/sdc
INFO_HD_NAME="atk-bup-XXX"
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
kc-backup mount
df -h
kc-backup umount
```
#### Repair backup device

```
lsblk
DEV=/dev/sdb
NAME=xxx
cryptsetup luksOpen $DEV $NAME
xfs_repair /dev/mapper/$NAME 
mount  /dev/mapper/$NAME /mnt
umount  /mnt
xfs_repair /dev/mapper/$NAME
cryptsetup luksClose $DEV $NAME

```
### gluster administration


#### general
```
#
# Replica 2 volumes are prone to split-brain. Use Arbiter or Replica 3 to avoid this.
# See: http://docs.gluster.org/en/latest/Administrator-Guide/Split-brain-and-ways-to-deal-with-it/.
#
kcvmc status
glusterd -V
gluster volume status
gluster volume info
gluster peer probe arm2
gluster peer probe arm1
gluster peer status
tail -f /var/log/glusterfs/glusterd.log
systemctl status glusterd
systemctl stop   glusterd
systemctl start  glusterd
```

#### resolve split-brains

```
# Here, <HOSTNAME:BRICKNAME> is selected as source brick and <FILE> present in the source brick is taken as the source for healing.
# gluster volume heal <VOLNAME> split-brain source-brick <HOSTNAME:BRICKNAME>   <FILE>
  gluster volume heal   gv0     split-brain source-brick arm2:/srv/.bricks/gv0 /images/ubuntu-test.qcow2
```

#### testing/info
```
kvmc ls
kvmc status
gluster pool list
gluster volume status gv0
gluster volume info   gv0
gluster volume heal   gv0 info
gluster volume heal   gv0 info summary
```

#### remove brick of an existing replicated volume (gv0)

[https://support.rackspace.com/how-to/add-and-remove-glusterfs-servers/](URL)

```
PEER=arm3
REPLICA=2

# stop libvirt and un-mount /tsp0
ssh $PEER systemctl stop gluster-guests
ssh $PEER systemctl stop libvirtd
ssh $PEER ls /tsp0

gluster peer status
gluster volume heal gv0 info
echo y | gluster volume remove-brick gv0 replica ${REPLICA} ${PEER}:/srv/.bricks/gv0 force
gluster volume info gv0
echo y | gluster peer detach ${PEER}
gluster peer status
kvmc status
```

#### re-create brick at PEER
```
PEER=arm3

ssh $PEER systemctl stop libvirt-guests
ssh $PEER systemctl stop libvirtd
ssh $PEER ls /tsp0
ssh $PEER rm -rf /srv/.bricks/gv
#
# add-brick on the active host
# if done, then:
#
ssh $PEER gluster volume status gv0
ssh $PEER gluster volume info   gv0
ssh $PEER gluster volume heal   gv0 info
ssh $PEER systemctl start libvirtd
ssh $PEER ls /tsp0
```

#### add brick to an existing replicated volume (gv0)

```
PEER="arm3"
REPLICA=3

gluster peer probe ${PEER}
gluster peer status
gluster volume status
gluster volume add-brick gv0 replica ${REPLICA} ${PEER}:/srv/.bricks/gv0
gluster volume status
gluster volume info   gv0
gluster volume heal   gv0 info
```

# known issues

```
 #
 # Live Migration Failure With Error "same uuid":
 #
 vi /etc/libvirt/libvirtd.conf  # host_uuid_source = "machine-id"

 #
 # Live Migration Failure With Error "Unable to find security driver for model apparmor"
 #
apt install apparmor-utils
aa-status
aa-complain /etc/apparmor.d/usr.sbin.libvirtd
aa-enforce /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
systemctl restart libvirtd
aa-status
mount /tsp0/
```
