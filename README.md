mdadm for clearing Synology RAID (E) flag
=========================================

The Problem
-----------
It happened that on a Synology Hybrid RAID, when changing a disk, the disk that 
the new disk was syncing from developed a bad block in the system volume.
It was only a single bad block and caused the system volume to be marked with
the Synology-proprietary "Disk Error" flag (E):

```
sh-4.3# cat /proc/mdstat
Personalities : [linear] [raid0] [raid1] [raid10] [raid6] [raid5] 
[raid4]
md2 : active raid1 sda5[2] sdb5[1]
1948779648 blocks super 1.2 [2/2] [UU]

md1 : active raid1 sda2[0] sdb2[1]
2097088 blocks [2/2] [UU]

md0 : active raid1 sdb1[1](E)
2490176 blocks [2/1] [_E]
```

This created a bad situation: The system was still working fine, but only from
one disk in the RAID (drive 1) which was also developing read errors.
Now as the disk was marked with this error flag, the Synlogy system refused to 
sync it to the other new disk (second drive). As the RAID has been used in 
this condition for another few months (it wasn't mine), the official way - 
reinstalling Synology system from scratch and losing the configuration - 
was not very desirable, given the fact that only one block was bad on the 
system drive and it didn't look like there was any important data on that 
sector. Synology support was unable/unwilling to fix it.

So the bad drive was taken out of the Synology NAS and copied to a new, blank
drive using [ddrescue](https://www.gnu.org/software/ddrescue/):

`ddrescue -f -n /dev/sdSRC /dev/sdDST ~/recovery.log`

Still, the problem remained that the RAID volume was marked with "Disk error"
and thus wouldn't sync to the second drive, even though data was salvaged to 
a new working drive.
This safety measure generally makes sense, as the data integrity cannot be 
ensured on disk read errors, but in this case, the removed second drive 
that was in the second slot prior to chaging the second disk 
couldn't be taken, as the system volume was affected which is changing 
constantly during usage. The data RAID was intact and not affected by the 
bad block and was sync and only one block was affected that presumably 
did not contain important data, the system state changed over the months 
the Synology was in this state, so it seemed reasonable to just remove the E 
flag on the salvaged copy of the drive so that the System volume can sync 
to the second drive.

This is where this fork of mdadm comes into play in order to remove the E
flag from the faulty RAID disk. One would be tempted to just fix this using a
HEX editor, but keep in mind that the Superblock also has a checksum that 
needs to be updated, so it's safer and easier to use a modified copy of mdadm
to do this.

How to use 
----------
It is important to note that at least the DSM version that the affected 
device was using (where the RAID was created with an older version of DSM)
uses the 0.9 format of the SuprtBlock for the system drive, whereas it
uses a 1.2 format for the data drive!
**Be aware that this version currently only implements "curing" Superblock 
0.9 format!**
It would not be hard to implement also fixing Superblock 1 format (remove 
bit 0x8000 from role), but as I didn't have a disk with that format that 
was affected and therefore was unable to test the code, I didn't implement 
it for safety reasons. Feel free to do so, if you are affected!

### Installation
1) Attach the affected disk to a Linux machine, i.e. Ubuntu (you can use 
   a Live-System DVD, if you want)
   Then go your working directory, i.e.: `cd /usr/local/src`

2) Clone this repository either with git or by downloading the master.zip:

   ```
   git clone -b synology https://github.com/leecher1337/mdadm.git
   cd mdadm
   ```

   or 
   
   ```
   wget https://github.com/leecher1337/mdadm/archive/refs/heads/synology.zip
   unzip mdadm-synology.zip
   cd mdadm-synology
   ```

3) Build the special synology build with:
   
   `make mdadm.synology`
   
If everything goes well, you now have a `mdadm.synology` executable in your 
working directory. 

### Fixing the Superblock
**I strongly recommend that you only work on a cloned copy of the original drive
for safety reasons! You should have one anyway, as the original drive is
partly damaged!** See ddrescue command above for an example.

1) First, you have to check if your RAID is a Version 0.9 superblock, 
   otherwise it won't work:
   Replace `/dev/sdb1` with the disk partition you want to fix!
   
   `./mdadm.synology --examine /dev/sdb1 -e 0.9`
   
   If you get an output like this, it works correctly:
```
/dev/sdb1:
          Magic : a92b4efc
        Version : 0.90.00
           UUID : 5b116fc2:267b242b:3017a5a8:c86610be
  Creation Time : Sat Jan  1 00:00:02 2000
     Raid Level : raid1
  Used Dev Size : 2490176 (2.37 GiB 2.55 GB)
     Array Size : 2490176 (2.37 GiB 2.55 GB)
   Raid Devices : 2
  Total Devices : 1
Preferred Minor : 0

    Update Time : Wed Mar  8 16:25:39 2023
          State : clean
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0
       Checksum : c1539a6f - correct
         Events : 12649239


      Number   Major   Minor   RaidDevice State
this     1       8       17        1      active sync diskerror   /dev/sdb1

   0     0       0        0        0      removed
   1     1       8       17        1      active sync diskerror   /dev/sdb1
```

   Depending on the host endianness of your system, you may also 
   have a volume that matches your hsot machine's endianness, so if above
   command fails, you can also try:
   
   `./mdadm.synology --examine /dev/sdb1 -e 0.90`
   
   Remember the -e switch you used to examine and also use it accordingly
   in the command for removing the error flag in the next step!
   The example here is with swapped endianness, so I'm using `-e 0.9`.
   
   Check, if you see the `diskerror` marking in the output. If not, 
   there is no need to fix the flag or you are looking at the wrong volume,
   do not go ahead!
   
   If neither 0.9, nor 0.90 format works for you, you may not have a 
   supported superblock and you should not go ahead! You can open an issue
   here and we can walk through the analysis of your volume in order to 
   implement a working version for Superblock version 1.
   
2) Now remove the diskerror flag with:

   `./mdadm.synology --misc --no-diskerr /dev/sdb1 -e 0.9`
   
   Don't forget to adjust the -e paramter according to 1)
   If there is no output, it should have worked.
   
3) Verify that the `diskerror` flag is gone by issuing the same command 
   as in step 1):
   
```
/dev/sdb1:
          Magic : a92b4efc
        Version : 0.90.00
           UUID : 5b116fc2:267b242b:3017a5a8:c86610be
  Creation Time : Sat Jan  1 00:00:02 2000
     Raid Level : raid1
  Used Dev Size : 2490176 (2.37 GiB 2.55 GB)
     Array Size : 2490176 (2.37 GiB 2.55 GB)
   Raid Devices : 2
  Total Devices : 1
Preferred Minor : 0

    Update Time : Wed Mar  8 16:25:39 2023
          State : clean
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
  Spare Devices : 0
       Checksum : c15399ef - correct
         Events : 12649239


      Number   Major   Minor   RaidDevice State
this     1       8       17        1      active sync   /dev/sdb1

   0     0       0        0        0      removed
   1     1       8       17        1      active sync   /dev/sdb1
```

Now you can put the drive back into the Synology and cross fingers that it
boots up. If it doesn't work and you lose all your data, don't blame me,
no warranty whatsoever!

## Contact
Author: leecher@dose.0wnz.at

Use the github issue tracker on https://github.com/leecher1337/mdadm
