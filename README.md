# Disclaimer
It is a 'proof of concept'! Not even a alpha-version!
This piece of code is another try to beat 'boot-storm'.
It uses torrent to deliver image to node.

|Number of nodes|Time for cold boot, min|
|:-------------:|:---------------------:|
|              1|                      3|
|             36|                      4|
|             72|                      4|

Image size is 1GB, provision node is equipped with 1Gb interface

# Start
Let's assume you have server with ip 10.30.255.254 and this repo in /luna

# Server preparation

TODO. Will be raplaced by rpm scripts later on.
```
useradd -d /opt/luna luna
chown luna: /opt/luna
mkdir /run/luna
chown luna: /run/luna/
mkdir /var/log/luna
chown luna: /var/log/luna
mkdir /opt/luna/{boot,torrents}
chown luna: /opt/luna/{boot,torrents}
yum -y install https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
yum -y install mongodb-server python-pymongo mongodb
yum -y install nginx
yum -y install python-tornado
yum -y install ipxe-bootimgs tftp-server tftp xinetd
yum -y install rb_libtorrent-python net-snmp-python
mkdir /tftpboot
sed -e 's/^\(\W\+disable\W\+\=\W\)yes/\1no/g' -i /etc/xinetd.d/tftp
sed -e 's|^\(\W\+server_args\W\+\=\W-s\W\)/var/lib/tftpboot|\1/tftpboot|g' -i /etc/xinetd.d/tftp
systemctl enable xinetd
systemctl start xinetd
cp /usr/share/ipxe/undionly.kpxe /tftpboot/luna_undionly.kpxe
mv /etc/dhcp/dhcpd.conf{,.bkp_luna}
cp /luna/config/dhcpd/dhcpd.conf /etc/dhcp/
vim /etc/dhcp/dhcpd.conf
cp /luna/config/nginx/luna.conf /etc/nginx/conf.d/
systemctl start nginx
systemctl enable nginx
systemctl start mongod
systemctl enable mongod
```
# Creating links
```
cd /usr/lib64/python2.7
ln -s ../../../luna/src/module luna
cd /usr/sbin
ln -s ../../luna/src/exec/luna
ln -s ../../luna/src/exec/lpower
ln -s ../../luna/src/exec/lweb
ln -s ../../luna/src/exec/ltorrent
cd /opt/luna
ln -s /luna/src/templates
```
# Creating osimage
```
mkdir -p /opt/luna/os/compute/var/lib/rpm
rpm --root /opt/luna/os/compute --initdb
yumdownloader  centos-release
rpm --root /opt/luna/os/compute -ivh centos-release\*.rpm
yum --installroot=/opt/luna/os/compute -y groupinstall Base
yum --installroot=/opt/luna/os/compute -y install kernel rootfiles openssh-server openssh openssh-clients tar nc wget curl rsync gawk sed gzip parted e2fsprogs ipmitool vi
mkdir /opt/luna/os/compute/root/.ssh
chmod 700 /opt/luna/os/compute/root/.ssh
mount -t devtmpfs devtmpfs /opt/luna/os/compute/dev/
chroot  /opt/luna/os/compute
ssh-keygen -f /etc/ssh/ssh_host_ecdsa_key -N '' -t ecdsa
exit
cat /root/.ssh/id_rsa.pub >> /opt/luna/os/compute/root/.ssh/authorized_keys
chmod 600 /opt/luna/os/compute/root/.ssh/authorized_keys
cp -pr /luna/src/dracut/95luna /opt/luna/os/compute/usr/lib/dracut/modules.d/
```
# Create cluster config
```
luna cluster init
luna cluster change --frontend_address 10.30.255.254
luna osimage add -n compute -p /opt/luna/os/compute
luna osimage pack -n compute
luna bmcsetup add -n base
luna network add -n cluster -N 10.30.0.0 -P 16
luna network add -n ipmi -N 10.31.0.0 -P 16
luna switch add -n switch01 --oid .1.3.6.1.2.1.17.7.1.2.2.1.2 --ip 10.31.253.21
luna group add -n compute -i enp7s0 -o compute
luna group change -n compute -b base
luna group change -n compute --boot_if enp7s0
luna group change -n compute --interface enp7s0 --setnet cluster
luna group change -n compute --bmcnetwork --setnet ipmi
```
# Edit partitioning

You can use ramdisk, or write your own partition script.
```
luna group change -n compute --partscript -e
```
It could be, for example
```
parted /dev/sda -s 'rm 1; rm 2'
parted /dev/sda -s 'mkpart p ext2 1 256m'
parted /dev/sda -s 'mkpart p ext3 256m 100%'
parted /dev/sda -s 'set 1 boot on'
mkfs.ext2 /dev/sda1
mkfs.ext4 /dev/sda2
mount /dev/sda2 /sysroot
mkdir /sysroot/boot
mount /dev/sda1 /sysroot/boot
```
# Add node.
```
luna node add -g compute
```
Name will be generated.
```
luna node change -n node001 -s switch01
luna node change -n node001 -p 1
```
# Start services
```
ltorrent start
lweb
```
The last one is not the service. Will daemonize it later. Please run it in screen.
# Check if it is working
```
curl "http://10.30.255.254:7050/luna?step=boot"
wget "http://10.30.255.254:7050/boot/compute-vmlinuz-3.10.0-327.10.1.el7.x86_64"
curl "http://10.30.255.254:7050/luna?step=install&node=node001"
```
# Boot node