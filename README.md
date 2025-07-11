# LVM Hands-on Tutorial: Extend Disks, Create LVMs, Snapshots

This tutorial walks through a complete LVM demonstration using a KVM virtual machine. It includes:

- Creating a VM
- Attaching new disks
- Creating Physical Volumes (PVs), Volume Groups (VGs), Logical Volumes (LVs)
- Mounting and resizing
- Creating and restoring LVM snapshots

---

# First thing first, we will clone any Linux VM. In my case, I am going with Ubuntu 24.04 CLI

Clone the master VM image:

```bash
virt-clone --original <source-vm> --name LVM-Tutorial-VM \
  --file <destination-path>/LVM-Tutorial-VM.qcow2
```

##  Step 1: Create VM from Master Copy

```bash
virsh start LVM-Tutorial-VM
```

##  Step 2: Create and Attach First Additional Disk

```bash
qemu-img create -f qcow2 -o preallocation=metadata \
  /<path-to-your-disk>/LVM-Tutorial-VM_disk2.qcow2 20G

virsh attach-disk --domain LVM-Tutorial-VM \
  --source /<path-to-your-disk>/LVM-Tutorial-VM_disk2.qcow2 \
  --target vdb --persistent --targetbus virtio
```

Login into VM:

```bash
virsh console LVM-Tutorial-VM
```

## 🔧 Step 3: Create Physical Volume and Volume Group

```bash
sudo pvcreate /dev/vdb
sudo vgcreate vg_data /dev/vdb
```
![Alt text](/images/vg_pv_creation.png)


## 🧱 Step 4: Create Logical Volume and Format

```bash
sudo lvcreate -L 10G -n lv_data vg_data
sudo mkfs.ext4 /dev/vg_data/lv_data
```

## 📁 Step 5: Mount and Use the Volume

```bash
sudo mkdir /data
sudo mount /dev/vg_data/lv_data /data
```

---

## ➕ Step 6: Add Another Disk and Extend LV

Create and attach a second disk:

```bash
qemu-img create -f qcow2 -o preallocation=metadata \
  /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk3.qcow2 20G

virsh attach-disk --domain LVM-Tutorial-VM \
  --source /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk3.qcow2 \
  --target vdc --persistent --targetbus virtio
```

Inside VM:

```bash
sudo pvcreate /dev/vdc
sudo vgextend vg_data /dev/vdc
sudo lvextend -l +100%FREE /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data
```

---

## ➕ Step 7: Add Third Disk and Reserve for Snapshots

```bash
qemu-img create -f qcow2 -o preallocation=metadata \
  /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk4.qcow2 30G

virsh attach-disk --domain LVM-Tutorial-VM \
  --source /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk4.qcow2 \
  --target vdd --persistent --targetbus virtio
```

Inside VM:

```bash
sudo pvcreate /dev/vdd
sudo vgextend vg_data /dev/vdd
sudo lvextend -L +15G /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data
```

---

## 📸 Step 8: LVM Snapshot Demo

Create snapshot:

```bash
sudo lvcreate -L 2G -s -n lv_data_snap /dev/vg_data/lv_data
```

Modify data under `/data`...

Restore from snapshot:

```bash
sudo umount /data
# If busy:
sudo fuser -km /data

sudo lvchange -an /dev/vg_data/lv_data
sudo lvconvert --merge /dev/vg_data/lv_data_snap
sudo lvchange -ay /dev/vg_data/lv_data
sudo mount /dev/vg_data/lv_data /data
```

---

## ❌ Detach Disk (optional cleanup)

```bash
virsh detach-disk --domain LVM-Tutorial-VM --target vdb --persistent
virsh detach-disk --domain LVM-Tutorial-VM --target vdc --persistent
virsh detach-disk --domain LVM-Tutorial-VM --target vdd --persistent
```

---

## 📂 Delete Disk Files (optional cleanup)

```bash
rm -f /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk{2,3,4}.qcow2
```

---

## ✅ Conclusion

You successfully:

- Created a VM
- Attached multiple disks
- Configured LVM from scratch
- Extended LVs live
- Demonstrated LVM snapshot and recovery

Perfect for LinkedIn posts, blogs, and internal training!

> *Created by Prashant S B* 🚀

