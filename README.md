


# kc  ... kvm controller 
 
## Description

kc environment contains a couple of bash scripts to install and manage kvm  
from a central host and directory (master).

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 22.04 (amd64|arm64)
 * bare metal with VT support

## Installation

### arm:orangepi 5 plus

* https://jobcespedes.dev/2023/11/running-virtual-machines-on-orange-pi-5/
* https://github.com/Joshua-Riek/ubuntu-rockchip/wiki/Orange-Pi-5-Plus
* https://github.com/unimock/kc

**known issues with RK3588:**
  1. libvirt: migration -> assign cpu in xml :
     ````
       <vcpu placement="static" cpuset="0-3">2</vcpu>
       <vcpu placement="static" cpuset="4-7">2</vcpu>
     ````
  3. libvirt: copy out large files of a virtual machine (use e1000 instead virtio network adapter) -> solved in ubuntu-24.04
  4. virt-customize -a /srv/var/tmp/ubuntu-22.04-server.qcow2 --install qemu-guest-agent
  5. AppArmor not enabled (aa-status; journalctl --all | grep 'AppArmor')

#### prepare SD card
```
FI=/cust/images/ubuntu-24.04-preinstalled-server-arm64-orangepi-5-plus.img.xz
ls -la $FI
wget -O $FI https://github.com/Joshua-Riek/ubuntu-rockchip/releases/download/v2.1.0/ubuntu-24.04-preinstalled-server-arm64-orangepi-5-plus.img.xz
lsblk
DEV=/dev/sda
xz -dc $FI | sudo dd of=$DEV  bs=4k
```
#### backup/restore from SD
```
ssh arm3
cd /
tar -cvf  /xxx/armX.tar \
 etc/hosts \
 etc/hostname \
 etc/netplan/50-cloud-init.yaml \
 etc/netdata/netdata.conf \
 root/.bash_aliases \
 root/.ssh/ \
 etc/fstab \
 etc/systemd/timesyncd.conf \
 etc/cloud/cloud.cfg \
 etc/rsyslog.d/10-remote.conf

# local:
scp arm3:/xxx/armX.tar /xxx/
lsblk
DEV=/dev/sda2
sudo mount /dev/sda2 /mnt
cd /mnt
sudo tar xvf /xxx/armX.tar
sudo mkdir -p tsp0 srv/.bricks srv/var
cd / ; sudo umount /mnt
# apt-get dist-upgrade
# virtnbdbackup update (see above)
# (cd /opt/kc ; git pull)
```

#### boot system

Login: ubuntu
Password: ubuntu

#### installation/configurations

```
lsblk
# format NVMe Storage
fdisk /dev/nvme0n1 # p1: 50G; p2: rest
mkfs.xfs -f /dev/nvme0n1p1
mkfs.xfs -f /dev/nvme0n1p2
mkdir -p  /srv/var /srv/.bricks
echo "/dev/nvme0n1p1 /srv/var     xfs defaults 0 0" >> /etc/fstab
echo "/dev/nvme0n1p2 /srv/.bricks xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
# network configuration
vi /etc/netplan/50-cloud-init.yaml
chmod 600 /etc/netplan/50-cloud-init.yaml
netplan apply
# set timeserver
vi /etc/systemd/timesyncd.conf
 [Time]
 NTP=185.125.190.58
systemctl restart systemd-timesyncd.service
# set hostname 
echo "arm1" > /etc/hostname
sed -i "s|preserve_hostname: false|preserve_hostname: true|g" /etc/cloud/cloud.cfg
# set timezone
timedatectl set-timezone Europe/Berlin
# ssh keys
vi .ssh/authorized_keys
vi .ssh/config
vi .ssh/id_ed25519
vi .ssh/id_ed25519.pub
# define gluster hosts
vi /etc/hosts # arm1,arm2,..,amd1,amd2,...
```

### ubuntu-22.04 server
```
apt-get purge -y snapd
systemctl daemon-reload
apt-get update
apt-get dist-upgrade -y
apt-get autoremove -y
apt-get install -y net-tools tree s-tui libnet-ssleay-perl swaks
```
### kc

