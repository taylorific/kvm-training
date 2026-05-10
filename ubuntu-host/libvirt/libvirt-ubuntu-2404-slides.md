---
layout: section
---

# Install and configure libvirt (Ubuntu host)

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Install QEMU/KVM on Ubuntu 24.04

Install QEMU/KVM and libvirtd

```bash
sudo apt-get update
sudo apt-get install qemu-kvm libvirt-daemon-system
sudo apt-get install virtinst
```

Make sure the current user is a member of the libvirt and kvm groups

```bash
$ sudo adduser $(id -un) libvirt
Adding user '<username>' to group 'libvirt' ...
$ sudo adduser $(id -un) kvm
Adding user '<username>' to group 'kvm' ...
```

Be sure to reboot!!!!

```bash
sudo reboot
```

---
hideInToc: true
---

# Validate config

```bash
$ virt-host-validate qemu
```

X86_64-based machines will likely display a warning about cgroup devices controller support not being enabled. This allows you to apply resource management to virtual machines. For more information refer to this doc. To add cgroup 'devices' controller support, edit /etc/default/grub and change the line that looks like GRUB_CMDLINE_LINUX_DEFAULT="quiet splash" to:

```bash
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on systemd.unified_cgroup_hierarchy=0"
```

```bash
# amd
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on iommu=pt systemd.unified_cgroup_hierarchy=0"
```

```bash
sudo update-grub
```

---
hideInToc: true
---

# Default Network

```bash
$ cat /usr/share/libvirt/networks/default.xml
<network>
  <name>default</name>
  <bridge name='virbr0'/>
  <forward/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

```bash
$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp113s0         DOWN
eno1             UP             10.67.132.223/22 metric 100 fe80::a6ae:11ff:fe1e:48fa/64
wlp4s0           DOWN
virbr0           DOWN           192.168.122.1/24
```

```bash
$ virsh net-list --all
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

---
hideInToc: true
---

# Image pool

```bash
$ POOL_NAME=default
$ virsh pool-define-as \
    --name "$POOL_NAME" \
    --type dir \
    --target "/var/lib/libvirt/$POOL_NAME"
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

# cloud-init image pool

Note: There is a `--cloud-init` parameter for virt-install to auto-generate the
cloud-init ISO. However there's currently a bug in virt-install <= 4.1.0 that
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

# ISO image pool

```bash
$ POOL_NAME=iso
$ virsh pool-define-as \
    --name "$POOL_NAME" \
    --type dir \
    --target "/var/lib/libvirt/$POOL_NAME"
$ virsh pool-build "$POOL_NAME"
$ virsh pool-start "$POOL_NAME"
$ virsh pool-autostart "$POOL_NAME"
```

```bash
$ virsh pool-list --all
$ virsh vol-list --pool "$POOL_NAME" --details
```
