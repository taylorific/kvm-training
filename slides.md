---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
hideInToc: true
title: KVM/QEMU Training
author: Mischa Taylor
info: |
  ## Slidev Starter Template
  Presentation slides for developers.
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
# seoMeta:
#  ogImage: https://cover.sli.dev
themeConfig:
  paginationX: r
  paginationY: t
  paginationPagesDisabled: [1]
---

# Virtual Machines with KVM/QEMU

##### Mischa Taylor | 📧 <taylor@linux.com>

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Press Space for next page <carbon:arrow-right />
</div>

<div class="abs-br m-6 text-xl">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="slidev-icon-btn">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
hideInToc: true
routeAlias: toc
---

# Table of Contents

<Toc columns="2"/>

---
hideInToc: true
---

# What is KVM?

- KVM - **K**ernel-based **V**irtual **M**achine
- Built-in to the Linux kernel
- Kernel allocates memory for each VM
  - Ensures that each guest has its own memory space
  - Ensures that memory for each virtual machine is isolated from the other guests and the host
- KVM cannot:
  - Create a VM
  - Save a VM
  - Open a VM
  - Provide user-accessible APIs
- Controlled using ioctl calls

---
hideInToc: true
---

# KVM is useless by itself

- QEMU
  - Provides user-space CLI binaries to create and load VMs
  - Emulates different hardware by replicating in software
  - Can use hardware to accelerate virtualization for near-native performance (a.k.a. paravirtualization or virtio)
- Libvirt
  - Daemons that control virtual machine lifecycle
  - Provides standard API for starting, stopping, configuring VMs

---
src: ./ubuntu-host/libvirt/libvirt-ubuntu-2604-slides.md
---

---
layout: section
---

# Spin up your first VM

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Verify host resources

```bash
$ virsh nodeinfo
CPU model:           x86_64
CPU(s):              12
CPU frequency:       800 MHz
CPU socket(s):       1
Core(s) per socket:  6
Thread(s) per core:  2
NUMA cell(s):        1
Memory size:         64312264 KiB
```

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           1.5G  2.8M  1.5G   1% /run
/dev/vda2        62G   12G   47G  20% /
tmpfs           3.6G     0  3.6G   0% /dev/shm
efivarfs        256K   68K  184K  27% /sys/firmware/efi/efivars
none            1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs           3.6G  8.0K  3.6G   1% /tmp
none            1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/vda1       1.1G  6.4M  1.1G   1% /boot/efi
tmpfs           727M   80K  727M   1% /run/user/1000
```

---
hideInToc: true
---

# Ubuntu cloud images

https://cloud-images.ubuntu.com/

- Fast to spin up and minimal
- Do not come pre-configured with a default login
- Minimal drivers/hardware support out of the box

---
hideInToc: true
---

# Download cloud image template and resize

```bash
$ mkdir ubuntu-server-2604
$ cd ubuntu-server-2604
```

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
```

```bash
$ curl -LO \
    https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img
$ qemu-img info resolute-server-cloudimg-amd64.img
```

```bash
$ sudo qemu-img convert \
    -f qcow2 -O qcow2 \
    resolute-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/ubuntu-server-2604.qcow2
$ sudo qemu-img resize -f qcow2 \
    /var/lib/libvirt/images/ubuntu-server-2604.qcow2 \
    32G
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-cloud-images-proxy/resolute/current/resolute-server-cloudimg-amd64.img
```
-->

---
hideInToc: true
---

# Define login parameters for cloud-init ISO

```bash
# Required for NoCloud module to function, uniquely identifies instance
cat >meta-data <<EOF
instance-id: ubuntu-server-2604
local-hostname: ubuntu-server-2604
EOF

# Main configuration script, tells cloud-init what to do when instance starts
cat >user-data <<EOF
#cloud-config
password: superseekret
chpasswd:
  expire: false
ssh_pwauth: true
EOF
```

---
hideInToc: true
---

# Spin up image and configure with cloud-init

```bash
virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2604 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu-lts-latest \
  --disk /var/lib/libvirt/images/ubuntu-server-2604.qcow2,bus=virtio \
  --cloud-init user-data=user-data,meta-data=meta-data,disable=on \
  --network network=default,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

# Virtual machine console

```bash
# Command line console
virsh console ubuntu-server-2604

# Graphical console
virt-viewer ubuntu-server-2604
```

---
hideInToc: true
---

# Install qemu-guest-agent

```
$ sudo apt-get update
$ sudo apt-get install qemu-guest-agent
```

---
hideInToc: true
---

# Verify cloud-init finished

In the guest:
```bash
# login with ubuntu user
$ cloud-init status
status: done

# volume labeled cidata is mounted as /dev/sr0 with files
# and copied to /var/lib/cloud/instance/user-data.txt
$ lsblk -f
NAME FSTYPE FSVER LABEL           UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sr0  iso966 Jolie cidata          2026-05-12-22-28-56-00
```

On the host:
```bash
$ virsh dumpxml ubuntu-server-2604 | grep -iA5 cdrom
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/boot/virtinst-l9b9vxtc-cloudinit.iso' index='1'/>
      <backingStore/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <alias name='sata0-0-0'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

```
$ sudo shutdown -h now
```

---
hideInToc: true
---

After reboot:
```bash
# restart and login with ubuntu user

# /usr/lib/cloud-init/ds-identify checks for cidata volume
# if found, sets the provider to nocloud, otherwise disabled
$ cloud-init status --long
status: disabled
extended_status: disabled
boot_status_code: disabled-by-generator
detail: Cloud-init disabled by cloud-init-generator
errors: []
recoverable_errors: {}
```

---
hideInToc: true
---


# Snapshots

```bash
$ virsh snapshot-create-as --domain ubuntu-server-2604 --name clean --description "Initial install"
$ virsh snapshot-list ubuntu-server-2604
$ virsh snapshot-revert ubuntu-server-2604 clean
$ virsh snapshot-delete ubuntu-server-2604 clean
```

# Cleanup

```bash
$ virsh shutdown ubuntu-server-2604
$ virsh undefine ubuntu-server-2604 --nvram --remove-all-storage
```

---
hideInToc: true
---

# Get IP of virtual machine

Preferred - use qemu-guest-agent

```bash
virsh start ubuntu-server-2604

virsh list --all
```

```
$ virsh domifaddr ubuntu-server-2604 --source agent
```

When using default NAT-based networking, can query dnsmasq

```
virsh net-dhcp-leases default
```

```bash
virsh domiflist ubuntu-server-2604
sudo apt-get install net-tools
arp -an
```

---
hideInToc: true
---

# If you need additional drivers

