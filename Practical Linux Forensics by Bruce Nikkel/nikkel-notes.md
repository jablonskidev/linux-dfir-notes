# Study Notes: Practical Linux Forensics by Bruce Nikkel

Here are my study notes as I work through *Practical Linux Forensics* by Bruce Nikkel.

**Table of contents:**
- [Evidence From Storage Devices and File Systems](#Evidence-From-Storage-Devices-and-File-Systems)
- [Analysis of Storage Layout and Volume Management](#Analysis-of-Storage-Layout-and-Volume-Management)
- [Analysis of Partition Tables](#Analysis-of-Partition-Tables)
- [Logical Volume Manager](#Logical-Volume-Manager)
- [Linux Software RAID](#Linux-Software-RAID)

## Evidence From Storage Devices and File Systems

This chapter focuses on the forensic analysis of Linux storage, including partition tables, volume management and RAID, filesystems, swap partitions and hibernation, and drive encryption.
Each of these areas have Linux-specific artifacts we can analyze.

When performing a forensic analysis of a computer system's storage, the first step is to identify what is on the drive.
You must understand the layout, formats, versions, and configuration.

### Analysis of Storage Layout and Volume Management

This section explains Linux partitions and volumes on storage media.
You'll need to reconstruct or reassemble volumes that may contain filesystems and highlight traces of information interesting to an investigation.

#### Analysis of Partition Tables

Typical storage media are organized using a defined partition scheme.
Partitions are defined with a partition table, which provides information like the partition type, size, off set, etc.
Linux file systems are often divided into partitions to create separate filesystems.
We want to identify the partition scheme, analyze the partition tables, and look for possible inter-partition gaps.

During a forensic examintation, DOS or GPT partition types may indicate the contents, but users can define any partition type they want and then create a completely different filesystem.
The partition type is used as an indicatior for various tools, but there is no guarantee that it will be correct.
If a partition type is incorrect or misleading, it could be an attempt to hide or obfuscate information, similar to trying ot hide a file type by changing the extension.

On a Linux system, detected partitions appear in the `/dev/` directory.
This is a mounted pseudo-directory on a running system.
In a postmortem forensics examination, this directory will be emtpty, but the device names may still be found in the logs, referenced in configuration files, or found elsewhere in files on the filesystem.

If a Linux system detects partitions on a particular drive, then additional device files are created to represent those partitions.
The naming convention usually adds a number to the drive or the letter `p` with a number.

**Note:** Be careful with units. Some tools use sectors while others use bytes.

#### Logical Volume Manager

Modern operating systems provide volume management for organizing and managing groups of physical drive, allowing the flexibility to create logical (virtual) drives that can contain partitions and filesysystem.
Volume management can be a separate subsystem like Logical Volume Manage (LVM), or it can be built directly in the filesystem as in btrfs or zfs.
The most common volume manager in Linux environment is LVM.

Key concepts for LVM systems:
- **Physical volume (PV):** Physical storage device (SATA, SAS, and NVMe drives)
- **Volume grouping (VG):** Created from a group of PVs
- **Logical volume (LV):** Virtual storage device within a VG
- **Physical extents (PEs):** Sequence of consecutive sectors in a PV
- **Logical extents (LEs):** Sequence on consecutive sectors in an LV

In the context of LVM, extents are similar to traditional filesystem blocks, and they have a fixed size defined at creation.
A typical default LVM extentsixe is 8192 sectors (4 MB) and is used for both PEs and LEs.
LVM is also able to provide redundancy and stripping for logical volumes.

The use of partition tables is not required for LVM, and PVs can be created on the raw disk without partition.
When partitions are used, LVM has a partition entry type indicating that the physical drive is a PV.

LVM tools operate on devices, not plain files.
To examine an LVM setup on a Linux forensic analysis workstation, the suspect drive must be attachaed with a write blocker or as a read-only acquired image file associated with a loop device.

LVM also has the ability to perform copy-on-write (CoW) snapshots.
There can be useful for forensics because snapshot of volumes there may be snapshots of volumes from a previous point in time.
On running systems, the volumes can be frozen in a snapshot for analysis of acquisition.

#### Linux Software RAID

**RAID:** Redundant array of independent disks

RAID configurations:
- **Mirror** refers to two disk that are mirror images of each other
- **Striped** refers to stripes of data spread across multiple disks for performance. Multiple disks can be read from and written to simultaneously.
- **Parity** is a computer science term for an extra bit of data used to error detection and/or correction.

A RAID has different levels that describe how a group of disks work together:
- **RAID** Striped for performance, not redundancy
- **RAID1** Mirrored disks for reundancy: half the capacity, but up to half of the disks can fail.
- **RAID2,3,4,5** Variations of parity allowing a single disk to fail
- **RAID6** Dougle parity allowing up to two disks to fail
- **RAID10** Mirrored and striped ("1 + 0") for maximum redundancy and performance
- **JBOD** "Just a Bunch of Disks" concatenated. No redundancy of performance. Maximum capacity.

Organizations choose a RAID level based on a balance of cost, performance, and reliability.

The most commonly used method of RAID is the Linux software RAID or `md`.
This kernel module produces a metadevice from the configured array of disks.
You can use the `mdadm` userspace tool to configure and manage the RAID.

Each device from a Linux RAID system has a superblock (not ot be confused with filesystem superblocks) that contains information about the device and the array.
The default location of the `md` superblock on a modern Linux RAID device is eight sectors from the start of the partition.

You can examine superblock information with a hex editor or the `mdadm` command. The outputcontains several artifacts that may be interest in a forensic examination:
- **`Array UUID`** will identify the overall RAID system. Each disk belonging to this the RAID (including previously replaced disks) will have this same UUID string in its superblock.
- **`Name`** can be specified by the admin or autogenerated.
- **`Device UUID`** uniquely identifies the individual disks.
- **`Creation time`** refers to the creation date of the array. A newly replaced disk will inherit the original array's creation date.
- **`Update time`** refers to the last time the superblock updated due to some filesystem event.

The disks in an array might not be identical sizes.
For a forensics examination, this can be important.

The kernel should automatically scan and recognize Linux RAID devices on boot.
However, they can also be defined in separate configutation files.
During an examination involving RAID systems, check for uncommented `DEVICE` and `ARRAY` lines in the `/etc/mdadm.conf` file or files in `/etc/mdadm.conf/d`.

If previously failed disks can be physically located, they may still be readable.
Failed or replaced disks contain a snapshot of data at a certain point in time and may be relevant to a forensic investigation.
