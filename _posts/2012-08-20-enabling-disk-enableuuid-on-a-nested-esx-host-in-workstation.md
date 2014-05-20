---
title: Enabling disk.EnableUUID on a Nested ESX Host in Workstation
author: Karim Elatov
layout: post
permalink: /2012/08/enabling-disk-enableuuid-on-a-nested-esx-host-in-workstation/
dsq_thread_id:
  - 1404673163
categories:
  - Home Lab
  - Storage
  - VMware
tags:
  - disk.enableuuid
  - esxcfg-volume
  - ForceMount VMFS
  - Persistent Mount VMFS
  - Resignature VMFS
  - Snapshot LUN
  - VMFS UUID
  - VPD Pages
  - vsd-mount
---
As I was writing this [blog post](http://virtuallyhyper.com/2012/08/determine-disk-vpd-information-from-esx-classic) about VPD (Vital Product Data) Pages, I ended up breaking my test ESX host for a little bit. By default virtual disks presented to a VM don't have VPD Pages and therefore don't have NAA IDs. For example here is how my disk looks like from the ESX host without any changes:


	# esxcfg-scsidevs -m
	mpx.vmhba0:C0:T0:L0:5 /dev/sda5 502e8a75-d04a58db-431a-000c29d33d34 0 Storage1


In fact here are how all the devices look like:


	# vim-cmd /hostsvc/summary/scsilun
	Device Type Vendor Model UUID Hba
	mpx.vmhba0:C0:T0:L0 disk VMware, VMware Virtual S 0000000000766d686261303a303a30 vmhba0
	mpx.vmhba32:C0:T0:L0 cdrom NECVMWar VMware IDE CDR10 0005000000766d68626133323a303a30 vmhba32


Any device that has an MPX identifier is a device that didn't have an NAA ID or any other Unique Identifier. More information can be seen at VMware KB [1014953](http://kb.vmware.com/kb/1014953), from that KB:

> Some devices do not provide the NAA number described above. In these circumstances, an MPX Identifier is generated by ESX to represent the LUN or disk. The identifier takes the form similar to that of the canonical name of previous versions of ESX with the mpx. prefix. This identifier can be used in the exact same way as the NAA Identifier described above.

Finally querying the device for any VPD Pages, I would see the following:


	# sg_vpd -p 0x80 /dev/sda
	Unit serial number VPD page:
	fetching VPD page failed

	# sg_vpd -p 0x83 /dev/sda
	Device Identification VPD page:
	fetching VPD page failed


When I was writing the <a title="Missing links to devices on Linux VMs under /dev/disk/by-id/" href="http://virtuallyhyper.com/2012/03/missing-links-to-devices-on-linux-vms-under-devdiskby-id/">post</a> about missing links under /dev/disks/by-id for linux VMs, I remember running into this. The fix is to add the following to the vmx file:


	disk.EnableUUID = "TRUE"


So I went ahead and first found out where my vmx file is for my ESX VM in Workstation:


	$ vmrun list
	Total running VMs: 1
	/home/elatov/.vmware/ESXU2/ESX4U2.vmx


Next I shutdown the VM:


	$ vmrun stop /home/elatov/.vmware/ESXU2/ESX4U2.vmx soft


I then added the new setting:


	$ echo 'disk.EnableUUID = "TRUE"' >> /home/elatov/.vmware/ESXU2/ESX4U2.vmx


Now let's check it's there:


	$ grep disk /home/elatov/.vmware/ESXU2/ESX4U2.vmx
	disk.EnableUUID = "TRUE"


That looks good, so let's go ahead and power on the VM:


	$ vmrun start /home/elatov/.vmware/ESXU2/ESX4U2.vmx


Upon booting the VM I saw the following:

![boot-issue-after-enableuuid](http://virtuallyhyper.com/wp-content/uploads/2012/08/boot-issue-after-enableuuid.png)

checking out the logs, I saw the following:


	/bin/sh: can't access tty; job control turned off
	/ # tail messages.log
	FILE: File_GetVMFSfsType: File_GetVMFSAttributes failed
	DISNLIB-DSCPTR: DescriptorDetermineType: failed to open '/vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34/esxconsole-4c87dcc7-73b8-71d6-02e2-000c29d33d34/esxconsole.vmdk': Could not find the file (63)
	DISKLIB-LIHK : ''/vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34/esxconsole-4c87dcc7-73b8-71d6-02e2-000c29d33d34/esxconsole.vmdk'': failed to open (The system cannot find the file specified).
	DISKLIB-CHAIH: ''/vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34/esxconsole-4c87dcc7-73b8-71d6-02e2-000c29d33d34/esxconsole.vmdk'': failed to open (The system cannot find the file specified)
	DISKLIB-LIB : Failed to open '/vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34/esxconsole-4c87dcc7-73b8-71d6-02e2-000c29d33d34/esxconsole.vmdk' with flags 0xa The system cannot find the file specified (25)
	VSDCreateVirtualDev: Failed to open '/vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34/esxconsole-4c87dcc7-73b8-71d6-02e2-000c29d33d34/esxconsole.vmdk' : The system cannot find the file specified (25).
	AIOMGR-S: stat o=1 r=0 w=0 i=0 br=0 bw=0
	sysboot: Failed to Mount VMDK Disk: /vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34/esxconsole-4c87dcc7-73b8-71d6-02e2-000c29d33d34/esxconsole.vmdk
	sysboot: 66.vsd-mount returned a recoverable error
	sysboot: Executing 'chvt 1'
	/#


Looks like we can't load **console.vmdk** (our service console), which should be located under the vmfs volume **/vmfs/volumes/502e8a75-d04a58db-431a-000c29d33d34**. Checking to see if that file exists, I noticed that the vmfs volume wasn't even mounted:


	/ # ls -l /vmfs/volumes
	/ #


The reason why it wasn't mounted is because I just changed the virtual disk to have an NAA ID and this will of course make ESX see this virtual disk as a snapshot. More information on snapshots can be found in VMware KB [1011387](http://kb.vmware.com/kb/1011387)

> This issue is caused when the ESX/ESXi host can't confirm the identity of the LUN with what we expect to see in the VMFS metadata. This can be caused by replaced SAN hardware, firmware upgrades, SAN replication, DR tests, some HBA firmware upgrades or some ESX/ESXi host upgrades from 3.5 to 4.x (due to the change in naming convention from mpx to naa) have been known to cause this but this is a rare occurrence.

Basically upon formatting a vmfs volume a UUID is generated from the VPD pages and is stored in the VMFS meta data. You can check the UUID of a LUN in the following manner:


	# vmkfstools -Ph /vmfs/volumes/Storage1/
	VMFS-3.33 file system spanning 1 partitions.
	File system label (if any): Storage1
	Mode: public
	Capacity 8.8 GB, 1013 MB available, file block size 1 MB
	UUID: 502e8a75-d04a58db-431a-000c29d33d34
	Partitions spanned (on "lvm"):
	mpx.vmhba0:C0:T0:L0:5


Upon re-scanning a LUN, if we find the UUID doesn't match the UUID stored in the VMFS meta data we determine a device to be a snapshot.

Here are some common causes for LUNs to be seen as snapshots:

1.  The LUN presentation has been changed on the Array (e.g. SPC-2 bit set on an EMC Symmetrix Director, firmware upgrade on certain arrays)‏.
2.  The ESX servers are defined in different initiator groups/storage groups and the presentation LUN ID is different.
3.  A LUN has been un-presented and a new LUN has been presented back with the same LUN ID but the SAN has not been scanned in the meantime.
4.  A LUN has been snapshot’ed on the array and presented back to the ESX server (a real snapshot)

Most of these are discussed in VMware KB [6482648](http://kb.vmware.com/kb/6482648).
I checked to see if my local drive is seen as a snapshot:


	/ # esxcfg-volume -l
	VMFS3 UUID/label: 502e8a75-d04a58db-431a-000c29d33d34/Storage1
	Can Mount: Yes
	Can resignature: Yes
	Extent name: naa.6000c29e28495d0b9ecd280a8dc01a5c:5 range: 0 - 8959 (MB)


and that was the case. When working with snapshots there are usually three things to do:

> 1_. Mount the volume without performing a resignature of that volume (this volume is unmounted when the ESX host is rebooted), run this command:
>
> 	# esxcfg-volume -m

Or you can

> 2_. Persistently mount the volume without performing a resignaturing of that volume (this volume is mounted when the ESX host is rebooted), run this command:
>
>	# esxcfg-volume -M

And lastly you can

> 3_. Resignature the volume (the volume is mounted immediately after the resignature):
>
>	# esxcfg-volume -r

These are all take from VMware KB [1011387](http://kb.vmware.com/kb/1011387). Option 1 didn't really suit me cause I wanted my fix to be permanent, so I will discuss options 2 and 3.

### Resignature the VMFS Volume


	/ # esxcfg-volume -r Storage1
	Resignaturing volume Storage1
	/ # ls -l /vmfs/volumes
	drwxr-xr-t 1 root root 1120 Mar 11 2011 502f7868-f1a09a22-baa9-000c29d33d34
	lrwxr-xr-x 1 root root 35 Aug 18 06:29 snap-69a82519-Storage1 -> 502f7868-f1a09a22-baa9-000c29d33d34


Upon resignaturing we can see that the volume is now mounted, and also has the word 'snap-xx' (**snap-69a82519-Storage1**) in front of the original Datastore name which was Storage1. Also you will notice that a new UUID is generated (**502f7868-f1a09a22-baa9-000c29d33d34 **).If you ever resignature a volume you will have to re-register your VMs that were on this volume. The reason for this is because the vmx files holds the datastore by UUID and not by datastore name. This way you can rename your Datastore and this will never impact any running or registered VMs. In our case the only VM that we had was the service console VM. So we will have to edit */etc/vmware/esx.conf* file to update the new UUID. First find the old UUID:


	/ # grep console /etc/vmware/esx.conf | cut -d '=' -f 2 | cut -d '/' -f 4 | uniq
	502e8a75-d04a58db-431a-000c29d33d34


Next find your new UUID:


	/ # vmkfstools -Ph /vmfs/volumes/snap-69a82519-Storage1/ | grep UUID | awk '{print $2}'
	502f7868-f1a09a22-baa9-000c29d33d34


Now let's go ahead and store those values in variables and use sed to update the *esx.conf* file accordingly:


	/ # old=$(grep console /etc/vmware/esx.conf | cut -d '=' -f 2 | cut -d '/' -f 4 | uniq)
	/ # new=$(vmkfstools -Ph /vmfs/volumes/snap-69a82519-Storage1/ | grep UUID | awk '{print $2}')
	/ # echo $old
	502e8a75-d04a58db-431a-000c29d33d34
	/ # echo $new
	502f7868-f1a09a22-baa9-000c29d33d34


That looks good, now for sed and to confirm it looks good:


	/ # sed -i ''s/$old/$new/g'' /etc/vmware/esx.conf
	/ # grep console /etc/vmware/esx.conf | cut -d '=' -f 2 | cut -d '/' -f 4 | uniq
	502f7868-f1a09a22-baa9-000c29d33d34
	/ # vmkfstools -Ph /vmfs/volumes/snap-69a82519-Storage1/ | grep UUID | awk '{print $2}'
	502f7868-f1a09a22-baa9-000c29d33d34


Everything looks good. Go ahead and type 'exit' and if everything is good the ESX host should continue to boot. Upon booting you can check to make sure it's mounted appropriately:


	# esxcfg-scsidevs -m
	naa.6000c29e28495d0b9ecd280a8dc01a5c:5 /dev/sda5 502f7868-f1a09a22-baa9-000c29d33d34 0 snap-69a82519-Storage1


If you want to rename the datastore to the original name, you can do the following:


	# ln -sf `readlink -f /vmfs/volumes/snap-69a82519-Storage1` /vmfs/volumes/Storage1


and then make sure the name looks good:


	# esxcfg-scsidevs -m
	naa.6000c29e28495d0b9ecd280a8dc01a5c:5 /dev/sda5 502f7868-f1a09a22-baa9-000c29d33d34 0 Storage1


Or you could actually persistently mount the vmfs volume without worrying about the UUID or resignaturing.

### Persistently Mount the VMFS Volume without Resignaturing

Check for the snapshot'ed LUN:


	* nfs [ ok ]
	* cos-snapshot [ ok ]
	* vsd-Mount [ !! ]

	You have entered the recovery shell.The situation you are in May
	be recoverable. If you are able to fix this situation the boot
	process will continue normally after you exit this terminal
	/bin/sh: can't access tty; job control turned off
	/ # esxcfg-volume -l
	VMFS3 UUID/label: 502e8a75-d04a58db-431a-000c29d33d34/Storage1
	Can mount: Yes
	Can resignature: Yes
	Extent name: naa.6000c29e28495d0b9ecd280a8dc01a5c:5 range: 0 - 8959 (MB)


Then go ahead and forcefully mount it and make it persistent across reboots:


	/ # ls -l /vmfs/volumes
	/ # esxcfg-volume -M Storage1
	Persistently Mounting volume Storage1
	/ # ls -l /vmfs/volumes
	drwxr-xr-t 1 root root 1120 Mar 11 2011 502e8a75-d04a58db-431a-000c29d33d34
	lrwxr-xr-x 1 root root 35 Aug 18 07:05 Storage1 -> 502e8a75-d04a58db-431a-000c29d33d34


Notice the UUID (**502e8a75-d04a58db-431a-000c29d33d34**) didn't change and the datastore name (**Storage1**) didn't change either. Then go ahead and type in 'exit' and it will continue to boot. Then you can check that it's still mounted:


	# esxcfg-scsidevs -m
	naa.6000c29e28495d0b9ecd280a8dc01a5c:5 /dev/sda5 502e8a75-d04a58db-431a-000c29d33d34 0 Storage1


Whenever you forcefully mount a VMFS volume the following settings are added to the */etc/vmware/esx.conf* file:


	# grep forceMountedLvm /etc/vmware/esx.conf
	/fs/vmfs[502e8a75-d04a58db-431a-000c29d33d34]/forceMountedLvm/forceMount = "true"
	/fs/vmfs[502e8a75-d04a58db-431a-000c29d33d34]/forceMountedLvm/lvmName = "snap-53c719bd-4c87ae60-9c9b2d60-413f-000c29d33d34"
	/fs/vmfs[502e8a75-d04a58db-431a-000c29d33d34]/forceMountedLvm/readOnly = "false"


This is usually not recommended because if you actually present a snapshot of this disk, it will go ahead and forcefully mount which is probably not the desired affect. So please use the persistent mount option with care. As a side note VMware KB [1012142](http://kb.vmware.com/kb/1012142) talks about similar fixes as above, but their steps are a little different.

Either option you choose, after it's said and done you should see your local disk identified with an NAA ID instead of the mpx ID:


	# vim-cmd hostsvc/summary/scsilun
	Device Type Vendor Model UUID Hba
	naa.6000c29e28495d0b9ecd280a8dc01a5c disk VMware, VMware Virtual S 02000000006000c29e28495d0b9ecd280a8dc01a5c564d77617265 vmhba0
	mpx.vmhba32:C0:T0:L0 cdrom NECVMWar VMware IDE CDR10 0005000000766d68626133323a303a30 vmhba32


Notice your local drive has an NAA ID while your cd-rom remained with the mpx ID.
