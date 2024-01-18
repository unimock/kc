
# kc  ... kvm controller 
 
## Description

kc environment contains a couple of bash scripts to install and manage kvm  
from a central host and directory (master).

It only works as a wrapper for libvirt for convenient usage.

## Requirements

 * ubuntu-server 22.04
 * bare metal with VT support

## Installation

### ubuntu-22.04 server
```
apt-get purge -y snapd
systemctl daemon-reload
apt-get update
apt-get dist-upgrade -y
apt-get autoremove -y
apt-get install -y net-tools
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
#touch /etc/cloud/cloud-init
vi /root/.bash_aliases
alias ipa='ip -br -c a'
alias ipl='ip -br -c l'
alias ipr='ip -br -c r'
```
### glusterfs

https://www.howtoforge.com/how-to-install-and-configure-glusterfs-on-ubuntu-22-04/

```
vi /etc/hosts # "10.10.10.1 node1 10.10.10.2 node2 10.10.10.3 node3"
vi /etc/netplan/00-installer-config.yaml
netplan apply

apt-get install -y glusterfs-server
systemctl start glusterd
systemctl enable glusterd
systemctl status glusterd
lsblk
DEV=/dev/sdX
fdisk $DEV
mkfs.xfs ${DEV}1
mkdir /srv/.bricks
echo "${DEV}1 /srv/.bricks xfs defaults 0 0" >> /etc/fstab
mount -a
df -h
gluster pool list
gluster peer probe node2
gluster pool list
mkdir /tsp0
gluster volume create gv0 replica 2 node1://srv/.bricks/gv0 node2://srv/.bricks/gv0
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
egrep -c '(vmx|svm)' /proc/cpuinfo
apt-get install -y cpu-checker
kvm-ok
apt-get install -y qemu-kvm libvirt-daemon-system virtinst libvirt-clients bridge-utils libvirt-daemon-driver-storage-gluster
systemctl restart libvirtd
apt-get install -y libguestfs-tools    # virt-customize
apt-get install -y cloud-image-utils   # cloud-localds
systemctl enable --now libvirtd
systemctl start        libvirtd
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
vi /etc/fstab
  localhost:gv0 /tsp0 glusterfs defaults,noauto 0 0

vi /usr/lib/systemd/system/libvirtd.service
  After=glusterd.service
  ExecStartPre=mount /tsp0

systemctl daemon-reload

```


### bridges and bonds

https://fabianlee.org/2019/04/01/kvm-creating-a-bridged-network-with-netplan-on-ubuntu-bionic/

```
# netzwerk interfaces, die nicht verwendet werden auskommentieren!!! (wait for interface @ boot)
vi /etc/netplan/00-installer-config.yaml
netplan generate
netplan apply
brctl show
#
# create lan-net
#
cat <<EOF >/tmp/x
<network>
  <name>lan-net</name>
  <forward mode="bridge"/>
  <bridge name="lan-net"/>
</network>
EOF
virsh net-define /tmp/x
virsh net-start lan-net
virsh net-autostart lan-net
#
# create mgmt-net
#
cat <<EOF >/tmp/x
<network>
  <name>mgmt-net</name>
  <forward mode="bridge"/>
  <bridge name="mgmt-net"/>
</network>
EOF
virsh net-define /tmp/x
virsh net-start mgmt-net
virsh net-autostart mgmt-net
virsh net-list --all
```

### virtnbdbackup

https://github.com/abbbi/virtnbdbackup

```
wget -O /tmp/vnb.deb https://github.com/abbbi/virtnbdbackup/releases/download/v1.9.51/virtnbdbackup_1.9.51-1_all.deb
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
systemctl enable netdata
systemctl start  netdata

```

### create ssh keys for kvm's communication

```
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C  gluster
cat .ssh/id_ed25519.pub >> .ssh/authorized_keys
echo "StrictHostKeyChecking=no"     >  /root/.ssh/config
echo "UserKnownHostsFile=/dev/null" >> /root/.ssh/config
# scp /root/.ssh/* root@node2:/root/.ssh
sed -i "s|#PasswordAuthentication yes|PasswordAuthentication no|" /etc/ssh/sshd_config
systemctl restart sshd

```


# Administration

## manage qemu images / domains

### dist-upgrade

```
 # @node2: (cold-standhy)
 kvmc status
 kvmc ls
 apt-get update ; apt-get autoremove ; apt-get dist-upgrade
 init 6
 # change the productive node. migrate VMs from node1 to node2
 kvmc --node=node1 mig node2
```

### kvmc util examples

```
 kvmc ls                   # list VMs
 kvmc --node=node1 mig node2 # migrate VMs from node1 to node2 
 kvmc status               # show gluster status
 kvmc syncxml              # sync VMs definition files between hosts
```

### virsh

```
 virsh list --all --name           # list all domains
 virsh start|shutdowm <domain>     # start/stop a domain
 virsh domstate <domain>
 virsh domtime  <domain>
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


## Backup

### TBD: delete all virtnbdbackup data and remove virtnbdbackup bitmaps from images

```
DOM="vw"
kc-backup info ${DOM}
# shutdown domain
kvmc --domain=${DOM} down
# delete all virtnbdbackup data and remove virtnbdbackup bitmaps from images
rm -rvf /tsp0/sync/backup/${DOM}.conf /tsp0/sync/backup/${DOM}
images=$(kvmc --domain=${DOM} list images)
for i in $images ; do
  echo "doing image: $i"
  bitmaps=$(qemu-img info $i | grep " name: virtnbdbackup" | awk -F: '{ print $2}')
  for b in $bitmaps ; do
    echo "remove bitmap: $b"
    qemu-img bitmap  $i --remove $b
  done
  qemu-img info $i
done
# start domain
kvmc --domain=${DOM} up
# start full backup
kc-backup backup ${DOM}
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

## gluster administration


### general
```
 kcvmc status
 glusterd -V
 gluster volume status
 gluster volume info
 gluster peer probe node2
 gluster peer probe node1
 gluster peer status
 tail -f /var/log/glusterfs/glusterd.log
  systemctl status glusterd
 systemctl stop   glusterd
 systemctl start  glusterd
```

### resolve split-brain

```
# Here, <HOSTNAME:BRICKNAME> is selected as source brick and <FILE> present in the source brick is taken as the source for healing.
# gluster volume heal <VOLNAME> split-brain source-brick <HOSTNAME:BRICKNAME>   <FILE>
  gluster volume heal   gv0     split-brain source-brick node2:/srv/.bricks/gv0 /images/ubuntu-test.qcow2
```

### testing/info
```
kvmc ls
kvmc status
gluster pool list
gluster volume status gv0
gluster volume info   gv0
gluster volume heal   gv0 info
```

### remove brick of an existing replicated volume (gv0)

[https://support.rackspace.com/how-to/add-and-remove-glusterfs-servers/](URL)

```
# SSH into the glusterfs machine you wish to keep and do:
# @node1:
gluster peer status
gluster volume info
node=node2
gluster volume remove-brick gv0 replica 1 ${node}:/srv/.bricks/gv0 force
gluster volume info gv0
gluster peer detach ${node}
gluster peer status
```

### re-create brick **@node2**
```
# @node2:
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
# @node1:
node=node2
gluster peer probe ${node}
gluster peer status
gluster volume status
gluster volume add-brick gv0 replica 2 ${node}:/srv/.bricks/gv0
gluster volume status
gluster volume info   gv0
gluster volume heal   gv0 info
```

