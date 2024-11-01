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

#If your VM has a graphical interface and you prefer to use a graphical console, you can use virt-viewer
sudo apt install virt-viewer 
virt-viewer <vm-name>

#If you prefer a graphical user interface (GUI) for managing your VMs, you can use virt-manager
sudo apt install virt-manager
virt-manager
```