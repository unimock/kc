# Installation

## installation (install-complete)

```install-complete
# execute install-xxx sections below
md-exec /opt/kc/INSTALL.md run install-base
md-exec /opt/kc/INSTALL.md run install-glusterd
md-exec /opt/kc/INSTALL.md run install-libvirtd
md-exec /opt/kc/INSTALL.md run install-virtnbdbackup
md-exec /opt/kc/INSTALL.md run install-kvmc-config
md-exec /opt/kc/INSTALL.md run install-dist-upgrade
#init 6
#md-exec /opt/kc/INSTALL.md run test-vm-backup
```

### installation (install-base)

```install-base
apt-get purge -y snapd
rm -Rf /root/snap
apt-get update
apt-get install -y net-tools tree s-tui libnet-ssleay-perl swaks
#
# install netdata (https://wiki.crowncloud.net/?how_to_Install_netdata_monitoring_tool_ubuntu_22_04)
#
apt-get install -y netdata
MYIP=$(host `hostname` | cut -d " " -f 4)
sed -i "s|IP = 127.0.0.1|IP = $MYIP|g" /etc/netdata/netdata.conf
systemctl enable --now netdata
# test firefox <ip>:19999

#
# define gluster node in /etc/hosts
#
ENTRY="$(host `hostname` | cut -d " " -f 4) `hostname`"
sed -i "1s/^/$ENTRY \n\n/" /etc/hosts
#
# create ssh keys for kvm's communication
#
ssh-keygen -q -N "" -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C gluster
cat .ssh/id_ed25519.pub >> .ssh/authorized_keys
echo "StrictHostKeyChecking=no"     >  /root/.ssh/config
echo "UserKnownHostsFile=/dev/null" >> /root/.ssh/config

#
# useful aliases
#
cat <<'EOF' >>/root/.bash_aliases
alias ipa='ip -br -c a'
alias ipl='ip -br -c l'
alias ipr='ip -br -c r'
EOF
. .bash_aliases

# check apparmor
aa-status ; systemctl status apparmor

# check timesyncd
timedatectl status
```

### glusterfs-server (install-glusterd)

https://www.howtoforge.com/how-to-install-and-configure-glusterfs-on-ubuntu-22-04/

```install-glusterd
IS_GLUSTER_PEER=1
apt-get install -y glusterfs-server
mkdir -p /tsp0
if [ "${IS_GLUSTER_PEER}" = "1" ] ; then
  # format NVMe Storage
  lsblk
  DEV="/dev/nvme0n1"
  wipefs --all --force $DEV
  sgdisk -o $DEV
  sgdisk -n 1:0:+50G $DEV
  sgdisk -n 2:0:0 $DEV
  mkfs.xfs -f ${DEV}p1
  mkfs.xfs -f ${DEV}p2
  mkdir -p  /srv/var /srv/.bricks
  echo "${DEV}p1 /srv/var     xfs defaults 0 0" >> /etc/fstab
  echo "${DEV}p2 /srv/.bricks xfs defaults 0 0" >> /etc/fstab
  mount -a
  df -h | grep "${DEV}p1"
  df -h | grep "${DEV}p2"
  
  systemctl enable --now glusterd
  systemctl status glusterd
  gluster pool list
  gluster volume create gv0 $(hostname)://srv/.bricks/gv0 force
  gluster volume start  gv0
  gluster volume status
  gluster volume info   gv0
  mount -t glusterfs localhost:gv0 /tsp0
else
  systemctl enable --now glusterd
  echo "arm1:gv0 /tsp0 glusterfs defaults,noauto,backupvolfile-server=arm2 0 0" >> /etc/fstab
  systemctl daemon-reload
fi
mount /tsp0
df -h | grep "/tsp0"
umount /tsp0
```

### kvm/libvirt (install-libvirtd)

https://www.linuxtechi.com/how-to-install-kvm-on-ubuntu-22-04/

