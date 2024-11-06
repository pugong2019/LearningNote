## Intsall VM(KVM)
1. Install KVM and related packages
```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
lsmod | grep kvm #Verify KVM installation
sudo usermod -aG libvirt $(whoami) #Add your user to the libvirt and kvm groups, this allows you to manage VMs without root privileges.
sudo usermod -aG kvm $(whoami)
```
2. Download the Linux ISO
3. Install
```shell
qemu-img create -f qcow2 ~/Workspace/VM/ubuntu-22.04.qcow2 500G #Create a virtual disk
virt-install --name sos_build_vm --ram 32768 --vcpus 12 --disk path=~/Workspace/VM/ubuntu-22.04.qcow2,size=500 --os-variant ubuntu22.04 --network bridge=virbr0 --graphics none --console pty,target_type=serial --location /home/gp/Workspace/VM/image/ubuntu-22.04.4-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd --extra-args 'console=ttyS0,115200n8 serial' #Use virt-install to create and install the VM. Replace the paths and parameters as needed.
```
4. Most frenquently used cmds
```shell
virsh start ubuntu-vm
virsh console ubuntu-vm # crtl + ] Press the two keys simultaneously to quit the console
virsh shutdown ubuntu-vm
virsh list --all
virsh reboot <vm-name>
virsh destroy <vm-name> #Destroy (force stop) a VM
virsh suspend <vm-name> #Suspend a VM
virsh resume <vm-name> #Resume a suspended VM
virsh dominfo <vm-name> #Get VM information
virsh domifaddr ubuntu-22 #Find the VM's IP Address
sudo arp-scan --interface=virbr0 --localnet  #Find the VM's IP Address

#If your VM has a graphical interface and you prefer to use a graphical console, you can use virt-viewer
sudo apt install virt-viewer 
virt-viewer <vm-name>

#If you prefer a graphical user interface (GUI) for managing your VMs, you can use virt-manager
sudo apt install virt-manager
virt-manager
virsh domblklist Vname #get vm disk info
qemu-img resize  /home/Free/VM1.qcow2 +10G #虚拟机关闭状态下扩容, 直接扩容，新增容量未分区，要进行合并等操作
virsh blockresize  VM1 /home/Free/VM1.qcow2 40G #虚拟机运行状态下扩容
```

## Increase VM Disk Capacity
1. enter vm
```
gp@test-vm:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0 63.9M  1 loop /snap/core20/2105
loop1                       7:1    0   87M  1 loop /snap/lxd/27037
loop2                       7:2    0 63.7M  1 loop /snap/core20/2434
loop3                       7:3    0   87M  1 loop /snap/lxd/29351
loop4                       7:4    0 40.4M  1 loop /snap/snapd/20671
sr0                        11:0    1 1024M  0 rom
vda                       252:0    0  1.3T  0 disk
├─vda1                    252:1    0    1M  0 part
├─vda2                    252:2    0    2G  0 part /boot
└─vda3                    252:3    0  498G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0  100G  0 lvm  /
```
2. Get qcow info:
```shell
$ qemu-img info ubuntu-22.04.qcow2
image: ubuntu-22.04.qcow2
file format: qcow2
virtual size: 1.27 TiB (1395864371200 bytes)
disk size: 87.1 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: ubuntu-22.04.qcow2
    protocol type: file
    file length: 87.1 GiB (93555916800 bytes)
    disk size: 87.1 GiB
## virtual size: 1.27 TiB (1395864371200 bytes) 是设置的vm的disk的大小，但是不一定是实际占用的磁盘的大小，当前机器为1T，但是virtual size > 1T
```

3. Get partion Format
```shell
$ df -T
Filesystem                        Type  1K-blocks     Used Available Use% Mounted on
tmpfs                             tmpfs   3286160     1132   3285028   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv ext4  102626232 97391688         0 100% /
tmpfs                             tmpfs  16430800        0  16430800   0% /dev/shm
tmpfs                             tmpfs      5120        0      5120   0% /run/lock
/dev/vda2                         ext4    1992552   127312   1744000   7% /boot
tmpfs                             tmpfs   3286160        4   3286156   1% /run/user/1000
```

4. Steps to increase Capacity
enter vm by ssh or virsh console

```shell
$lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0 63.9M  1 loop /snap/core20/2105
loop1                       7:1    0 63.7M  1 loop /snap/core20/2434
loop2                       7:2    0   87M  1 loop /snap/lxd/27037
loop3                       7:3    0   87M  1 loop /snap/lxd/29351
loop4                       7:4    0 40.4M  1 loop /snap/snapd/20671
sr0                        11:0    1 1024M  0 rom  
vda                       252:0    0  1.3T  0 disk 
├─vda1                    252:1    0    1M  0 part 
├─vda2                    252:2    0    2G  0 part /boot
└─vda3                    252:3    0  498G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:0    0  490G  0 lvm  /

$sudo fdisk -l
Disk /dev/vda: 1.27 TiB, 1395864371200 bytes, 2726297600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 01326846-BF7E-4A72-9DD3-B52D1DE68115

Device       Start        End    Sectors  Size Type
/dev/vda1     2048       4095       2048    1M BIOS boot
/dev/vda2     4096    4198399    4194304    2G Linux filesystem
/dev/vda3  4198400 1048573951 1044375552  498G Linux filesystem


Disk /dev/mapper/ubuntu--vg-ubuntu--lv: 490 GiB, 526133493760 bytes, 1027604480 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```
```shell
sudo pvresize /dev/vda3
sudo lvextend -L 480G /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h
```