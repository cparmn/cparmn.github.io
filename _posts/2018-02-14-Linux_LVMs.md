---
published: true
---
# Adding a Disk to existing LVM

Adding a disk to an existing LVM takes a two step approach as you first need to gather information about the current configuration.
The process described below takes place after the disk has been added to the Host Machine.

## Information Gathering 
Before working with a specific disk, you should verify disk is attached to the system correctly, using the `fdisk` command 
In this case we know the Full path of the disk and I'll use the `fdisk -l /dev/mapper/mpathf` if you dont know the the disk you can use `fdisk -l` to view all disk 

The expected output will show the disk, with no partitions created. 

```bash
Disk /dev/mapper/mpathf: 5497.6 GB, 5497558138880 bytes
255 heads, 63 sectors/track, 668373 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```


You'll also need to understand the information of the LVM.  This can be accomplished by looking at three different parts of LVM.  

1. Physical Volumes (PV)
  These are the physical disk attached to the system output of `fdisk -l`
  To view Physical Volumes use `pvs` or `pvdisplay` 
2. Volume Groups (VG)
  Volume Groups are a collection of Physical VOlumes This allows multiple Physical volumes to exist in a single Volume Group. 
  To view Volume Group information use `vgs` or `vgdisplay`
3. Logical Volumes (LV)
  Logical Volume exist inside of a volume group, you're allowed to have multiple Logical Volumes within a single Volume group.  T

Viewing Physical Volumes 
---
`pvs`
```
 PV                 VG             Fmt  Attr PSize   PFree
  /dev/mapper/mpathb HotStorage     lvm2 a--u   4.00t    0 
  /dev/mapper/mpathc HotStorage     lvm2 a--u   1.50t    0 
```

`pvdisplay`
```
 --- Physical volume ---
  PV Name               /dev/mapper/mpathb
  VG Name               HotStorage
  PV Size               4.00 TiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              1048575
  Free PE               0
  Allocated PE          1048575
  PV UUID               gT5p2h-raov-zB0q-FNii-Ttim-6cEp-dpxOMs
   
  --- Physical volume ---
  PV Name               /dev/mapper/mpathc
  VG Name               HotStorage
  PV Size               1.50 TiB / not usable 4.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              393215
  Free PE               0
  Allocated PE          393215
  PV UUID               GvXD22-VZuM-eYm0-Gw6r-SU40-vkw6-VlPkAU
```

Viewing Volume Groups
---
`vgs`
```
  VG             #PV #LV #SN Attr   VSize   VFree
  HotStorage       2   1   0 wz--n-   5.50t    0 
  vg_cbtresponse   1   3   0 wz--n- 278.38g    0 
```
`vgdisplay`
```
  --- Volume group ---
  VG Name               HotStorage
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               5.50 TiB
  PE Size               4.00 MiB
  Total PE              1441790
  Alloc PE / Size       1441790 / 5.50 TiB
  Free  PE / Size       0 / 0   
  VG UUID               IjIpql-4sw3-UiRZ-2ekU-FRkp-nafh-VN9IPl
```


Viewing Logical Volumes
---
`lvs`
```
  LV      VG             Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  events  HotStorage     -wi-ao----   5.50t      
```

`lvdisplay`
 ```
  --- Logical volume ---
  LV Path                /dev/HotStorage/events
  LV Name                events
  VG Name                HotStorage
  LV UUID                m6YguX-gBs8-fhBc-Iz3z-fYb4-tgx1-QfmtB4
  LV Write Access        read/write
  LV Creation host, time hostname.localdomain.com, 2017-04-06 09:38:52 -0500
  LV Status              available
  # open                 1
  LV Size                5.50 TiB
  Current LE             1441790
  Segments               3
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:5
 ```  


Gathering this information will provide you with all the information you need to being added the disk to the LVM.

Adding the disk to the LVM.
---
In order to add a disk to an LVM you need to Initialize a physical volume to be used by LVM.  This is accomplished with the `pvcreate` command as shown below.  The disk will be the same disk you verified with the `fdisk` command in this case, `/dev/mapper/mpathf` due to the fact that I'm using multipathing 

