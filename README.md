
## Instructions for GPU passthrough in KVM on RHEL/CentOS 7

Tested OS version: `CentOS Linux release 7.3.1611 (Core)`

Enable BIOS settings: Intel VT-d

Enable BIOS support for large-BAR1 GPUs: 'MMIO above 4G' or 'Above 4G encoding', etc.

Update the system: `sudo yum update`

Verify virtualization support is enabled in the BIOS, you should see `vmx` for Intel or `svm` for AMD processors:

```
grep -oE 'svm|vmx' /proc/cpuinfo | uniq
```

Find GPU devices to assign to the pci-stub driver:

```
[centos@sas03 ~]$ lspci -n | grep 10de
04:00.0 0302: 10de:102d (rev a1)
05:00.0 0302: 10de:102d (rev a1)
86:00.0 0302: 10de:102d (rev a1)
87:00.0 0302: 10de:102d (rev a1)
```

Edit `/etc/default/grub` and add default kernel parameters to enable IOMMU groups, blacklist the nouveau GPU driver and assign GPUs to the pci-stub driver:

> GRUB_CMDLINE_LINUX=" ... intel_iommu=on iommu=pt pci=realloc pci=nocrs rdblacklist=nouveau nouveau.modeset=0 pci-stub.ids=10de:102d"

Re-install GRUB: `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`

Blacklist the nouveau driver:

```
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee -a /etc/modprobe.d/blacklist.conf
```

Enable the `vfio-pci` driver on boot:

```
echo vfio-pci | sudo tee /etc/modules-load.d/vfio-pci.conf
```

Reboot

Install KVM packages:

```
sudo yum install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install
```

Start libvirtd:

```
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

Create VM:

> Use GPU device PCI address from previous `lspci` output

> Hit `Ctrl-]` to exit virt-install console

```
sudo virt-install \
--name centos6 \
--ram 24576 \
--disk path=./centos6.qcow2,size=8 \
--vcpus 8 \
--os-type linux \
--os-variant=rhel6 \
--network bridge=virbr0 \
--graphics none \
--console pty,target_type=serial \
--location 'http://mirror.centos.org/centos/6/os/x86_64/' \
--extra-args 'console=ttyS0,115200n8 serial' \
--hostdev 04:00.0
```

To list running VMs (add `-a` for all VMs): `virsh list`

To connect to a running VM: `virsh --connect qemu:///session start centos6`

To stop VM: `virsh shutdown centos6`

To remove VM: `virsh undefine centos6`

---

References:

  * https://linux.dell.com/files/whitepapers/KVM_Virtualization_in_RHEL_7_Made_Easy.pdf
