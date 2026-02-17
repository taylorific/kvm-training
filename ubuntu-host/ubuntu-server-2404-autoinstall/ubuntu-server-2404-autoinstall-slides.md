---
layout: section
---

# Ubuntu Server 24.04 autoinstall

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Ubuntu Server 24.04 autoinstall - prepare ISO

```bash
mkdir -p ~/github/boxcutter && cd ~/github/boxcutter
git clone https://github.com/boxcutter/kvm
cd ~/github/boxcutter/kvm/autoinstall/generic/kvm/ubuntu-server-2404

$ curl -LO https://releases.ubuntu.com/noble/ubuntu-24.04.3-live-server-amd64.iso
$ shasum -a 256 ubuntu-24.04.3-live-server-amd64.iso
c3514bf0056180d09376462a7a1b4f213c1d6e8ea67fae5c25099c6fd3d8274b  ubuntu-24.04.3-live-server-amd64.iso

docker pull docker.io/boxcutter/ubuntu-autoinstall
docker run -it --rm \
  --mount type=bind,source="$(pwd)",target=/data \
  docker.io/boxcutter/ubuntu-autoinstall \
    -a autoinstall.yaml \
    -g grub.cfg \
    --config-root \
    -s ubuntu-24.04.3-live-server-amd64.iso \
    -d ubuntu-24.04.3-live-server-amd64-autoinstall.iso

# Verify autoinstall.yaml exists in root
$ isoinfo -R -i \
    ubuntu-24.04.3-desktop-autoinstall.iso -f | grep -i autoinstall
/autoinstall.yaml
```

<!--
```
curl -LO \
  https://crake-nexus.org.boxcutter.net/repository/ubuntu-releases-proxy/noble/ubuntu-24.04.3-live-server-amd64.iso
```
-->

---

# Ubuntu Server 24.04 autoinstall - test in a VM

```bash
sudo cp ubuntu-24.04.3-live-server-amd64.iso \
  /var/lib/libvirt/iso/ubuntu-server-2404-autoinstall.iso

virsh vol-create-as default ubuntu-server-2404.qcow2 50G --format qcow2

virt-install \
  --connect qemu:///system \
  --name ubuntu-server-2404 \
  --boot uefi \
  --cdrom /var/lib/libvirt/iso/ubuntu-server-2404-autoinstall.iso \
  --memory 2048 \
  --vcpus 2 \
  --os-variant ubuntu24.04 \
  --disk vol=default/ubuntu-server-2404.qcow2,bus=virtio \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole \
  --console pty,target_type=serial \
  --debug

# To watch the install. Once it completes it will stop the vm:
$ virsh console ubuntu-server-2404
$ virsh-viewer ubuntu-server-2404
```# 