`pvcreate /dev/mapper/mpathf` 
``` 
Physical volume "/dev/mapper/mpathf" successfully created
```

In order to verify the Physical Volume was created use the `lvmdiskscan` command as shown below.
 `lvmdiskscan -l`
  ```
  WARNING: only considering LVM devices
  /dev/mapper/mpathc          [       1.50 TiB] LVM physical volume
  /dev/mapper/mpathb          [       4.00 TiB] LVM physical volume
  /dev/mapper/mpathf          [       5.00 TiB] LVM physical volume
  3 LVM physical volume whole disks
  1 LVM physical volume
```

You'll notice that the new disk shows up at the bottom of the list.  Now you can add the Physical Volume to a Volume group, this is done with the `vgextend` command. In this case I'll be added the Physical volume to the HotStorage Volume Group.

`vgextend HotStorage /dev/mapper/mpathf`
```
  Volume group "HotStorage" successfully extended
```

Now you can verify this was added successfully by using the `vgs` command
`vgs`
```
  VG             #PV #LV #SN Attr   VSize   VFree
  HotStorage       3   1   0 wz--n-  10.50t 5.00t
 ```
 As you can see we now have 5.0 T of Free Space, but this is currently only associated with the Volume Group.  If you view the Logical Volume this is currently unchanged as shown below.

`lvdisplay`
```
  --- Logical volume ---
  LV Path                /dev/HotStorage/events
  LV Name                events
  VG Name                HotStorage
  LV UUID                m6YguX-gBs8-fhBc-Iz3z-fYb4-tgx1-QfmtB4
  LV Write Access        read/write
  LV Creation host, time hostname.localdomain.com,2017-04-06 09:38:52 -0500
  LV Status              available
  # open                 1
  LV Size                5.50 TiB
  Current LE             1441790
  Segments               3
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:5
 ```
   
in order to add the extended volume group to the Physical volume we'll use the `lvm` command in order to complete this successfully you'll need to know the logical volume path. in this case `/dev/HotStorage/events`  and because we want to use all of the space we'll use the `-l +100%FREE` argument. 


`lvm lvextend -l +100%FREE /dev/HotStorage/events`
```
  Size of logical volume HotStorage/events changed from 5.50 TiB (1441790 extents) to 10.50 TiB (2752509 extents).
  Logical volume events successfully resized.
```
As you can see above, this command extended the logical volume to 10.50 TiB, we can verify this with the `lvdisplay` command 

`lvdisplay`
```
  --- Logical volume ---
  LV Path                /dev/HotStorage/events
  LV Name                events
  VG Name                HotStorage
  LV UUID                m6YguX-gBs8-fhBc-Iz3z-fYb4-tgx1-QfmtB4
  LV Write Access        read/write
  LV Creation host, time hostname.localdomain.com, 2017-04-06 09:38:52 -0500
  LV Status              available
  # open                 1
  LV Size                10.50 TiB
  Current LE             2752509
  Segments               4
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:5
```

Above the LV Size verifies that it was extended successfully. Finally you'll need to extend the partition. 

Here it is important to understand the filesystem.  if you're using ext2, ext3 or ext4 you'll be able to use the `resize2fs` command, however in this case the file system is an xfs.  In order to verify the filesystem you can use the `df` command with the `-T` argument as shown below.

`df -Th`
```
Filesystem           Type   Size  Used Avail Use% Mounted on
/dev/mapper/HotStorage-events
                     xfs    5.5T  5.4T  119G  98% /var
```

Since this is an xfs filesystem you'll need to use the `xfs_growfs` command as shown below.  

Note: using `xfs_growfs` without any additional arguments the filesystem will be increased to the maximum available size. in order to specify a specific size use `-D SIZE`



`xfs_growfs /dev/mapper/HotStorage-events`
```
meta-data=/dev/mapper/HotStorage-events isize=256    agcount=8, agsize=201326336 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=1476392960, imaxpct=5
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal               bsize=4096   blocks=393215, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1476392960 to 2818569216
```

Now running the `df` command we can see the size is 11 Terabytes. 

`df -h`
```
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/HotStorage-events
                       11T  5.4T  5.2T  52% /var/cb/data
```

~Enjoy~

