---
title: "Configuration of IVSHMEM between VMs"
layout: post
date: 2019-09-13 17:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- markdown
- components
- extra
category: blog
author: Qiang Su
description: Shared memory
---

## Summary:

IVSHMEM is a kind of shared memory between VMs.

### Install KVM  
```
wget https://download.qemu.org/qemu-2.5.0.tar.bz2
tar -jxvf qemu-2.5.0.tar.bz2
sudo apt-get install build-essential gcc pkg-config glib-2.0 libglib2.0-dev libsdl1.2-dev
sudo apt-get install qemu-kvm libvirt-bin
cd qemu-2.5.0
./configure --enable-kvm
make
sudo make install
sudo qemu-img --help | grep version # see version  
kvm --version
sudo apt-get install qemu-kvm libvirt-bin ubuntu-vm-builder bridge-utils
sudo apt-get install virtinst virt-manager virt-viewer
sudo apt-get install qemu-system
kvm-ok #verify the installation  
dpkg -s qemu-kvm | grep Version # see version  
sudo lsmod | grep kvm #see the module information  
```

### Configure a IVSHMEM channel between two VMs  
#### Create two VMs with libvirt and kvm  

```
qemu-img create -f qcow2 node1.qcow2 5G
sudo virt-install --name node1 --vcpus=1 --os-type=Linux --virt-type kvm \
    --ram=1024 --location=/home/suqiang/images/ubuntu-16.04.6-server-amd64.iso \
    --disk path=/home/suqiang/vms/ubuntu1/node1.qcow2 \
    --graphics none --extra-args console=ttyS0
```
[**Note**]: Choose to install openssh server when being asked if install new programme.
```
sudo virsh list
sudo virsh dumpxml node1 | grep mac
arp -a # get the IP of node1  
```
Find the IP corresponding to the mac, here node1’s ip is `192.168.122.24`  
```
ssh suqiang@192.168.122.24
sudo systemctl enable serial-getty@ttyS0.service
sudo systemctl start serial-getty@ttyS0.service  
```

Clone another vm - node2  
```
sudo virsh dumpxml node1 > /home/suqiang/vms/ubuntu2/node2.xml
cp /home/suqiang/ubuntu1/node1.qcow2 /home/suqiang/ubuntu2/node2.qcow2
vim /home/suqiang/vms/ubuntu2/node2.xml
```
Modify uuid, disk source and console port; delete mac (will be generated automatically)  
```
sudo virsh define /home/suqiang/ubuntu2/node2.xml
sudo virsh start node2 # ssh and enable console
sudo virsh console node2
sudo vim /etc/hostname # change the hostname of node2  
sudo reboot
```
Now two vms are created  

#### Configure IVSHMEM between VMs and host  
```
echo 4096 >  /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 4096 >  /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
mkdir /mnt/huge
sudo mount -t hugetlbfs nodev /mnt/huge
sudo ivshmem-server -v -F -m /mnt/huge/ -S /tmp/ivs1 -l 512M -n 2
sudo virsh dumpxml node1 > /home/suqiang/vms/ubuntu1/node1.xml 
nano node1.xml
```
Add xmlns at domain  `<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>`  
```  
qemu-system-x86_64 -machine help  #Select a supported machine - here pc-i440fx-2.5 is chose  
```
Modify machine to pc-i440fx-2.5  
Delete the line of model in cpu, or it will not find the model  
Add qemu command line after `</devices>`  
```html
`<qemu:commandline>`  
    `<qemu:arg value='-chardev'/>`  
    `<qemu:arg value='socket,path=/tmp/ivs1,id=ivshmem_socket'/>`  
    `<qemu:arg value='-device'/>`  
    `<qemu:arg value='ivshmem,chardev=ivshmem_socket,vectors=2,size=512M'/>`  
`</qemu:commandline>`  
```
Save `node1.xml `   
```
sudo virsh undefine node1
sudo virsh define node1.xml
sudo vim /etc/libvirt/qemu.conf #Ensure that both user and group are “root”  
cd /etc/apparmor.d/libvirt/
sudo virsh dumpxml node1 | grep uuid
sudo aa-complain libvirt-<uuid>
sudo virsh start node1
sudo virsh console node1
```
In node1, run `lspci`, if it shows something like `00:05.0 RAM memory: Red Hat, Inc. Inter-VM shared memory`, the IVSHMEM is configured successfully.  
Do the same thing for node2.  

#### Install IVSHMEM Driver in VMs
```
git clone https://github.com/cmacdonell/ivshmem-code.git
cd ivshmem-code/kernel_module/uio
make
sudo make install
sudo modprobe uio
sudo insmod uio_ivshmem
```