```
git clone https://github.com/unimock/kc.git /opt/kc
ln -s /opt/kc/bin/kvmc /usr/local/bin/kvmc
kvmc install
```

### fix ip, hostname, bridges

https://www.linuxtechi.com/static-ip-address-on-ubuntu-server/

```
timedatectl set-timezone Europe/Berlin
vi /etc/hosts
vi /etc/hostname
chmod 600  /etc/netplan/00-installer-config.yaml
vi /etc/netplan/00-installer-config.yaml
netplan generate
netplan apply
brctl show
# apply your ssh-keys
vi /root/.ssh/authorized_keys
```

### useful aliases

```
echo "alias ipa='ip -br -c a'" >> /root/.bash_aliases
echo "alias ipl='ip -br -c l'" >> /root/.bash_aliases
echo "alias ipr='ip -br -c r'" >> /root/.bash_aliases
. .bash_aliases
```
### glusterfs-server

https://www.howtoforge.com/how-to-install-and-configure-glusterfs-on-ubuntu-22-04/

```
vi /etc/hosts # "10.10.10.1 arm1 10.10.10.2 arm2 10.10.10.3 arm3"
vi /etc/netplan/00-installer-config.yaml
netplan apply

apt-get install -y glusterfs-server
systemctl enable --now glusterd
systemctl status glusterd
lsblk
DEV=/dev/sdX
fdisk $DEV
mkfs.xfs ${DEV}1
mkdir /srv/.bricks
echo "${DEV}1 /srv/.bricks xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
df -h
gluster pool list
gluster peer probe arm2
gluster pool list
mkdir /tsp0
#gluster volume create gv0 arm1://srv/.bricks/gv0 force
gluster volume create gv0 replica 2 arm1://srv/.bricks/gv0 arm2://srv/.bricks/gv0
#
# Replica 2 volumes are prone to split-brain. Use Arbiter or Replica 3 to avoid this.
# See: http://docs.gluster.org/en/latest/Administrator-Guide/Split-brain-and-ways-to-deal-with-it/.
#
gluster volume heal   gv0 info
gluster volume start  gv0
gluster volume status
gluster volume info   gv0

mkdir -p /tsp0
mount -t glusterfs localhost:gv0 /tsp0
```

### kvm/libvirt

https://www.linuxtechi.com/how-to-install-kvm-on-ubuntu-22-04/

```
apt-get install -y cpu-checker
kvm-ok
apt-get install -y qemu-kvm libvirt-daemon-system virtinst libvirt-clients bridge-utils libvirt-daemon-driver-storage-gluster
if [ "arm64" ] ; then
  # https://jobcespedes.dev/2023/11/running-virtual-machines-on-orange-pi-5/
  cd /usr/share/qemu/firmware/ ; ln -s 60-edk2-aarch64.json 00-edk2-aarch64.json
  apt-get install -y arm-trusted-firmware qemu-system-arm qemu-efi-aarch64 qemu-efi-arm  seabios ipxe-qemu
fi
apt-get install -y libguestfs-tools    # virt-customize
apt-get install -y cloud-image-utils   # cloud-localds
apt-get install -y virt-top            # top utility for VMs
systemctl restart libvirtd
systemctl enable --now libvirtd
systemctl status       libvirtd
#ADMIN_USER=madmin 
#usermod -aG kvm     $ADMIN_USER
#usermod -aG libvirt $ADMIN_USER
```

### dirty hack for glusterd-/libvirtd-/mount-issues on boot up

```
# with following mount option in /etc/fstab, only unsafe migration is possible: 
localhost:gv0 /tsp0 glusterfs defaults,_netdev,x-systemd.requires=glusterd.service,x-systemd.automount 0 0

# so dirty hack:
echo "localhost:gv0 /tsp0 glusterfs defaults,noauto 0 0"  >> /etc/fstab

vi /usr/lib/systemd/system/libvirtd.service
  After=glusterd.service
  ExecStartPre=mount /tsp0

systemctl daemon-reload
```