```bash
# First try to install the kernel modules package
sudo apt-get update
sudo apt-get install linux-modules-$(uname -r)

# sudo modprobe <kernel_module>

# If that still doesn't work, try installing the full generic kernel image
sudo apt-get install linux-image-generic
# linux-generic-hwe-26.04
sudo reboot
```

---
layout: section
---

# Legacy method using boot-scratch pool

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# cloud-init image pool

Note: There is a `--cloud-init` parameter for virt-install to auto-generate the
cloud-init ISO. However in Ubuntu 24.04 there's currently a bug in
virt-install <= 4.1.0 that
makes it usuable. So we manage the lifecycle manually.
https://github.com/virt-manager/virt-manager/issues/178

```bash
$ POOL_NAME=boot-scratch
$ virsh pool-define-as \
    --name "$POOL_NAME" \
    --type dir \
    --target /var/lib/libvirt/boot
$ virsh pool-build "$POOL_NAME"
$ virsh pool-start "$POOL_NAME"
$ virsh pool-autostart "$POOL_NAME"
```

```bash
$ virsh pool-list --all
$ virsh vol-list --pool "$POOL_NAME" --details
```

---
hideInToc: true
---

# Generate cloud-init ISO

```bash
sudo apt-get update
sudo apt-get install genisoimage
```

```bash
$ genisoimage \
    -input-charset utf-8 \
    -output ubuntu-server-2604-cloud-init.img \
    -volid cidata -rational-rock -joliet \
    user-data meta-data

sudo cp ubuntu-server-2604-cloud-init.img \
  /var/lib/libvirt/boot/ubuntu-server-2604-cloud-init.iso
```

Alternative way with cloud-localds:

```bash
sudo apt-get update
sudo apt-get install cloud-image-utils
```

```bash
sudo cloud-localds \
  /var/lib/libvirt/boot/ubuntu-server-2604-cloud-init.iso \
  user-data meta-data \
  --verbose
```

---
hideInToc: true
---

# Spin up image and configure with cloud-init

```bash
virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2604 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu-lts-latest \
  --disk /var/lib/libvirt/images/ubuntu-server-2604.qcow2,bus=virtio \
  --disk /var/lib/libvirt/boot/ubuntu-server-2604-cloud-init.iso,device=cdrom \
  --network network=default,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

# Virtual machine console

```bash
# Command line console
virsh console ubuntu-server-2604

# Graphical console
virt-viewer ubuntu-server-2604
```

---
hideInToc: true
---

# Disable cloud-init and remove cloud-init ISO

In the guest:
```bash
# login with ubuntu user
$ cloud-init status
status: done

# Disable cloud-init
$ sudo touch /etc/cloud/cloud-init.disabled

$ sudo apt-get update
$ sudo apt-get install qemu-guest-agent

$ sudo shutdown -h now
```

On the host
```bash
$ virsh domblklist ubuntu-server-2604
$ virsh change-media ubuntu-server-2604 sda --eject
$ sudo rm /var/lib/libvirt/boot/ubuntu-server-2604-cloud-init.iso
```


---
hideInToc: true
---


# Snapshots

```bash
$ virsh snapshot-create-as --domain ubuntu-server-2604 --name clean --description "Initial install"
$ virsh snapshot-list ubuntu-server-2604
$ virsh snapshot-revert ubuntu-server-2604 clean
$ virsh snapshot-delete ubuntu-server-2604 clean
```

# Cleanup

```bash
$ virsh shutdown ubuntu-server-2604
$ virsh undefine ubuntu-server-2604 --nvram --remove-all-storage
```

---
hideInToc: true
---

# Get IP of virtual machine

Preferred - use qemu-guest-agent

```bash
virsh start ubuntu-server-2604

virsh list --all
```

```
$ virsh domifaddr ubuntu-server-2604 --source agent
```

When using default NAT-based networking, can query dnsmasq

```
virsh net-dhcp-leases default
```

```bash
virsh domiflist ubuntu-server-2604
sudo apt-get install net-tools
arp -an
```

---
hideInToc: true
---

# If you need additional drivers

```bash
# First try to install the kernel modules package
sudo apt-get update
sudo apt-get install linux-modules-$(uname -r)

# sudo modprobe <kernel_module>

# If that still doesn't work, try installing the full generic kernel image
sudo apt-get install linux-image-generic
# linux-generic-hwe-26.04
sudo reboot
```

---
hideInToc: true
---

# Resource allocation

- Reserving a core for the host is a good idea
- Resource starvation can make a host inaccessible remotely
- Memory can be overcommitted
- Measure peak memory usage in each VM and add an overhead of 20% for host to manage guest

---
hideInToc: true
---

# Change number of CPUs

Change the CPUs offline
```bash
# Shutdown the VM
virsh shutdown <vm-name>
# Wait until vm is shut off
virsh list --all

# Edit the VM definition
virsh edit <vm name>
# Look for the <vcpu> tag and update the count
# You can also add a <cpu> block to define topology, features or NUMA if needed
<vcpu placement='static'>4</vcpu>

# Start the VM
virsh start <vm-name>
```

---
hideInToc: true
---

# Verify CPU settings

On host:
```bash
virsh dominfo <vm-name>
```

In guest:
```bash
# Print number of vCPUS
nproc
# More detail
lscpu
# Count cpu entries in /proc/cpuinfo
grep -c ^processor /proc/cpuinfo
```

---
hideInToc: true
---

# Changing memory

Change the memory offline

```bash
# Shutdown the VM
virsh shutdown <vm-name>
# Wait until vm is shut off
virsh list --all

# Edit the VM definition
virsh edit <vm name>
# Locate and update the <memory> and <currentMemory> elements (values in KiB):
# currentMemory - how much memory is used at boot
# memory - how much memory that can be hotplugged later if greater
# Normally you'll just make these two fields the same
<memory unit='KiB'>4194304</memory>         <!-- Max memory (e.g., 4 GiB) -->
<currentMemory unit='KiB'>2097152</currentMemory> <!-- Initial memory -->

# Start the VM
virsh start <vm-name>

