# PHẦN I — CHUẨN BỊ SERVER VẬT LÝ (RUNBOOK)

## Áp dụng
- Server A / B / C  
- Chạy tuần tự hoặc Ansible sau khi test OK

---

## 🔹 BƯỚC 1 — CẬP NHẬT HỆ THỐNG
sudo apt update && sudo apt full-upgrade -y
sudo reboot

uname -r

---

## 🔹 BƯỚC 2 — CÀI PACKAGES NỀN TẢNG
sudo apt install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients \
  virtinst libguestfs-tools cloud-image-utils genisoimage \
  openvswitch-switch openvswitch-common \
  lvm2 thin-provisioning-tools gdisk parted \
  bridge-utils dnsmasq \
  iptables iptables-persistent netfilter-persistent \
  curl wget git vim htop iotop net-tools tcpdump

---

## 🔹 BƯỚC 3 — CẤU HÌNH USER + SERVICE
sudo usermod -aG libvirt,kvm $USER

sudo systemctl enable --now libvirtd

virsh net-destroy default
virsh net-undefine default

virsh net-list --all

sudo systemctl enable --now openvswitch-switch
sudo ovs-vsctl show

---

## 🔹 BƯỚC 4 — BẬT NESTED KVM
cat /sys/module/kvm_intel/parameters/nested

echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-nested.conf

sudo modprobe -r kvm_intel && sudo modprobe kvm_intel

cat /sys/module/kvm_intel/parameters/nested

---

## 🔹 BƯỚC 5 — BẬT IOMMU
sudo nano /etc/default/grub

GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"

sudo update-grub && sudo reboot

dmesg | grep "Intel-IOMMU" | head -3

ls /sys/kernel/iommu_groups/ | wc -l

---

## 🔹 BƯỚC 6 — LVM THIN POOL

lsblk -d -o NAME,SIZE,TYPE,MOUNTPOINT

sudo pvcreate /dev/sdb
sudo vgcreate vg-lab /dev/sdb

sudo lvcreate \
  --type thin-pool \
  --extents 95%VG \
  --name thin-pool \
  vg-lab

sudo lvs

cat > /tmp/pool-vg-lab.xml << EOF
<pool type="logical">
  <name>vm-pool</name>
  <source>
    <name>vg-lab</name>
    <format type="lvm2"/>
  </source>
  <target>
    <path>/dev/vg-lab</path>
  </target>
</pool>
EOF

virsh pool-define /tmp/pool-vg-lab.xml
virsh pool-start vm-pool
virsh pool-autostart vm-pool

virsh pool-list --all

---

## 🔹 (OPTION) CEPH OSD
sudo wipefs -a /dev/sdb

sudo ceph orch daemon add osd server-a:/dev/sdb
sudo ceph orch daemon add osd server-b:/dev/sdb
sudo ceph orch daemon add osd server-c:/dev/sdb