```install-libvirtd
IS_GLUSTER_PEER=1
apt-get install -y cpu-checker
kvm-ok
apt-get install -y qemu-kvm libvirt-daemon-system virtinst libvirt-clients bridge-utils libvirt-daemon-driver-storage-gluster
if [ "$(dpkg --print-architecture)" = "arm64" ] ; then
  # https://jobcespedes.dev/2023/11/running-virtual-machines-on-orange-pi-5/
  apt-get install -y arm-trusted-firmware qemu-system-arm qemu-efi-aarch64 qemu-efi-arm  seabios ipxe-qemu
  ( cd /usr/share/qemu/firmware/ ; ln -s 60-edk2-aarch64.json 00-edk2-aarch64.json )
fi
apt-get install -y libguestfs-tools    # virt-customize
apt-get install -y cloud-image-utils   # cloud-localds
apt-get install -y virt-top            # top utility for VMs
  
systemctl restart libvirtd
systemctl enable --now libvirtd
systemctl status       libvirtd

echo "#system is gluster client:"                                              >> /etc/fstab
echo "#arm1:gv0 /tsp0 glusterfs defaults,noauto,backupvolfile-server=arm2 0 0" >> /etc/fstab

#ADMIN_USER=madmin ; usermod -aG kvm $ADMIN_USER ; usermod -aG libvirt $ADMIN_USER
systemctl stop libvirtd
#
# dirty startup hack for glusterd-/libvirtd-/mount-issues on boot up
#
if [ "${IS_GLUSTER_PEER}" = "1" ] ; then
  echo "localhost:gv0 /tsp0 glusterfs defaults,noauto 0 0  # do not mount /tsp0 automatically"  >> /etc/fstab
  mount /tsp0
  # mount /tsp0/ before libvirtd will be started :
  mkdir -p /tsp0/scripts
  FI=/tsp0/scripts/dirty-startup-hack
  md-exec /opt/kc/INSTALL.md dump dirty-startup-hack > $FI
  chmod a+x $FI
  /tsp0/scripts/dirty-startup-hack
  umount /tsp0
  systemctl start libvirtd
  df -h | grep tsp0
  #
  # create script for networks and pools creation (can also be used in further installations)
  #
  mkdir -p /tsp0/scripts
  md-exec /opt/kc/INSTALL.md dump kc-virt-init >/tsp0/scripts/kc-virt-init
  chmod a+x /tsp0/scripts/kc-virt-init
  /tsp0/scripts/kc-virt-init
  md-exec /opt/kc/INSTALL.md dump kc-virt-destroy >/tsp0/scripts/kc-virt-destroy
  chmod a+x /tsp0/scripts/kc-virt-destroy
else
  mount /tsp0
  /tsp0/scripts/kc-virt-init
  /tsp0/scripts/dirty-startup-hack
fi
```


### virtnbdbackup (install-virtnbdbackup)

https://github.com/abbbi/virtnbdbackup

```install-virtnbdbackup
# update:
# apt list --installed | grep virtnbdbackup
# apt-get purge -y virtnbdbackup
VERSION="2.29-1"
wget -O /tmp/vnb.deb https://github.com/abbbi/virtnbdbackup/releases/download/v${VERSION%-*}/virtnbdbackup_${VERSION}_all.deb
dpkg --force-depends -i /tmp/vnb.deb
apt --fix-broken install -y

cat <<EOF > /etc/apparmor.d/local/abstractions/libvirt-qemu
/var/tmp/virtnbdbackup.* rw,
/var/tmp/backup.* rw,
EOF
#
# fix unmask apparmor?????
#
#virsh capabilities | grep -C 3 secmodel
systemctl status apparmor
systemctl unmask apparmor
systemctl start  apparmor
systemctl status apparmor
```

### create config for kvmc utils (install-kvmc-config)

```install-kvmc-config
mkdir -p /tsp0/config
echo "KC_GLUSTER_ACTIVE=true"   > /tsp0/config/gluster.conf
echo "KC_GLUSTER_VOLUME=gv0"   >> /tsp0/config/gluster.conf
echo "$(hostname)"              > /tsp0/config/hosts.conf

kvmc ls
kvmc status
```

### dist-upgrade (install-dist-upgrade)
```install-dist-upgrade
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get dist-upgrade -y
apt-get autoremove -y
```

### tests (test-vm-backup)