# Verify settings:
virsh dominfo <vm-name>
```

---
hideInToc: true
---

# QCOW2

qcow2 stands for **QEMU Copy On Write version 2**, and it’s a disk image format used by QEMU/KVM virtual machines. It’s designed to be more flexible and efficient than raw disk images (.img, .raw).

---
hideInToc: true
---

# 🔍 Key Features of qcow2

| Feature             | Description |
| ------------------- | ----------- |
| Copy-on-write (COW) | Only changes from the base image are stored. Great for snapshots and templates. |
| Smaller size        | Sparse files — actual disk usage grows only as data is written. |
| Compression         | Supports zlib-based compression for smaller image size. |
| Snapshots           | Supports internal snapshots for point-in-time rollback. |
| Backing files       | Can base one image on another (ideal for golden images/templates). |

---
hideInToc: true
---

# 🧠 How it works

- A qcow2 image starts small.
- When the VM writes data, the image grows dynamically.
- You can layer qcow2 files by using a backing file, allowing multiple VMs to share a base OS image and store only their deltas.

Example:

```bash
qemu-img create -f qcow2 -b ubuntu-base.qcow2 vm1.qcow2
```

---
hideInToc: true
---

# ⚙️  Tools to work with qcow2

Inspect

```bash
qemu-img info disk.qcow2
```

Create

```bash
qemu-img create -f qcow2 disk.qcow2 20G
```

Convert from raw

```bash
qemu-img convert -f raw -O qcow2 disk.raw disk.qcow2
```

---
hideInToc: true
---

# Compacting qcow2 image files

Within the guest - zerofill the data on disk:
```
root@guest # sfill -fllvz
```

Then clone the image file:

```
qemu-img convert -p -O qcow2 ./source.img ./packed.img
```

---
hideInToc: true
---

# Mounting cloud images outside of a VM

There are multiple ways to mount image files:

- virt-customize
   - Provides simple image customization options through CLI parameters
   - Not as useful as one would hope because network won't be configured propery   - No default users are on cloud images
- If image is raw, you can just mount the image file
- mount+nbd
  - really fast
  - requires root
  - cannot mount read-only
- guestfish
  - does not need root
  - more complicated to use
  - more safe, can mount read-only

---
hideInToc: true
---

# virt-customize

```bash
sudo apt-get update
sudo apt-get install libguestfs-tools
```

```bash
$ curl -LO \
    https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img
$ qemu-img info resolute-server-cloudimg-amd64.img
```

```bash
$ sudo virt-customize \
    -a resolute-server-cloudimg-amd64.img \
    --root-password password:superseekret \
    --hostname resolute-vm \
    --run-command 'useradd -m -s /bin/bash autobot' \
    --password autobot:password:superseekret \
    --run-command 'usermod -aG sudo autobot'
[   0.0] Examining the guest ...
[  14.1] Setting a random seed
virt-customize: warning: random seed could not be set for this type of
guest
[  14.1] Setting the machine ID in /etc/machine-id
[  14.1] Setting passwords
[  15.0] SELinux relabelling
[  15.2] Finishing off
```

---
hideInToc: true
---

```bash
$ sudo qemu-img convert \
    -f qcow2 -O qcow2 \
    resolute-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/resolute-vm.qcow2
$ sudo qemu-img resize -f qcow2 \
    /var/lib/libvirt/images/resolute-vm.qcow2 \
    32G
```

```bash
virt-install \
  --connect qemu:///system \
  --name resolute-vm \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu24.04 \
  --disk /var/lib/libvirt/images/resolute-vm.qcow2,bus=virtio \
  --network network=default,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

```bash
# login with root
virsh console resolute-vm
```

---
hideInToc: true
---

# Mount cloud image without any special tooling

Install prerequisites:
```bash
sudo apt update
sudo apt install qemu-utils
```

If the image is in qcow2 format, convert it to raw:
```bash
$ qemu-img convert -f qcow2 -O raw \
    resolute-server-cloudimg-amd64.img \
    resolute-server-cloudimg-amd64.raw
```

---
hideInToc: true
---

Find the offset of where the partition starts:

```bash
$ fdisk -l resolute-server-cloudimg-amd64.raw
Disk resolute-server-cloudimg-amd64.raw: 3.5 GiB, 3758096384 bytes, 7340032 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7F8E2919-DFBA-400E-A316-9AE9885D44BE

Device                                 Start     End Sectors  Size Type
resolute-server-cloudimg-amd64.raw1  2324480 7339998 5015519  2.4G Linux root (x86-64)
resolute-server-cloudimg-amd64.raw13    2048 2097152 2095105 1023M Linux extended boot
resolute-server-cloudimg-amd64.raw14 2099200 2107391    8192    4M BIOS boot
resolute-server-cloudimg-amd64.raw15 2107392 2324479  217088  106M EFI System

Partition table entries are not in disk order.

# Offset is the second number times the sector size (usually 512 bytes)
$ echo $((2048 * 512))
1048576
```

---
hideInToc: true
---

```bash
# Mount the image
sudo mkdir /mnt/my_image
sudo mount -o loop,offset=1048576 \
  resolute-server-cloudimg-amd64.raw /mnt/my_image

# Image contents are available as /mnt/my_image

# Unmount the iamge
sudo umount /mnt/my_image
rmdir /mnt/my_image
```

---
hideInToc: true
---

# Mount cloud image with qemu-nbd - prerequisites

```bash
# nbd module not loaded by default

# Load the nbd module
sudo modprobe nbd
# Verify the nbd module is loaded
 lsmod | grep nbd
nbd                    65536  0
# Should see nbd devices
$ ls /dev/nbd*
/dev/nbd0   /dev/nbd11  /dev/nbd14  /dev/nbd3  /dev/nbd6  /dev/nbd9
/dev/nbd1   /dev/nbd12  /dev/nbd15  /dev/nbd4  /dev/nbd7
/dev/nbd10  /dev/nbd13  /dev/nbd2   /dev/nbd5  /dev/nbd8
```

---
hideInToc: true
---

# Mount cloud image with qemu-nbd

```bash
# Connect the QCOW2 image as a network block device
sudo qemu-nbd --connect=/dev/nbd0 resolute-server-cloudimg-amd64.img

# List partitions
$ sudo fdisk -l /dev/nbd0
Disk /dev/nbd0: 3.5 GiB, 3758096384 bytes, 7340032 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 131072 bytes
Disklabel type: gpt
Disk identifier: 7F8E2919-DFBA-400E-A316-9AE9885D44BE

Device         Start     End Sectors  Size Type
/dev/nbd0p1  2324480 7339998 5015519  2.4G Linux root (x86-64)
/dev/nbd0p13    2048 2097152 2095105 1023M Linux extended boot
/dev/nbd0p14 2099200 2107391    8192    4M BIOS boot
/dev/nbd0p15 2107392 2324479  217088  106M EFI System

Partition table entries are not in disk order.
```

---
hideInToc: true
---

# Mount cloud image with qemu-nbd

```bash
# Create a mount point directory
sudo mkdir /mnt/my_image
sudo mount /dev/nbd0p1 /mnt/my_image

$ ls /mnt/my_image
bin                etc    lib.usr-is-merged  opt   sbin                sys
bin.usr-is-merged  home   lost+found         proc  sbin.usr-is-merged  tmp
boot               lib    media              root  snap                usr
dev                lib64  mnt                run   srv                 var

# Use chroot jail to execute commands inside of the mounted root filesystem
$ sudo chroot /mnt/my_image

root@robot00:/# ls
bin                etc    lib.usr-is-merged  opt   sbin                sys
bin.usr-is-merged  home   lost+found         proc  sbin.usr-is-merged  tmp
boot               lib    media              root  snap                usr
dev                lib64  mnt                run   srv                 var

root@robot00:/# exit
exit
```

