---
layout: post
title: Convert VM disk format
date: 2023-05-25 15:00 +800
categories: [Hypervisors,VMware]
tags: [vmware]
---

![vm-banner](https://www.techtarget.com/visuals/searchServerVirtualization/infrastructure_architecture/servervirtualization_article_016.jpg)

## Convert VHDX To VMDK

### Virtual Box to convert 
[Download](https://www.virtualbox.org/wiki/Downloads)

Open Command Prompt as admin and run this command:
```
cd C:\Program Files\Oracle\VirtualBox
```
Next run the following command with the correct paths to the vhdx file and where you want to store the copy
```
.\VBoxManage.exe clonehd --format vmdk C:\temp\My-VM.vhdx С:\temp\My-VM.vmdk
```

### QEMU-IMG
- Windows
[Download](https://cloudbase.it/qemu-img-windows/)

- Linux
```
sudo apt-get install qemu-utils
```

Open a Command Prompt as Admin to the location of the program and you can run this command to view the disk info:
```
qemu-img.exe info c:\temp\My-VM.vhd
```

Here is an example of how to convert a vhdx file:
```
qemu-img.exe convert -p c:\temp\My-VM.vhdx -O vmdk c:\temp\My-VM.vmdk
```

## Converting VMDK to the ESXi format
Upload the converted VMDK file to the datastore of ESXi
![upload](https://www.nakivo.com/blog/wp-content/uploads/2020/01/How-to-import-VHD-to-VMware-ESXi-%E2%80%93-uploading-the-VMDK-file-after-conversion.png)

Enable SSH access on your ESXi host (Manage > Services > TSM-SSH)
![ssh-enabled](https://www.nakivo.com/blog/wp-content/uploads/2020/01/SSH-access-is-enabled-on-an-ESXi-host.png)

Go to the directory where the VMDK was uploaded
`cd /vmfs/volumes/SSD2/converted`

Use vmkfstools to convert VMDK to ESXi format.
```
vmkfstools -i My-VM.vmdk My-VMthin.vmdk -d thin
```

As you can see on the directory, the conversion  with vmkfstools has completed successfully and two new files have been created:

My-VMthin-flat.vmdk

My-VMthin.vmdk

## Importing a VMDK disk to a VM on ESXi
1. Select Virtual Machines and hit Create/Register VM.

2. Select creation type > Create a new virtual machine. Click Next on each step to continue.

3. Select a name and guest OS

4. Select storage.

5. Customize settings. Delete the virtual disk that was created by default with the new VM. Then, click Add hard disk > Existing hard disk.
![customize](https://www.nakivo.com/blog/wp-content/uploads/2020/01/Importing-a-virtual-disk-converted-from-VHD-to-VMDK-to-a-VM-on-ESXi.png)

6. Select the converted My-VMthin.vmdk 

7. Start your Virtual Machine now.

## Additional
You can download the [vCenter Converter](https://www.vmware.com/products/converter.html) to do as well
This tool is easy to do P2V and V2V process
