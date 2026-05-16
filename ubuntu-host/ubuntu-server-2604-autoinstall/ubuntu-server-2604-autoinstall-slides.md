---
layout: section
---

# Ubuntu Server 26.04 autoinstall

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

## Prepare autoinstall ISO - Ubuntu Server 26.04

```bash
$ mkdir -p ~/github/boxcutter
$ cd ~/github/boxcutter
$ git clone https://github.com/boxcutter/kvm.git
$ cd ~/github/boxcutter/kvm/autoinstall/generic/ubuntu-server-2604/headless

$ curl -LO https://releases.ubuntu.com/resolute/ubuntu-26.04-live-server-amd64.iso
$ shasum -a 256 ubuntu-26.04-live-server-amd64.iso
dec49008a71f6098d0bcfc822021f4d042d5f2db279e4d75bdd981304f1ca5d9  ubuntu-26.04-live-server-amd64.iso

$ docker pull docker.io/boxcutter/ubuntu-autoinstall
$ export UBUNTU_SERVER_26_04_ISO="ubuntu-26.04-live-server-amd64.iso"
$ export UBUNTU_SERVER_26_04_ISO_AUTOINSTALL="ubuntu-26.04-live-server-amd64-autoinstall.iso"
$ docker run -it --rm \
  --mount type=bind,source="$(pwd)",target=/data \
  docker.io/boxcutter/ubuntu-autoinstall \
    -a autoinstall.yaml \
    -g grub.cfg \
    --config-root \
    -s "$UBUNTU_SERVER_26_04_ISO" \
    -d "$UBUNTU_SERVER_26_04_ISO_AUTOINSTALL"

# Verify autoinstall.yaml exists in root
$ isoinfo -R -i \
    "$UBUNTU_SERVER_26_04_ISO_AUTOINSTALL" -f | grep -i autoinstall
/autoinstall.yaml
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-releases-proxy/resolute/ubuntu-26.04-live-server-amd64.iso
```
-->

---
hideInToc: true
---

## Ubuntu Server 24.04 autoinstall - test in a VM

```bash
export UBUNTU_SERVER_26_04_ISO_AUTOINSTALL="ubuntu-26.04-live-server-amd64-autoinstall.iso"
sudo cp "$UBUNTU_SERVER_26_04_ISO_AUTOINSTALL" \
  "/var/lib/libvirt/iso/$UBUNTU_SERVER_26_04_ISO_AUTOINSTALL"

export VM_NAME=ubuntu-server-2604
export VM_MEMORY=2048
export VM_VCPU=2
virsh vol-create-as default "$VM_NAME.qcow2" 64G --format qcow2

virt-install \
  --connect qemu:///system \
  --name "$VM_NAME" \
  --boot uefi \
  --cdrom "/var/lib/libvirt/iso/$UBUNTU_SERVER_26_04_ISO_AUTOINSTALL" \
  --memory "$VM_MEMORY" \
  --vcpus "$VM_VCPU" \
  --os-variant ubuntu-lts-latest \
  --disk vol=default/$VM_NAME.qcow2,bus=virtio \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --debug

# To watch the install. Once it completes it will stop the vm:
$ virsh console ubuntu-server-2604
$ virsh-viewer ubuntu-server-2604
```