---
hideInToc: true
---

# nbd cleanup

```bash
# Check for active nbd entries
$ lsblk | grep '^nbd'

# Unmount the image file
$ sudo umount /mnt/my_image
$ sudo qemu-nbd --disconnect /dev/nbd0
```

---
hideInToc: true
---

# Mount cloud image with guestfish - prerequisites

```bash
sudo apt-get update
sudo apt-get install libguestfs-tools
```

---
layout: section
---

# Networking modes

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Networking modes

Libvirt requires configuration of virtual network switches, also known as
"bridges" to support different network operating modes
- https://wiki.libvirt.org/VirtualNetworking.html
- https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#
- https://docs.kernel.org/networking/switchdev.html

In the default NAT mode, computers external to the host can't communicate
with guest virtual machines. Guests can communicate with the outside
world, however, similar to how home internet connections work behind a
NAT gateway.

---
hideInToc: true
---

# What VirtualBox does that KVM doesn't

1. VirtualBox Runs as a privileged daemon

   - VirtualBox networking (including bridging is managed by the
     VBoxNetAdp/VBoxNetFlt kernel modules that run with elevated privileges.

   - These components hook into the host NIC at a lower level (via a
     proprietary filter driver).

   - This lets VirtualBox inject packets into the host NIC as if they came
     directly from the guest - without needing the host to have any special
     preconfigure bridge.

2. No Need for Pre-Existing Linux bridges

---

# macvtap

Create XML definition

```bash
cat <<EOF > /tmp/macvtap-network.xml
<network>
  <name>macvtap-network</name>
  <forward mode="bridge">
    <interface dev="enp1s0"/>
  </forward>
</network>
EOF
```

Configure

```bash
$ virsh net-define /tmp/macvtap-network.xml
$ virsh net-start macvtap-network
$ virsh net-autostart macvtap-network
```

Destroy

```bash
virsh net-destroy macvtap-network
virsh net-undefine macvtap-network
```

---
hideInToc: true
---

Verify
```bash
$ virsh net-list
 Name              State    Autostart   Persistent
----------------------------------------------------
 default           active   yes         yes
 macvtap-network   active   yes         yes
```

---
hideInToc: true
---

# Download cloud image template and resize

```bash
curl -LO https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img
```

```bash
$ qemu-img info resolute-server-cloudimg-amd64.img
$ sudo qemu-img convert \
    -f qcow2 -O qcow2 \
    resolute-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/ubuntu-server-2604.qcow2
$ sudo qemu-img resize -f qcow2 \
    /var/lib/libvirt/images/ubuntu-server-2604.qcow2 \
    32G
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-cloud-images-proxy/resolute/current/resolute-server-cloudimg-amd64.img
```
-->

---
hideInToc: true
---

```bash
# Required for NoCloud module to function, uniquely identifies instance
cat >meta-data <<EOF
instance-id: ubuntu-server-2604
local-hostname: ubuntu-server-2604
EOF
```

---
hideInToc: true
---

```bash
# Main configuration script, tells cloud-init what to do when instance starts
cat >user-data <<EOF
#cloud-config
hostname: ubuntu-server-2604
users:
  - name: autobot
    uid: 63112
    primary_group: users
    groups: users
    shell: /bin/bash
    plain_text_passwd: superseekret
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
chpasswd: { expire: False }
ssh_pwauth: True
package_update: False
package_upgrade: false
packages:
  - qemu-guest-agent
growpart:
  mode: auto
  devices: ['/']
power_state:
  mode: reboot
EOF
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2604 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu-lts-latest \
  --disk /var/lib/libvirt/images/ubuntu-server-2604.qcow2,bus=virtio \
  --cloud-init user-data=user-data,meta-data=meta-data,disable=on \
  --network network=macvtap-network,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

```bash
sudo ip link add link eno1 name macvtap0 type macvtap mode bridge
sudo ip link set macvtap0 up
```

🧠 macvtap0 acts like a NIC plugged into the LAN; eth0 must not be enslaved to a bridge.

- /dev/tapX (e.g., /dev/tap0)
- Associated macvtap0 interface bound to eth0

```
sudo ip link delete macvtap0
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2604 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu-lts-latest \
  --disk /var/lib/libvirt/images/ubuntu-server-2604.qcow2,bus=virtio \
  --cloud-init user-data=user-data,meta-data=meta-data,disable=on \
  --network type=direct,source=eno1,source_mode=bridge,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---

# br0 - Bridged Networking

```plantuml
@startnwdiag

nwdiag {

  network br0 {
    address = "10.67.132.2";

    robot01;
    ubuntu-server-2404 [ address = "10.67.132.3"];
  }
}
@endnwdiag
```

---
hideInToc: true
---

# Determine networking type

- NetworkManager - normally used on Ubuntu Desktop
- systemd-networkd - normally used on Ubuntu Server

---

NetworkManager check:
```bash
$ nmcli general
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN     METERED
connected  full          enabled  enabled  missing  enabled  no (guessed)

$ nmcli connection
NAME            UUID                                  TYPE      DEVICE
netplan-enp1s0  cac41fbe-bc18-3d87-bba7-af2af7f8ffab  ethernet  enp1s0
lo              0323f487-84dd-4ec1-bb47-75a4a0dc56ec  loopback  lo
virbr0          da53e323-660a-4f73-b1a7-6616079bebb5  bridge    virbr0

$ networkctl
IDX LINK   TYPE     OPERATIONAL SETUP
  1 lo     loopback -           unmanaged
  2 enp1s0 ether    -           unmanaged
  3 virbr0 bridge   -           unmanaged

3 links listed.
```

---
hideInToc: true
---
systemd-networkd check
```bash
$ networkctl
IDX LINK  TYPE     OPERATIONAL SETUP
  1 lo    loopback carrier     unmanaged
  2 ens33 ether    routable    configured

2 links listed.
```



---
hideInToc: true
---

# Configure bridged networking with NetworkManager (1 of 2)

```bash
$ ip -brief link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp1s0           UP             52:54:00:5a:22:d1 <BROADCAST,MULTICAST,UP,LOWER_UP>
virbr0           DOWN           52:54:00:33:85:7c <NO-CARRIER,BROADCAST,MULTICAST,UP>

$ nmcli connection show --active
NAME            UUID                                  TYPE      DEVICE
netplan-enp1s0  cac41fbe-bc18-3d87-bba7-af2af7f8ffab  ethernet  enp1s0
```

```bash
$ ls /etc/netplan
00-installer-config.yaml  01-network-manager-all.yaml
$ sudo cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp1s0:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: 52:54:00:5a:22:d1
      set-name: enp1s0
  version: 2
$ sudo cat /etc/netplan/01-network-manager-all.yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
```

---
hideInToc: true
---

# Configure bridged networking with NetworkManager (2 of 2)

```bash
cat <<'EOF' | sudo tee /etc/netplan/02-bridge.yaml > /dev/null
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    br0:
      dhcp4: true
      dhcp6: false
      accept-ra: false
      link-local: []
      interfaces:
        - enp1s0
EOF
sudo chmod 600 /etc/netplan/02-bridge.yaml
```

```
$ sudo netplan try
$ sudo netplan apply
```

---
hideInToc: true
---

# systemd-networkd -  Configure bridged networking (1 of 3)

```bash
$ ip -brief link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
ens33            UP             00:0c:29:b6:83:61 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

```bash
$ ls /etc/netplan
00-installer-config.yaml
$ cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: true
  version: 2
```

---
hideInToc: true
---

# systemd-networkd - Configure bridged networking (2 of 3)

```bash
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    br0:
      dhcp4: true
      dhcp6: false
      accept-ra: false
      link-local: []
      interfaces:
        - ens33
```

---
hideInToc: true
---

# systemd-networkd - Configure bridged networking (3 of 3)

```
$ sudo netplan try
$ sudo netplan apply
```

```bash
$ networkctl
IDX LINK  TYPE     OPERATIONAL SETUP
  1 lo    loopback carrier     unmanaged
  2 ens33 ether    enslaved    configured
  3 br0   bridge   routable    configured

3 links listed.

$ ip -brief addr
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens33            UP
br0              UP             172.25.0.112/22 metric 100 fe80::f4d4:91ff:feed:5e12/64
```

---
hideInToc: true
---

# Create a definition for the host network (1 of 2)

Create XML definition

```bash
cat <<EOF > /tmp/host-network.xml
<network>
  <name>host-network</name>
  <forward mode="bridge"/>
  <bridge name="br0" />
</network>
EOF
```

Configure

```bash
$ sudo virsh net-define /tmp/host-network.xml
$ sudo virsh net-start host-network
$ sudo virsh net-autostart host-network
```

---
hideInToc: true
---

# Create a definition for the host network (2 of 2)

Verify

```bash
$ virsh net-list --all
 Name           State    Autostart   Persistent
-------------------------------------------------
 default        active   yes         yes
 host-network   active   yes         yes
```

---
layout: section
---

# Ubuntu Server 26.04 cloud image

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Download cloud image template and resize

```bash
curl -LO https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img
```

```bash
$ qemu-img info resolute-server-cloudimg-amd64.img
$ sudo qemu-img convert \
    -f qcow2 -O qcow2 \
   resolute-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/ubuntu-server-2604.qcow2
$ sudo qemu-img resize -f qcow2 \
    /var/lib/libvirt/images/ubuntu-server-2604.qcow2 \
    32G
```

---
hideInToc: true
---

# Define login parameters for cloud-init ISO (1 of 2)

```bash
touch network-config

cat >meta-data <<EOF
instance-id: ubuntu-server-2604
local-hostname: ubuntu-server-2604
EOF
```

---
hideInToc: true
---

# Define login parameters for cloud-init ISO (2 of 2)

```bash
cat >user-data <<EOF
#cloud-config
hostname: ubuntu-server-2604
users:
  - name: autobot
    uid: 63112
    primary_group: users
    groups: users
    shell: /bin/bash
    plain_text_passwd: superseekret
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
chpasswd: { expire: False }
ssh_pwauth: True
package_update: False
package_upgrade: false
packages:
  - qemu-guest-agent
growpart:
  mode: auto
  devices: ['/']
power_state:
  mode: reboot
EOF
```

---
hideInToc: true
---

# Spin up image and configure with cloud-init (on host-network)

```bash
virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2604 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu-lts-latest \
  --disk /var/lib/libvirt/images/ubuntu-server-2604.qcow2,bus=virtio \
  --cloud-init user-data=user-data,meta-data=meta-data,disable=on \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

# Accessing image

```bash
virsh console ubuntu-server-2604
```

```bash
virt-viewer ubuntu-server-2604
```

---
hideInToc: true
---

# Verify cloud-init is disabled and qemu-guest-agent installed

```bash
# login with autobot user
$ cloud-init status --long
status: disabled
extended_status: disabled
boot_status_code: disabled-by-generator
detail: Cloud-init disabled by cloud-init-generator
errors: []
recoverable_errors: {}

$ qemu-ga --version
QEMU Guest Agent 10.2.1
```

---
hideInToc: true
---

# Troubleshooting issues with cloud-init

```bash
# Main output log
sudo cat /var/log/cloud-init-output.log

# System-wide cloud-init log
sudo cat /var/log/cloud-init.log

sudo cloud-init schema --system

cloud-init status --long

sudo cloud-init clean
sudo cloud-init init
sudo cloud-init modules --mode=config
sudo cloud-init modules --mode=final
```

---
hideInToc: true
---

# Snapshots

```bash
$ virsh snapshot-create-as --domain ubuntu-server-2604 --name clean --description "Initial install"
$ virsh snapshot-list ubuntu-server-2604
$ virsh snapshot-revert ubuntu-server-2604 clean
$ virsh snapshot-delete ubuntu-server-2604 clean
```

```bash
$ virsh shutdown ubuntu-server-2604
$ virsh undefine ubuntu-server-2604 --nvram --remove-all-storage
```

---
hideInToc: true
---

# Get IP of virtual machine (bridged networking)

```bash
virsh start ubuntu-server-2604

virsh list --all
```

```
virsh domifaddr ubuntu-server-2604 --source agent
```

```
# Does not work (only for NAT networking)
virsh net-dhcp-leases default
```

```
# Also does not work (only for NAT networking)
virsh domiflist ubuntu-server-2604
sudo apt-get install net-tools
arp -an
```

---
hideInToc: true
---

# If you need additional drivers

```
# First try to install the kernel modules package
sudo apt-get update
sudo apt-get install linux-modules-$(uname -r)

# sudo modprobe <kernel_module>

# If that still doesn't work, try installing the full generic kernel image
sudo apt-get install linux-image-generic
# linux-generic-hwe-24.04
sudo reboot
```

---
hideInToc: true
---

# Centos Stream 9

```bash
$ curl -LO https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-x86_64-9-latest.x86_64.qcow2.SHA256SUM
$ curl -LO https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-x86_64-9-latest.x86_64.qcow2
```

```bash
$ qemu-img info CentOS-Stream-GenericCloud-x86_64-9-latest.x86_64.qcow2

$ sudo qemu-img convert \
    -f qcow2 \
    -O qcow2 \
    CentOS-Stream-GenericCloud-x86_64-9-latest.x86_64.qcow2 \
    /var/lib/libvirt/images/centos-stream-9.qcow2
$ sudo qemu-img resize \
    -f qcow2 \
    /var/lib/libvirt/images/centos-stream-9.qcow2 \
    32G
```

---
hideInToc: true
---

```bash
cat >meta-data <<EOF
instance-id: centos-stream-9
local-hostname: centos-stream-9
EOF
```

---
hideInToc: true
---

```bash
cat >user-data <<EOF
#cloud-config
hostname: centos-stream-9
users:
  - name: automat
    uid: 63112
    primary_group: users
    groups: users
    shell: /bin/bash
    plain_text_passwd: superseekret
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
chpasswd: { expire: False }
ssh_pwauth: True
package_update: False
package_upgrade: false
packages:
  - qemu-guest-agent
growpart:
  mode: auto
  devices: ['/']
power_state:
  mode: reboot
EOF
```

---
hideInToc: true
---

```bash
sudo apt-get update
sudo apt-get install genisoimage
```

```bash
genisoimage \
    -input-charset utf-8 \
    -output centos-stream-9-cloud-init.img \
    -volid cidata -rational-rock -joliet \
    user-data meta-data
sudo cp centos-stream-9-cloud-init.img \
  /var/lib/libvirt/boot/centos-stream-9-cloud-init.iso
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name centos-stream-9 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant centos-stream9 \
  --disk /var/lib/libvirt/images/centos-stream-9.qcow2,bus=virtio \
  --disk /var/lib/libvirt/boot/centos-stream-9-cloud-init.iso,device=cdrom \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
layout: section
---

# CentOS Stream 10 cloud image

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Centos Stream 10

```
$ mkdir -p centos-stream-10
$ cd centos-stream-10
```

```bash
$ curl -LO https://cloud.centos.org/centos/10-stream/x86_64/images/CentOS-Stream-GenericCloud-x86_64-10-latest.x86_64.qcow2.SHA256SUM
$ curl -LO https://cloud.centos.org/centos/10-stream/x86_64/images/CentOS-Stream-GenericCloud-x86_64-10-latest.x86_64.qcow2
```

```bash
$ qemu-img info CentOS-Stream-GenericCloud-x86_64-10-latest.x86_64.qcow2

$ sudo qemu-img convert \
    -f qcow2 \
    -O qcow2 \
    CentOS-Stream-GenericCloud-x86_64-10-latest.x86_64.qcow2 \
    /var/lib/libvirt/images/centos-stream-10.qcow2
$ sudo qemu-img resize \
    -f qcow2 \
    /var/lib/libvirt/images/centos-stream-10.qcow2 \
    64G
```

---
hideInToc: true
---

```bash
cat >meta-data <<EOF
instance-id: centos-stream-10
local-hostname: centos-stream-10
EOF
```

---
hideInToc: true
---

```bash
cat >user-data <<EOF
#cloud-config
hostname: centos-stream-10
users:
  - name: autobot
    uid: 63112
    primary_group: users
    groups: users
    shell: /bin/bash
    plain_text_passwd: superseekret
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
chpasswd: { expire: False }
ssh_pwauth: True
package_update: False
package_upgrade: false
packages:
  - qemu-guest-agent
growpart:
  mode: auto
  devices: ['/']
power_state:
  mode: reboot
EOF
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name centos-stream-10 \
  --boot uefi \
  --memory 2048 \
  --vcpus 2 \
  --os-variant centos-stream10 \
  --disk /var/lib/libvirt/images/centos-stream-10.qcow2,bus=virtio \
  --cloud-init user-data=user-data,meta-data=meta-data,disable=on \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug
```

---
hideInToc: true
---

# Verify cloud-init disable/qemu-guest agent installed

```bash
# login with autobot user
$ sudo cloud-init status --long
status: disabled
extended_status: disabled
boot_status_code: disabled-by-generator
detail: Cloud-init disabled by cloud-init-generator
errors: []
recoverable_errors: {}

$ qemu-ga --version
QEMU Guest Agent 10.1.0
```

---
hideInToc: true
---

# Snapshots

```bash
$ virsh snapshot-create-as --domain centos-stream-10 --name clean --description "Initial install"
$ virsh snapshot-list centos-stream-10
$ virsh snapshot-revert centos-stream-10 clean
$ virsh snapshot-delete centos-stream-10 clean
$ virsh shutdown centos-stream-10
$ virsh undefine centos-stream-10 --nvram --remove-all-storage
```

---
hideInToc: true
---

# Get IP of virtual machine

```bash
virsh start ubuntu-server-2604

virsh list --all
virsh domifaddr ubuntu-server-2604 --source agent
# Does not work (only for NAT networking)
virsh net-dhcp-leases default
# Also does not work (only for NAT networking)
virsh domiflist ubuntu-server-2604
sudo apt-get install net-tools
arp -an
```

---
hideInToc: true
---

# Ubuntu autoinstall

Ubuntu Automatic installation lets you answer all configuration questions
ahead of time with an autoinstall configuration allowing an Ubuntu install
to run without any interaction.

Primarily used to automate OS installs on physical machines (a.k.a. metal).

https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html

- Script Ubuntu manual install with pre-configured answers
- Big install payload, slow to install
- Works on metal
- Complete driver set

---
hideInToc: true
---

# Autoinstall YAML

Autoinstall configuration is provided via an `autoinstall.yaml` file in
the following form:

```yaml
#cloud-config
autoinstall:
    version: 1
    ....
```

The `autoinstall.yaml` file can be provided on the installation ISO itself
or downloaded over the network.

A good starting point is to use the output of the Ubuntu installer from a
manual install. Every time the Ubuntu installer is used, even in manual
mode, it generates an autoinstall for repeating the installation in
`/var/log/installer/autoinstall-user-data`.

---
hideInToc: true
---

# Helpful tools for creating bootable USB sticks from ISO

- Ventoy: https://www.ventoy.net/en/index.html
- Balena Etcher: https://etcher.balena.io/

---
hideInToc: true
---

# Preqrequisites - ubuntu-autoinstall container image

We'll be using the `ubuntu-autoinstall` container image to create ISO images.
Creating a bootable ISO requires embedding special boot sectors which requires
root privileges. It's easier to assemble a temporary filesystem with
with all the necessary permissions inside a container image, and then discard
it once the ISO has been created.

```bash
$ docker run -it --rm docker.io/boxcutter/ubuntu-autoinstall -h
Usage:  /app/image-create.sh [options]

Ubuntu Autoinstall ISO generator

  -h, --help                Print help
  -s, --source PATH         Source ISO path
  -d, --destination PATH    Destination ISO path
  -a, --autoinstall PATH    Autoinstall config file
  -g, --grub PATH           Grub.cfg file
  -l, --loopback PATH       Loopback.cfg file
  -m, --metadata PATH       meta-data config file
  -N, --config-nocloud      Copy autoinstall config as NoCloud provider
  -R, --config-root         Copy autoinstall to ISO root
```

---
src: ./ubuntu-host/docker/docker-install-slides.md
---

---
src: ./ubuntu-host/ubuntu-server-2604-autoinstall/ubuntu-server-2604-autoinstall-slides.md
---

---
src: ./ubuntu-host/ubuntu-desktop-2604-autoinstall/ubuntu-desktop-2604-autoinstall-slides.md
---

---
layout: section
---

# QEMU

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# x86_64 - Download cloud image template

```bash
$ curl -LO \
    https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img

$ qemu-img info resolute-server-cloudimg-amd64.img

$ qemu-img convert \
    -f qcow2 -O qcow2 \
    resolute-server-cloudimg-amd64.img \
    ubuntu-server-2604.qcow2
$ qemu-img resize -f qcow2 \
    ubuntu-server-2604.qcow2 \
    32G
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-cloud-images-proxy/resolute/current/resolute-server-cloudimg-amd64.img
```
-->

---
hideInToc: true
---

# Define login parameters for cloud-init ISO

```bash
cat >meta-data <<EOF
instance-id: ubuntu-server-2604
local-hostname: ubuntu-server-2604
EOF

cat >user-data <<EOF
#cloud-config
password: superseekret
chpasswd:
  expire: false
ssh_pwauth: true
EOF
```

---
hideInToc: true
---

# Create the cloud-init ISO

Install cloud image utils

```bash
sudo apt-get update
sudo apt-get install cloud-image-utils
```
```bash
cloud-localds cloud-init.iso user-data meta-data
```

---
hideInToc: true
---

# x86_64 - Run the VM with QEMU on UEFI

```bash
qemu-system-x86_64 \
  -name ubuntu-image \
  -machine virt,accel=kvm,type=q35 \
  -cpu host \
  -smp 2 \
  -m 2G \
  -device virtio-keyboard \
  -device virtio-mouse \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -drive file=ubuntu-server-2604.qcow2,if=virtio,format=qcow2 \
  -cdrom cloud-init.iso \
  -drive if=pflash,format=raw,readonly=on,unit=0,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
  -drive if=pflash,format=raw,readonly=on,unit=1,file=/usr/share/OVMF/OVMF_VARS_4M.fd \
  -nographic
```

---
hideInToc: true
---

# x86_64 - Run the VM with QEMU booting with BIOS

```bash
qemu-system-x86_64 \
  -name ubuntu-image \
  -machine virt,accel=kvm,type=q35 \
  -cpu host \
  -smp 2 \
  -m 2G \
  -device virtio-keyboard \
  -device virtio-mouse \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -drive file=ubuntu-server-2604.qcow2,if=virtio,format=qcow2 \
  -cdrom cloud-init.iso \
  -nographic
```

---
hideInToc: true
---

# arm64 - Download cloud image template

```bash
curl -LO https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-arm64.img

$ qemu-img convert \
    -f qcow2 -O qcow2 \
    resolute-server-cloudimg-arm64.img \
    ubuntu-server-2604.qcow2
$ qemu-img resize -f qcow2 \
    ubuntu-server-2604.qcow2 \
    32G
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-cloud-images-proxy/resolute/current/resolute-server-cloudimg-amd64.img
```
-->

---
hideInToc: true
---

# arm64 - Define login parameters for cloud-init ISO

```bash
cat >meta-data <<EOF
instance-id: ubuntu-server-2604
local-hostname: ubuntu-server-2604
EOF

cat >user-data <<EOF
#cloud-config
password: superseekret
chpasswd:
  expire: false
ssh_pwauth: true
EOF
```

---
hideInToc: true
---

# arm64 - Create the cloud-init ISO

Install cloud image utils

```bash
sudo apt-get update
sudo apt-get install cloud-image-utils
```
```bash
cloud-localds cloud-init.iso user-data meta-data
```

---
hideInToc: true
---

# arm64 - Create firmware image

```bash
# Qemu expects aarch firmware images to be 64M so the firmware
# images can't be used as is, some padding is needed to
# create an image for pflash
dd if=/dev/zero of=flash0.img bs=1M count=64
dd if=/usr/share/AAVMF/AAVMF_CODE.fd of=flash0.img conv=notrunc
dd if=/dev/zero of=flash1.img bs=1M count=64
```

---
hideInToc: true
---

# arm64 - Run the VM with QEMU

```bash
qemu-system-aarch64 \
  -name ubuntu-image \
  -machine virt,accel=kvm,gic-version=3,kernel-irqchip=on \
  -cpu host \
  -smp 2 \
  -m 2G \
  -device virtio-keyboard \
  -device virtio-mouse \
  -device virtio-gpu-pci \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -drive file=ubuntu-server-2604.qcow2,if=virtio,format=qcow2 \
  -cdrom cloud-init.iso \
  -drive if=pflash,format=raw,readonly=on,unit=0,file=flash0.img \
  -drive if=pflash,format=raw,unit=1,file=flash1.img \
  -nographic
```

---
layout: section
---

# CI image pipelines with Hashicorp Packer

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

```bash
# Add Hashicorp's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/hashicorp-archive-keyring.gpg

# Add the repository to Apt sources
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list > /dev/null
sudo apt-get update

# To install the latest version
sudo apt install packer

# Verify the install
$ packer --version
Packer v1.12.0

# Install dependencies for generating cloud-init images
sudo apt-get update
sudo apt-get install genisoimage
```

---
hideInToc: true
---

```bash
sudo apt-get update
sudo apt-get install git
```

```bash
mkdir -p ~/github/boxcutter
cd ~/github/boxcutter
git clone https://github.com/boxcutter/kvm.git
cd kvm

cd ubuntu/cloud/x86_64
packer init .
PACKER_LOG=1 packer build \
  -var-file ubuntu-24.04-x86_64.pkrvars.hcl \
  ubuntu.pkr.hcl
```

```bash
$ sudo qemu-img convert \
    -f qcow2 \
    -O qcow2 \
    output-ubuntu-24.04-x86_64/ubuntu-24.04-x86_64.qcow2 \
    /var/lib/libvirt/images/ubuntu-server-2404.qcow2
$ sudo qemu-img resize \
    -f qcow2 \
    /var/lib/libvirt/images/ubuntu-server-2404.qcow2 \
    32G
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2404 \
  --boot uefi \
  --memory 4096 \
  --vcpus 2 \
  --os-variant ubuntu24.04 \
  --disk /var/lib/libvirt/images/ubuntu-server-2404.qcow2,bus=virtio \
  --network network=default,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --import \
  --debug

virsh console ubuntu-server-2404

# login with packer user
```

---
layout: section
---

# GPU

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Identifify the CPU

Look for the model name. If it says "with Radeon Graphics", then you have
a Ryzen APU with integrated GPU.

```bash
$ lscpu

Architecture:             x86_64
Vendor ID:                AuthenticAMD
  Model name:             AMD Ryzen 7 7840U w/ Radeon  780M Graphics
```

---
hideInToc: true
---

# Identify the GPU

```bash
lspci | grep i -vga
```

---
hideInToc: true
---

# Check with GPU is active

```bash
$ sudo lshw -c video
  *-display
       description: VGA compatible controller
       product: Phoenix1
       vendor: Advanced Micro Devices, Inc. [AMD/ATI]
       physical id: 0
       bus info: pci@0000:05:00.0
       version: c4
       width: 64 bits
       clock: 33MHz
       capabilities: pm pciexpress msi msix vga_controller bus_master cap_list
       configuration: driver=amdgpu latency=0
       resources: iomemory:7f0-7ef irq:61 memory:7fe0000000-7fefffffff memory:fbe00000-fbffffff ioport:f000(size=256) memory:fc400000-fc47ffff
```

---
hideInToc: true
---

```bash
# Get PCI vendor and device IDs for NVIDIA GPU devices
$ lspci -nnk | grep -i nvidia
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3060 Ti Lite Hash Rate] [10de:2489] (rev a1)
	DeviceName:  WIFI
	Subsystem: ASUSTeK Computer Inc. GA104 [GeForce RTX 3060 Ti Lite Hash Rate] [1043:884f]
	Kernel driver in use: nouveau
	Kernel modules: nvidiafb, nouveau
01:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
	Subsystem: ASUSTeK Computer Inc. GA104 High Definition Audio Controller [1043:884f]
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel

01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3060 Ti Lite Hash Rate] [10de:2489] (rev a1)
	Kernel modules: nvidiafb, nouveau
01:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
```

---
hideInToc: true
---

# vfio-pci

vfio-pci is a kernel driver that allows user-space programs (like QEMU) to
safely and securely access PCI devices directly, bypassing the host OS drivers.

It’s the core mechanism used for PCI passthrough, such as giving a GPU, NIC,
or USB controller directly to a virtual machine.

✅ What vfio-pci Does

1. Claims ownership of a PCI device early in boot or on request.
2. Detaches the device from its default Linux driver (e.g., nouveau, i915, xhci_hcd, etc.).
3. Gives user-space (e.g., QEMU/KVM) secure access to the device’s MMIO, I/O ports, interrupts, and DMA.
4. Prevents host kernel access to the device while assigned to a VM.

---
hideInToc: true
---

# During early kernel boot, make sure PCI devices are reserved early before initramfs or other drivers bind, as a precaution
```bash
cat /etc/default/grub |grep iommu
GRUB_CMDLINE_LINUX="intel_iommu=on vfio-pci.ids=10de:2489,10de:228b
```

```bash
sudo update-grub
```

# When the vfio-pci module is loaded, make sure that the PCI devices are reserved, before other drivers can bind to the devices
```
$ sudo sh -c 'echo "options vfio-pci ids=10de:2489,10de:228b disable_vga=1" > /etc/modprobe.d/vfio.conf'
```

---
hideInToc: true
---

---
hideInToc: true
---

# Streaming

- NICE DCV
- NoMachine
- Parsec
- HP RGS

---
hideInToc: true
---

# VA-API and VDPAU via mesa

```bash
sudo apt-get update
sudo apt-get install \
  mesa-va-drivers \
  vainfo \
  vdpauinfo \
  libva-drm2 \
  libva-x11-2 \
  ffmpeg