### glusterfs-client (only for systems which works as gluster client)

```
systemctl disable --now glusterd
# Einträge in /usr/lib/systemd/system/libvirtd.service können bleiben (see dirty hack)
systemctl stop libvirtd
umount /tsp0
vi /etc/fstab
  # replace: "localhost:gv0 /tsp0 glusterfs defaults,noauto 0 0"
  # with   : "arm1:gv0 /tsp0 glusterfs defaults,noauto,backupvolfile-server=arm2 0 0"
systemctl start libvirtd
ls /tsp0
``` 

### create script for networks and pools creation (can also be used in further installations)

```
mkdir -p /tsp0/scripts
cat <<'XXX' >/tsp0/scripts/kc-virt-init
#!/bin/bash
XML=/tmp/x

TYPE="net" ; NAME="lan-net"
virsh $TYPE-info $NAME >/dev/null 2>&1
if [ "$?" != "0"  ] ; then
cat <<EOF > $XML
<network>
  <name>$NAME</name>
  <forward mode="bridge"/>
  <bridge name="$NAME"/>
</network>
EOF
  virsh ${TYPE}-define $XML ; virsh ${TYPE}-start $NAME ; virsh ${TYPE}-autostart $NAME
fi

TYPE="net" ; NAME="mgmt-net"
virsh $TYPE-info $NAME >/dev/null 2>&1
if [ "$?" != "0"  ] ; then
cat <<EOF > $XML
<network>
  <name>$NAME</name>
  <forward mode="bridge"/>
  <bridge name="$NAME"/>
</network>
EOF
  virsh ${TYPE}-define $XML ; virsh ${TYPE}-start $NAME ; virsh ${TYPE}-autostart $NAME
fi

TYPE="pool" ; NAME="tsp0-images" ; TDIR="/tsp0/images"
virsh $TYPE-info $NAME >/dev/null 2>&1
if [ "$?" != "0"  ] ; then
cat <<EOF > $XML
<pool type="dir">
  <name>$NAME</name>
  <target>
    <path>$TDIR</path>
  </target>
</pool>
EOF
  mkdir -p $TDIR
  virsh ${TYPE}-define $XML ; virsh ${TYPE}-start $NAME ; virsh ${TYPE}-autostart $NAME
fi

TYPE="pool" ; NAME="tsp0-ISOs" ; TDIR="/tsp0/ISOs"
virsh $TYPE-info $NAME >/dev/null 2>&1
if [ "$?" != "0"  ] ; then
cat <<EOF > $XML
<pool type="dir">
  <name>$NAME</name>
  <target>
    <path>$TDIR</path>
  </target>
</pool>
EOF
  mkdir -p $TDIR
  virsh ${TYPE}-define $XML ; virsh ${TYPE}-start $NAME ; virsh ${TYPE}-autostart $NAME
fi

virsh net-list  --all
virsh pool-list --all

XXX

chmod a+x /tsp0/scripts/kc-virt-init
/tsp0/scripts/kc-virt-init
```

### virtnbdbackup

https://github.com/abbbi/virtnbdbackup

```
# apt list --installed | grep virtnbdbackup
# apt-get purge virtnbdbackup
#VERSION="1.9.51-1"
VERSION="2.9-1"
wget -O /tmp/vnb.deb https://github.com/abbbi/virtnbdbackup/releases/download/v${VERSION%-*}/virtnbdbackup_${VERSION}_all.deb
dpkg --force-depends -i /tmp/vnb.deb
apt --fix-broken install -y
# append lines:
/var/tmp/virtnbdbackup.* rw,
/var/tmp/backup.* rw,
in:
vi /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
vi /etc/apparmor.d/local/abstractions/libvirt-qemu
vi /etc/apparmor.d/local/usr.sbin.libvirtd
systemctl restart apparmor
systemctl status  apparmor
#
rm -rvf /test-backup
mkdir -p /test-backup
DOMAIN=test
virtnbdbackup  -d $DOMAIN -l auto -o /test-backup
rm -rvf /test-backup
```

