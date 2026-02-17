---
layout: section
---

# Ubuntu Desktop 24.04 autoinstall

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

## Prepare autoinstall ISO - Ubuntu Desktop 24.04

```bash
$ mkdir -p ~/github/boxcutter
$ cd ~/github/boxcutter
$ git clone https://github.com/boxcutter/kvm.git
$ cd ~/github/boxcutter/kvm/autoinstall/generic/ubuntu-desktop-2404/headless

$ curl -LO https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso
$ shasum -a 256 ubuntu-24.04.4-desktop-amd64.iso
3a4c9877b483ab46d7c3fbe165a0db275e1ae3cfe56a5657e5a47c2f99a99d1e  ubuntu-24.04.4-desktop-amd64.iso

$ docker pull docker.io/boxcutter/ubuntu-autoinstall
$ export ISO_SOURCE="ubuntu-24.04.4-desktop-amd64.iso"
$ export ISO_DESTINATION="ubuntu-24.04.4-desktop-amd64-autoinstall.iso"
$ docker run -it --rm \
  --mount type=bind,source="$(pwd)",target=/data \
  docker.io/boxcutter/ubuntu-autoinstall \
    --autoinstall autoinstall.yaml \
    --config-root \
    --source "$ISO_SOURCE" \
    --destination "$ISO_DESTINATION"

# Verify autoinstall.yaml exists in root
$ isoinfo -R -i \
    "$ISO_DESTINATION" -f | grep -i autoinstall
/autoinstall.yaml
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-releases-proxy/noble/ubuntu-24.04.4-desktop-amd64.iso
```
-->

---
hideInToc: true
---

## Test in a VM - Ubuntu Desktop 24.04 autoinstall

```bash
export UBUNTU_DESKTOP_ISO="ubuntu-24.04.4-desktop-amd64-autoinstall.iso"
sudo cp "$UBUNTU_DESKTOP_ISO" \
  "/var/lib/libvirt/iso/$UBUNTU_DESKTOP_ISO"

export VM_NAME=ubuntu-desktop-2404
export VM_MEMORY=8096
export VM_VCPU=4

virsh vol-create-as default "$VM_NAME.qcow2" 64G --format qcow2

virt-install \
  --connect qemu:///system \
  --name "$VM_NAME" \
  --boot uefi \
  --cdrom "/var/lib/libvirt/iso/$UBUNTU_DESKTOP_ISO" \
  --memory "$VM_MEMORY" \
  --vcpus "$VM_VCPU" \
  --cpu mode=host-passthrough \
  --os-variant ubuntu24.04 \
  --disk vol=default/$VM_NAME.qcow2,bus=virtio,cache=none,discard=unmap \
  --network network=host-network,model=virtio \
  --graphics vnc,listen=0.0.0.0,password=foobar \
  --video qxl \
  --noautoconsole \
  --console pty,target_type=serial \
  --debug
```

---
hideInToc: true
---

```bash
# View locally:
virt-viewer
```


```bash
# View remotely
virsh vncdisplay ubuntu-desktop-2404
virsh dumpxml ubuntu-desktop-2404 | grep "graphics type='vnc'"

# vnc to server on port  to complete install
# Get the IP address of the default host interface
ip addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | grep -v 127.0.0.1
# Use a vnc client to connect to `vnc://<host_ip>:5900`
# When the install is complete the VM will be shut down
```

---
hideInToc: true
---

```bash
$ virsh domblklist ubuntu-desktop-2404
$ virsh change-media ubuntu-desktop-2404 sda --eject

# Reconfigure VNC
virsh edit ubuntu-desktop-2404
<graphics type='vnc' port='-1' autoport='yes' listen='127.0.0.1' passwd='foobar'/>
<graphics type='none'/>
virsh restart ubuntu-desktop-2404

$ virsh start ubuntu-desktop-2404

# Optional - Enable serial console access
# https://ravada.readthedocs.io/en/latest/docs/config_console.html
# enable serial service in VM
sudo systemctl enable --now serial-getty@ttyS0.service

# Install acpi or qemu-guest-agent in the vm so that
# 'virsh shutdown <image>' works
$ sudo apt-get update
$ sudo apt-get install qemu-guest-agent

$ virsh domifaddr ubuntu-desktop-2404 --source agent
```

---
hideInToc: true
---

# Snapshots and cleanup

```bash
export VM_NAME=ubuntu-desktop-2404
$ virsh snapshot-create-as --domain "$VM_NAME" --name clean --description "Initial install"
$ virsh snapshot-list "$VM_NAME"
$ virsh snapshot-revert "$VM_NAME" clean
$ virsh snapshot-delete "$VM_NAME" clean
```

```bash
export VM_NAME=ubuntu-desktop-2404
$ virsh shutdown "$VM_NAME"
$ virsh undefine "$VM_NAME" --nvram --remove-all-storage
```