# Expand a GCP Disk

## Hypervisor layer

### Google compute

1. Navigate to the Google compute Engine console for the Project that the VM resides in
2. Locate the VM and click into it
   1. NB: at the top of the page under the “VM Instances” header there’s a search / filter box. This can either be a free text filter or you can specify which type of data you would like to search under i.e
      1. “Name” - the of the server
      2. “\<Internal|External> IP Address” - the IP of the server
3. Scroll down to “Storage” and click into the disk you want to edit
4. (Optional) Google recommends snapshoting the disk before editing however it will incur a cost so depends upon requirements - [Notes](https://www.notion.so/Notes-6c395b2349014fa9aeb25295e0d9ec5d?pvs=21)
5. On the “Manage disk” row click on the “EDIT” link
6. Amend the disk size upwards as required.
7. click save

## Virtual machine

### Redhat derivative i.e. Rocky

NB: Redhat’s suggested filesystem at the time of typing is XFS so formating will apply accordingly

#### Non LVM extension

Much of this process has been lifted from the below and is run as root (for lack of typing sudo repeatedly)

[https://www.linuxtechi.com/extend-xfs-root-partition-without-lvm-linux/](https://www.linuxtechi.com/extend-xfs-root-partition-without-lvm-linux/)

* Run fdisk to check that the server has automatically picked up the additional disk space

```bash
fdisk -l
```

*   If the disk change hasn’t automatically been picked up run the below for it to be discovered

    ```bash
    for i in /sys/class/scsi_host/host*/scan; do echo "- - -" > $i; done
    for i in /sys/class/scsi_device/*/device/rescan; do echo "1" > $i; done
    ```

    *   Using “growpart” extend the partition

        ```bash
        growpart /dev/<disk> <partition on disk>
        ```
    * so for “/” whose mount is “/dev/sda2” → “`growpart /dev/sda 2`"
      *   if growpart isn’t available as a package it’s part of cloud-utils and can be installed with

          `yum install cloud-utils-growpart`
*   Next grow the mount to make use to the extra space in the partition

    ```bash
    xfs_growfs < mount point >
    ```

    * So in the case of the above “/” partition “xfs\_growfs / ”
*   check that the space has increased with df

    ```bash
    df -h
    ```

#### LVM extension

Much of this process has been lifted from the below and is run as root (for lack of typing sudo repeatedly) - [https://docs.ukfast.co.uk/operatingsystems/linux/basics/lvm-extend.html](https://docs.ukfast.co.uk/operatingsystems/linux/basics/lvm-extend.html)

*   Run fdisk to check that the server has picked up the extra diskspace

    ```bash
    fdisk -l
    ```

    *   If the disk change hasn’t automatically been picked up run the below for it to be discovered

        ```bash
        for i in /sys/class/scsi_host/host*/scan; do echo "- - -" > $i; done
        for i in /sys/class/scsi_device/*/device/rescan; do echo "1" > $i; done
        ```
*   ~~resize the physical volume to take advantage of the space~~ actually might be possible to use the grow part from above to increase the partition size. Need to test

    ```bash
    pvresize /dev/<disk>
    ```
*   Figure out the VolumeGroup/LogicalVolume that you want to extend

    ```bash
    lvs
    ```
*   grow the LV to take advantage of the space

    ```bash
    lvresize -l +100%FREE /dev/mapper/<VG>/<LV>
    ```
*   confirm with lvs that the space has increased

    ```bash
    lvs
    ```
*   Resize the file system to take advantage of the extra space

    ```bash
    xfs_growfs /dev/mapper/<VG>-<LV>

    or

    resize2fs /dev/mapper/<VG>-<LV>
    ```
*   confirm the disk space has increased

    ```bash
    df -h
    ```
