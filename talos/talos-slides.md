---
layout: section
---

# Talos

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

## Install Talos

---
hideInToc: true
---

### Generate an install iso

You need to add some custom parameters/extensions to the image:
- Go to the Talos image factory: https://factory.talos.dev/
  * Hardware Type: Bare-metal Machine
  * Latest stable Talos version
  * Marchine Architecture: amd64
  * System Extensions: qemu-guest-agent, util-linux-tools
  * Customization: Extra kernel command line arguments: console=ttyS0,115200
  * Customization: UEFI only
 
At "schematic ready" download the ISO.

|> ISO used because initial install boots in RAM-only live mode, and then the
|> disk is configured.

---
hideInToc: true
---

```bash
mkdir talos-cluster
cd talos-cluster

curl -LO https://factory.talos.dev/image/9385ba106a3a7d3eff277f44bf0468b6de749769beee4ff47951f31b4955d334/v1.12.3/metal-amd64.iso
shasum -a 256 metal-amd64.iso
sudo cp metal-amd64.iso /var/lib/libvirt/iso/metal-amd64.iso
```

```bash
sudo mkdir -p /mnt/iso
sudo mount -o loop,ro metal-amd64.iso /mnt/iso

sudo umount /mnt/iso
```

---
hideInToc: true
---

```bash
virt-install \
  --connect qemu:///system \
  --name talos-cluster \
  --boot loader=/usr/share/OVMF/OVMF_CODE_4M.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/OVMF/OVMF_VARS_4M.fd \
  --cdrom /var/lib/libvirt/iso/metal-amd64.iso \
  --disk pool=default,format=qcow2,bus=virtio,size=64 \
  --memory 4096 \
  --vcpus 4 \
  --os-variant linux2022 \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole

# Talos boots into RAM-only live mode
virt-viewer talos-cluster &
```

---
hideInToc: true
---

```bash
# Make sure you grab the same version of talosctl
$ curl -LO https://github.com/siderolabs/talos/releases/download/v1.12.3/talosctl-linux-amd64
$ shasum -a 256 talosctl-linux-amd64
1b761b351964dcf764d3ec1a02ccd366428be22895a68dc641b6b4fa301589ed  talosctl-linux-amd64

sudo cp talosctl-linux-amd64 /usr/local/bin/talosctl
sudo chmod +x /usr/local/bin/talosctl

$ talosctl version
Client:
	Tag:         v1.12.3
	SHA:         6d6471f6
	Built:       
	Go version:  go1.25.7
	OS/Arch:     linux/amd64
Server:
error constructing client: failed to determine endpoints
```

---
hideInToc: true
---

```bash
# Get IP address of the VM running qemu-guest-agent
virsh domifaddr talos-cluster --source agent

# get IP from console
export CONTROL_PLANE_IP=10.63.34.66

$ talosctl get disks --insecure --nodes "$CONTROL_PLANE_IP"
NODE   NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID   MODEL          SERIAL
       runtime     Disk   loop0   2         4.1 kB   true                                                       
       runtime     Disk   loop1   2         102 kB   true                                                       
       runtime     Disk   loop2   2         692 kB   true                                                       
       runtime     Disk   loop3   2         75 MB    true                                                       
       runtime     Disk   sr0     2         215 MB   false       sata        true                QEMU DVD-ROM   
       runtime     Disk   vda     2         69 GB    false       virtio      true

export DISK_ID=vda

# Generate cluster configuration
talosctl gen config talos-cluster https://$CONTROL_PLANE_IP:6443 --install-disk /dev/$DISK_ID -o configs/

cat >single-node-cluster-patch.yaml <<EOF
cluster:
  allowSchedulingOnControlPlanes: true
EOF

# Apply the configuration to bring the nodes and cluster online
talosctl apply-config --insecure \
  --nodes $CONTROL_PLANE_IP \
  --file configs/controlplane.yaml \
  --config-patch @single-node-cluster-patch.yaml
```

---
hideInToc: true
---

```bash
# Bootstrap etcd in the cluster
talosctl bootstrap --nodes $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP --talosconfig configs/talosconfig

talosctl dashboard --nodes $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP --talosconfig configs/talosconfig

talosctl logs -f -n $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP --talosconfig configs/talosconfig etcd
talosctl -n $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP --talosconfig configs/talosconfig service
```

---
hideInToc: true
---

```bash
# Make sure the version of kubectl you use is within one minor difference of your cluster
# Kubernetes: v1.35.0

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

talosctl -n $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP --talosconfig configs/talosconfig kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
$ kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
talos-j06-rk3   Ready    control-plane   8m54s   v1.35.0

$ kubectl get pods -A
NAMESPACE     NAME                                    READY   STATUS    RESTARTS      AGE
kube-system   coredns-7859998f6-fff9n                 1/1     Running   0             11m
kube-system   coredns-7859998f6-jzz6n                 1/1     Running   0             11m
kube-system   kube-apiserver-talos-j06-rk3            1/1     Running   0             11m
kube-system   kube-controller-manager-talos-j06-rk3   1/1     Running   2 (12m ago)   11m
kube-system   kube-flannel-hfrgr                      1/1     Running   0             11m
kube-system   kube-proxy-wlkwm                        1/1     Running   0             11m
kube-system   kube-scheduler-talos-j06-rk3            0/1     Running   2 (12m ago)   11m
```

---
hideInToc: true
---

```bash
virsh destroy talos-cluster
virsh undefine talos-cluster --nvram --remove-all-storage
```

---
hideInToc: true
---

References:
- https://docs.siderolabs.com/talos/v1.11/platform-specific-installations/virtualized-platforms/kvm
- https://unixorn.github.io/post/homelab/k8s/01-talos-with-cilium-cni-on-proxmox/