### install netdata

https://wiki.crowncloud.net/?how_to_Install_netdata_monitoring_tool_ubuntu_22_04

```
apt-get install -y netdata
vi /etc/netdata/netdata.conf # bind socket ip
# firefox <ip>:19999
systemctl enable --now netdata
systemctl status  netdata

```

### create ssh keys for kvm's communication

```
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C  gluster
cat .ssh/id_ed25519.pub >> .ssh/authorized_keys
echo "StrictHostKeyChecking=no"     >  /root/.ssh/config
echo "UserKnownHostsFile=/dev/null" >> /root/.ssh/config
# scp /root/.ssh/* root@arm2:/root/.ssh
sed -i "s|#PasswordAuthentication yes|PasswordAuthentication no|" /etc/ssh/sshd_config
systemctl restart sshd
```

# Administration

## manage qemu images / domains

### dist-upgrade

```
# @arm2: (cold-standhy)
kvmc status
kvmc ls
apt-get update ; apt-get autoremove ; apt-get dist-upgrade
init 6
# change the productive node. migrate VMs from arm1 to arm2
kvmc --node=arm1 mig arm2
```

### kvmc util examples

```
kvmc ls                   # list VMs
kvmc --node=arm1 mig arm2 # migrate VMs from arm1 to arm2 
kvmc status               # show gluster status
kvmc syncxml              # sync VMs definition files between hosts
```

### virsh

```
DOM= 
virsh list --all --name       # list all domains
virsh start|shutdowm $DOM     # start/stop a domain
virsh domstate $DOM
virsh domtime  $DOM
virsh define $DOM /tsp0/sync/xml/kvm3/wws.xml
virsh domrename <old> <new>
```

### install qemu-guest-agent

```
apt-get install -y qemu-guest-agent
# todo: disable cloud-init
```

### resize partition of a cloud-init based VM

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

### create a raw disk that is not taken into account in kc-backup

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

## kc-backup

### initialize kc-backup

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

### management

```
# backup all defined domains
kc-backup backup 
# backup specified VM-domain
DOM="xxx"
kc-backup backup $DOM
kc-backup info   $DOM
kc-backup reset  $DOM
kc-backup reset --force-restart $DOM  # stop VM and remove bitmaps within image
```

### Initialize first backup device

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
### Repair backup device

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
## gluster administration


### general
```
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

### resolve split-brains

```
# Here, <HOSTNAME:BRICKNAME> is selected as source brick and <FILE> present in the source brick is taken as the source for healing.
# gluster volume heal <VOLNAME> split-brain source-brick <HOSTNAME:BRICKNAME>   <FILE>
  gluster volume heal   gv0     split-brain source-brick arm2:/srv/.bricks/gv0 /images/ubuntu-test.qcow2
```

### testing/info
```
kvmc ls
kvmc status
gluster pool list
gluster volume status gv0
gluster volume info   gv0
gluster volume heal   gv0 info
gluster volume heal   gv0 info summary
```

### remove brick of an existing replicated volume (gv0)

[https://support.rackspace.com/how-to/add-and-remove-glusterfs-servers/](URL)

```
PEER=arm2
REPLICA=1

gluster peer status
gluster volume heal gv0 info
gluster volume remove-brick gv0 replica ${REPLICA} ${PEER}:/srv/.bricks/gv0 force
gluster volume info gv0
gluster peer detach ${PEER}
gluster peer status
```

### re-create brick **@arm2**
```
ssh arm2
systemctl stop libvirtd
umount /tsp0
systemctl stop glusterd
umount /srv/.bricks
lsblk
mkfs.xfs -f /dev/sdb1
mount /srv/.bricks
systemctl start glusterd
#
# add-brick on the active host; then:
#
mount /tsp0
gluster volume status gv0
gluster volume info   gv0
gluster volume heal   gv0 info
systemctl start libvirtd
```

### add brick to an existing replicated volume (gv0)

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