```

```bash
sudo apt update
sudo apt install ffmpeg vainfo intel-media-va-driver-non-free  # or mesa-va-drivers for AMD
```

---
hideInToc: true
---

```bash
sudo usermod -aG render $USER
newgrp render
```

```bash
$ vainfo --display drm
libva info: VA-API version 1.20.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
libva info: Found init function __vaDriverInit_1_20
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.20 (libva 2.12.0)
vainfo: Driver version: Mesa Gallium driver 24.2.8-1ubuntu1~24.04.1 for AMD Radeon 780M (radeonsi, gfx1103_r1, LLVM 19.1.1, DRM 3.61, 6.11.0-26-generic)
vainfo: Supported profile and entrypoints
      VAProfileH264ConstrainedBaseline:	VAEntrypointVLD
      VAProfileH264ConstrainedBaseline:	VAEntrypointEncSlice
      VAProfileH264Main               :	VAEntrypointVLD
      VAProfileH264Main               :	VAEntrypointEncSlice
      VAProfileH264High               :	VAEntrypointVLD
      VAProfileH264High               :	VAEntrypointEncSlice
      VAProfileHEVCMain               :	VAEntrypointVLD
      VAProfileHEVCMain               :	VAEntrypointEncSlice
      VAProfileHEVCMain10             :	VAEntrypointVLD
      VAProfileHEVCMain10             :	VAEntrypointEncSlice
      VAProfileJPEGBaseline           :	VAEntrypointVLD
      VAProfileVP9Profile0            :	VAEntrypointVLD
      VAProfileVP9Profile2            :	VAEntrypointVLD
      VAProfileAV1Profile0            :	VAEntrypointVLD
      VAProfileAV1Profile0            :	VAEntrypointEncSlice
      VAProfileNone                   :	VAEntrypointVideoProc
```

---
hideInToc: true
---

```bash
ffmpeg -init_hw_device vaapi=va:/dev/dri/renderD128 \
  -filter_hw_device va \
  -f lavfi -i testsrc=duration=5:size=1280x720:rate=30 \
  -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi \
  vaapi_test.mp4
```

```bash
ffmpeg -vaapi_device /dev/dri/renderD128 \
  -hwaccel vaapi -hwaccel_output_format vaapi \
  -i input.mp4 -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi -b:v 5M output.mp4
```

---
hideInToc: true
---

# Windows

- https://sysguides.com/install-a-windows-11-virtual-machine-on-kvm
- https://www.youtube.com/watch?v=7tqKBy9r9b4
- https://blandmanstudios.medium.com/tutorial-the-ultimate-linux-laptop-for-pc-gamers-feat-kvm-and-vfio-dee521850385

```bash
SHIFT+F10 OOBE\BYPOASSNRO
SHIFT+F10 start ms-cxh:localonly
```

---
src: ./talos/talos-slides.md
---

---
src: ./kairos/kairos-slides.md
---
