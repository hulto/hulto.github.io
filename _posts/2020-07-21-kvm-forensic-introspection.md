---
layout: post
title: Hypervisor introspection on KVM.
categories: [general, guide]
tags: [infrastructure, kvm, security, forensices, defense, guide]
description: Guide to setup kvm with support for introspection.
---

## Set up KVM
1. Install kvm
{% highlight bash linenos %}
sudo apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils libguestfs-tools genisoimage virtinst libosinfo-bin
{% endhighlight %}

2. Add user permissions
{% highlight bash linenos %}
sudo usermod -aG libvirt sysadmin
sudo adduser sysadmin libvirt-qemu
id
{% endhighlight %}

3. Configure briged networking
  1. Use real existing network
{% highlight bash linenos %}
sudo vi /etc/network/interfaces.d/br0

auto br0
iface br0 inet static
    address 172.22.0.19
    broadcast 172.22.0.255
    netmask 255.255.255.0
    gateway 172.22.0.1
    bridge_ports enp5s0f1
    bridge_stp off
    bridge_waitport 0
    bridge_fd 0

sudo systemctl restart network-manager

sudo virsh net-list --all
sudo vi /root/briged.xml

<network>
<name>br0</name>
<forward mode="bridge"/>
<bridge name="br0"/>
</network>

sudo virsh net-define --file /root/briged.xml
sudo virsh net-autostart br0
sudo virsh net-start br0
{% endhighlight %}

5. Create a VM
{% highlight bash linenos %}
cd /var/lib/libvirt/boot/
sudo wget https://mirrors.kernel.org/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-1708.iso

sudo virt-install \
--virt-type=kvm \
--name centos7 \
--ram 2048 \
--vcpus=2 \
--os-variant=rhel7 \
--virt-type=kvm \
--hvm \
--cdrom=/var/lib/libvirt/boot/CentOS-7-x86_64-Minimal-1810.iso \
--network=bridge=br0,model=virtio \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--disk path=/home/sysadmin/VirtualMachines/centos7.qcow2,size=40,bus=virtio,format=qcow2

sudo virsh dumpxml centos7 | grep vnc

sudo virsh vncdisplay centos7

ssh user@hostname -L 5900:127.0.0.1:5900
{% endhighlight %}

## Configure introspection (patched kernel method)
1. Install LibVMI
{% highlight bash linenos %}
sudo apt-get install cmake flex bison libglib2.0-dev libvirt-dev libjson-c-dev libyajl-dev git

git clone https://github.com/KVM-VMI/libvmi.git
cd libvmi
mkdir build
cd build
cmake ..
make
sudo make install

#Make sure that QMP is enabled
sudo virsh qemu-monitor-command centos7 --pretty '{"execute":"query-kvm"}'

#Patch linux kernel with better VMI support
git clone https://github.com/KVM-VMI/kvm-vmi.git --recursive --branch kvmi

sudo apt-get install bc fakeroot flex bison libelf-dev libssl-dev ncurses-dev

cd kvm-vmi/kvm
make olddefconfig

vi .config

CONFIG_KVM=y
CONFIG_KVM_INTEL=y
CONFIG_KVM_AMD=y
CONFIG_KSM=n
CONFIG_REMOTE_MAPPING=y
CONFIG_KVM_INTROSPECTION=y
CONFIG_SYSTEM_TRUSTED_KEYS=””

make -j4 bzImage
make -j4 modules
sudo make modules_install
sudo make install
sudo reboot now

uname -a 
# Should return 5.0.0


#Configure VM offsets
tar -czvf linux_offset_tool.tar libvmi/tools/linux-offset-finder
scp linux_offset_tool.tar user@vm-ip:/tmp/

user@vm-ip~$ cd /tmp/
user@vm-ip~$ tar -xzvf linux_offset_tool.tar
user@vm-ip~$ cd linux-offset-finder
# Choose link from https://linuxsoft.cern.ch/cern/centos/7/updates/x86_64/repoview/kernel-devel.html
user@vm-ip~$ sudo rpm -ivh --force https://linuxsoft.cern.ch/cern/centos/7/updates/x86_64/Packages/kernel-devel-3.10.0-957.el7.x86_64.rpm
user@vm-ip~$ sudo yum install make gcc
user@vm-ip~$ make
user@vm-ip~$ sudo insmod findoffsets.ko
user@vm-ip~$ sudo dmesg
user@vm-ip~$ sudo cp /boot/System* /tmp/
sysadmin@kvm-server:~: sudo scp user@vm-ip:/tmp/System-* /boot/

sysadmin@kvm-server:~: sudo vi /etc/libvmi.conf 
centos7
{
	sysmap = "/boot/System.map-3.10.0-957.el7.x86_64";
	ostype="Linux";
	linux_name = 0x678;
	linux_tasks = 0x430;
	linux_mm = 0x468;
	linux_pid = 0x4a4;
	linux_pgd = 0x58;
}

sysadmin@kvm-server:~/libvmi/build/examples$ sudo ./vmi-process-list centos7
{% endhighlight %}

