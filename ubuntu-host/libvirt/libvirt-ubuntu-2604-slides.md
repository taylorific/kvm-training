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

# Install QEMU/KVM on Ubuntu

Install QEMU/KVM and libvirtd

```bash
# Ubuntu 26.04
sudo apt-get update
# Ubuntu 26.04 split qemu-kvm into qemu-system-x86-hwe and qemu-system-x86.
# The hwe variant is updated more frequently, every 6 months
# https://ubuntu.com/server/docs/how-to/virtualisation/virt-hwe/
sudo apt-get install qemu-system-hwe libvirt-daemon-system-hwe
sudo apt-get install virtinst

# Ubuntu 24.04
sudo apt-get update
sudo apt-get install qemu-kvm libvirt-daemon-system
sudo apt-get install virtinst
```

Make sure the current user is a member of the libvirt and kvm groups

```bash
# Gives permission to manage virtual machines with virsh
$ sudo adduser $(id -un) libvirt
Adding user '<username>' to group 'libvirt' ...
# Gives permission to access the /dev/kvm device
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

---
hideInToc: true
---

# These `virt-host-validate` warnings are benign (1 of 2)

```
QEMU: Checking for cgroup 'devices' controller support : WARN
```

## cgroup 'devices' controller

Modern Ubuntu uses cgroup v2, which does not have a separate devices
controller like cgroup v1 did.

Verify that you are using cgroup v2:
```bash
stat -fc %T /sys/fs/cgroup
# cgroup2fs
```

---
hideInToc: true
---

# These `virt-host-validate` warnings are benign (2 of 2)

```
QEMU: Checking for secure guest support                : WARN
```

## secure guest support

This checks for optional confidential VM features:

* AMD: SEV / SEV-ES / SEV-SNP
* Intel: TDX

These are not required for normal KVM/QEMU virtualization. Older CPUs
don't have this functionality.

---
hideInToc: true
---

# Default Network (1 of 2)

Configured by `libvirt-daemon-config-network` package

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

---
hideInToc: true
---

# Default Network (2 of 2)

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

# Default image pool

Ubuntu 26.04 DOES NOT create a pool when KVM is installed by
`libvirt-daemon-common-hwe`. It only creates the
`/var/lib/libvirt/images` directory. It's easiest to create
your own default pool.

```bash
$ virsh pool-define-as \
    --name default \
    --type dir \
    --target "/var/lib/libvirt/images"
$ virsh pool-build default
$ virsh pool-start default
$ virsh pool-autostart default
```

```bash
$ virsh pool-list --all
$ virsh vol-list --pool default --details
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
