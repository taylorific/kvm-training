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
sudo cp ubuntu-24.04.4-desktop-amd64.iso \
  /var/lib/libvirt/iso/ubuntu-24.04.4-desktop-amd64.iso

virsh vol-create-as default ubuntu-desktop-2404.qcow2 64G --format qcow2

virt-install \
  --connect qemu:///system \
  --name ubuntu-desktop-2404 \
  --boot uefi \
  --cdrom /var/lib/libvirt/iso/ubuntu-24.04.4-desktop-amd64.iso \
  --memory 4096 \
  --vcpus 2 \
  --cpu mode=host-passthrough \
  --os-variant ubuntu24.04 \
  --disk vol=default/ubuntu-desktop-2404.qcow2,bus=virtio,cache=none,discard=unmap \
  --network network=host-network,model=virtio \
  --graphics vnc,listen=0.0.0.0,password=foobar \
  --video qxl \
  --noautoconsole \
  --console pty,target_type=serial \
  --debug
```
