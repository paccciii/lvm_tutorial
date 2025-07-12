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
![Alt text](/images/inital_disk_attached_lsblk.png)

##  Step 3: Create Physical Volume and Volume Group

```bash
sudo pvcreate /dev/vdb
sudo vgcreate vg_data /dev/vdb
sudo vgdisplay vg_data
```
![Alt text](/images/vg_pv_creation.png)


##  Step 4: Create Logical Volume and Format

```bash
sudo lvcreate -L 10G -n lv_data vg_data
sudo lvdisplay
```

![Alt text](/images/lvdisplay.png)

Create a new ext4 filesystem on the logical volume /dev/vg_data/lv_data.

```bash
sudo mkfs.ext4 /dev/vg_data/lv_data
```

##  Step 5: Mount and Use the Volume

```bash
sudo mkdir /data
sudo mount /dev/vg_data/lv_data /data
```
![Alt text](/images/final.png)

Adding the UUID entry to the /etc/fstab file ensures that the logical volume is automatically mounted at boot time. Without this, you'd need to manually mount the volume every time the system restarts.

Using the UUID (instead of device paths like /dev/vg_data/lv_data) is a more reliable and persistent method, as device names can change across reboots or if hardware changes occur. The UUID uniquely identifies the filesystem and helps avoid mount failures due to dynamic device naming.

![Alt text](/images/fstab_entry.png)

---

##  Step 6: Add Another Disk and Extend LV

Create and attach a second disk:

```bash
qemu-img create -f qcow2 -o preallocation=metadata \
  /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk3.qcow2 20G

virsh attach-disk --domain LVM-Tutorial-VM \
  --source /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk3.qcow2 \
  --target vdc --persistent --targetbus virtio
```
![Alt text](/images/kvm_disk-addtion_ss.png)

Inside VM:

```bash
sudo pvcreate /dev/vdc
sudo vgextend vg_data /dev/vdc
sudo lvextend -l +100%FREE /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data
```
![Alt text](/images/complete_setup_second_disk.png)

---

##  Step 7: Add Third Disk and Reserve for Snapshots

```bash
qemu-img create -f qcow2 -o preallocation=metadata \
  /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk4.qcow2 30G

virsh attach-disk --domain LVM-Tutorial-VM \
  --source /mnt/qa220250516104235_prashant_test_July_03/LVM-Tutorial-VM_disk4.qcow2 \
  --target vdd --persistent --targetbus virtio
```
Now I have added another disk size of 30G vdd for showcasing the snapshot feature



Inside VM:

```bash
sudo pvcreate /dev/vdd
sudo vgextend vg_data /dev/vdd
sudo lvextend -L +15G /dev/vg_data/lv_data
sudo resize2fs /dev/vg_data/lv_data
```
![Alt text](/images/snapshot_disk.png)

---

Now create a file in the /data directory 
```bash
cd /data
touch demo.txt
echo "Initial content" | tee demo.txt

```
![Alt text](/images/file_creattion.png)


##  Step 8: LVM Snapshot Demo

Create snapshot:

```bash
sudo lvcreate -L 10G -s -n lv_data_snap /dev/vg_data/lv_data
```
![Alt text](/images/snapshot_done.png)


Modify data under `/data`...

![alt text](/images/oops.png)

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

![alt text](/images/snapshot_restoration.png)

---

## Detach Disk (optional cleanup)

```bash
virsh detach-disk --domain LVM-Tutorial-VM --target vdb --persistent
virsh detach-disk --domain LVM-Tutorial-VM --target vdc --persistent
virsh detach-disk --domain LVM-Tutorial-VM --target vdd --persistent
```

---


##  Conclusion

You successfully:

- Created a VM
- Attached multiple disks
- Configured LVM from scratch
- Extended LVs live
- Demonstrated LVM snapshot and recovery

> Authored by Prashant S.B.