```test-vm-backup
systemctl status apparmor
df -h | grep /tsp0
kvmc ls
#
# create a test VM
#
mkdir -p /xxx
FI=/xxx/test.yml
if [ "$(dpkg --print-architecture)" = "arm64" -a "$(lsb_release -s -r 2>/dev/null)" = "24.04" ] ; then
  md-exec /opt/kc/INSTALL.md dump test-arm-u24-yml >$FI
else
  md-exec /opt/kc/INSTALL.md dump test-domain-yml  >$FI
fi
PUB_KEY=$( cat /root/.ssh/id_ed25519.pub )
sed -i "s|<PUB_KEY>|${PUB_KEY}|g" $FI
/opt/kc/bin/kc-dc provide test    $FI
/opt/kc/bin/kc-dc ip      test
kvmc ls
#
# test virtnbdbackup:
#
mkdir -p /srv/var/test-backup
virtnbdbackup  -d test -l auto -o /srv/var/test-backup
/opt/kc/bin/kc-dc delete test
#
virtnbdrestore -D -N test -i /srv/var/test-backup -o /tsp0/images
kvmc ls
kvmc --domain=test up
rm -rvf /srv/var/test-backup
```

## scripts and config files

**kc-virt-init** script

```kc-virt-init
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
exit
```

**kc-virt-destroy** script

```kc-virt-destroy
#!/bin/bash
#
# reset pools and networks
#
#virsh pool-list --all
list="tsp0-images tsp0-ISOs"
for i in $list ; do
  virsh pool-destroy $i
  virsh pool-undefine $i
done
#virsh net-list --all
list="lan-net mgmt-net"
for i in $list ; do
  virsh net-destroy $i
  virsh net-undefine $i
done
virsh pool-list --all
virsh net-list --all
exit
```

**dirty-startup-hack** script 

```dirty-startup-hack
#!/bin/bash
# mount /tsp0/ before libvirtd will be started :
FI=/usr/lib/systemd/system/libvirtd.service
SUB="ExecStartPre=mount /tsp0"
grep -q "${SUB}" $FI || sed -i "/\[Service\]/a ${SUB}" $FI
# unmount /tsp0 after libvirtd is stopped :
SUB="ExecStopPost=umount /tsp0"
grep -q "${SUB}" $FI || sed -i "/\[Service\]/a ${SUB}" $FI
# start libvirtd after glusterd :
SUB="After=glusterd.service"
grep -q "${SUB}" $FI || sed -i "/\[Unit\]/a ${SUB}" $FI
systemctl daemon-reload
exit
```

**test domain configuration**

```test-domain-yml
type: kvm
kvm:
  cloud_image: https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-arm64.img
  image_name: ubuntu-22.04-server-arm64
  vol-pool:
    name: tsp0-images
    target: /tsp0/images
  size: 5G
  virt-install:
    - --memory 2048
    - --vcpus=2
    - --os-variant ubuntu22.04
    - --network bridge=lan-net,model=virtio

cloud-config:
  packages:
    - qemu-guest-agent
  runcmd:
    - - systemctl
      - enable
      - '--now'
      - qemu-guest-agent.service
  chpasswd:
    expire: false
    list: |
      root:mypassword
  hostname: test
  users:
    - name: root
      ssh_pwauth: true
      lock-passwd: false
      ssh_authorized_keys:
        - '<PUB_KEY>'
  timezone: Europe/Berlin
```

```test-arm-u24-yml
type: kvm
kvm:
  cloud_image: https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-arm64.img
  image_name: ubuntu-22.04-server-arm64
  vol-pool:
    name: tsp0-images
    target: /tsp0/images
  size: 5G
  virt-install:
    - --memory 2048
    - --vcpus=2,cpuset=0-3
    - --os-variant ubuntu22.04
    - --network bridge=lan-net,model=virtio
    - --boot=loader=/usr/share/AAVMF/AAVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/AAVMF/AAVMF_VARS.fd,loader_secure=no
cloud-config:
  packages:
    - qemu-guest-agent
  runcmd:
    - - systemctl
      - enable
      - '--now'
      - qemu-guest-agent.service
  chpasswd:
    expire: false
    list: |
      root:mypassword
  hostname: test
  users:
    - name: root
      ssh_pwauth: true
      lock-passwd: false
      ssh_authorized_keys:
        - '<PUB_KEY>'
  timezone: Europe/Berlin
```
