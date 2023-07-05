## This instruction assumes that all actions are performed on Windows OS
# 1.Creating VHD
First you need to create an empty VHD image with a fixed size. If you need to minimize the size of the image, then to experiment with the CLI, it is enough to create a disk with a volume of ~ 1.5 GB. For a working system with a GUI, you can limit the volume to 10 GB (with the condition of storing user data outside the VHD).

Let's create a VHD using diskpart.exe on Windows:

create vdisk file=<path_to_vhd>\debian.vhd type=fixed maximum=1500

# 2. Installing base of Debian
Next, you need to install Debian on the VHD. I used VirtualBox 6.1 for this, installed debian 12 . The parameters of the virtual machine are by default, there is no need to create a new disk, it is enough to connect the previously created debian.vhd.

Debian installation is standard, I will pay attention only to some points.

When partitioning the disk, I created one ext4 boot partition. I did not do a swap partition on the VHD, after installation, you can place a file or swap partition in a convenient place.

When choosing additional software for installation, I left only the SSH server and standard system utilities. Everything else can be put later, if necessary. GRUB is installed in MBR.


# 3. Preparing Linux to boot from VHD

It is necessary to add NTFS support and the partprobe utility to the installed system, which allows you to inform the OS kernel about the need to re-read the hard disk partition table.

``` bash
sudo apt-get install ntfs-3g parted
```

Then you need to prepare scripts for initramfs.

/etc/initramfs-tools/hooks/ — here are the scripts that are run when generating an initramfs image. Here we will place a script to add the partprobe utility with the necessary libraries to initramfs.

/etc/initramfs-tools/scripts/local-top/ — after executing these scripts, the loader believes that the root device is mounted. I.e. there will be a script for mounting the VHD.

You need to create a "partcopy" script with following content and make it executable:
``` bash
#!/bin/sh
cp -p /sbin/partprobe "$DESTDIR/bin/partprobe"
cp -p /lib/x86_64-linux-gnu/libparted.so.2 "$DESTDIR/lib/x86_64-linux-gnu/libparted.so.2"
cp -p /lib/x86_64-linux-gnu/libreadline.so.7 "$DESTDIR/lib/x86_64-linux-gnu/libreadline.so.7"
cp -p /lib/x86_64-linux-gnu/libtinfo.so.6 "$DESTDIR/lib/x86_64-linux-gnu/libtinfo.so.6"
```
The script for mounting a VHD with following content must be named "loop_boot_vhd" and marked as executable:
```bash
#!/bin/sh

PREREQ=""
prereqs()
{
	echo "$PREREQ"
}
case $1 in
# get pre-requisites
prereqs)
	prereqs
	exit 0
	;;
esac

cmdline=$(cat /proc/cmdline)
#cmdline="root=/dev/loop0p1 loop_file_path=/debtst.vhd loop_dev_path=/dev/sda2"

for x in $cmdline; do
	value=${x#*=}
	case ${x} in
		root=*) value_loop_check=${value#*loop}
			if [ x$value_loop_check != x$value ]
				then loop_dev=${value%p*}
				loop_part_num=${value##*p}
				else echo "Root device is not a loop device. loopboot hook will be terminated."; return
			fi ;;
		loop_file_path=*) loop_file_path=$value ;;
		loop_dev_path=*) loop_dev_path=$value ;;
	esac
done

if [ -z $loop_file_path ] || [ -z $loop_dev_path ]
	then echo "Either loop_file_path or loop_dev_path parameter does not specified. loopboot hook will be terminated."; return
fi
#modprobe fuse
modprobe loop max_part=64 max_loop=8
loop_dev_type=$(blkid -s TYPE -o value $loop_dev_path)
if [ ! -d /host ]; then
	mkdir /host
fi
if [ "$loop_dev_type" != "ntfs" ]
	then mount -t $loop_dev_type $loop_dev_path /host; echo "mount -t $loop_dev_type $loop_dev_path /host"; echo "mounted using mount";
	else ntfs-3g $loop_dev_path /host; echo "mounted using ntfs-3g";
fi
losetup $loop_dev /host$loop_file_path
partprobe $loop_dev
```

# 4. Update initramfs
After placing the scripts in the necessary directories (/etc/initramfs-tools/scripts/local-top/loop_boot_vhd and /etc/initramfs-tools/hooks/partcopy), you need to rebuild initramfs with the command:
```bash
sudo update-initramfs -c -k all
```
#### For further configuration, you need to remember the kernel version number: /boot/initrd.img-*version*-amd64 and /boot/vmlinuz-*version*-amd64.
At this point, the image is ready to run on real hardware, you can turn off the virtual machine and start preparing the loader. The finished image "debian.vhd" must be copied to the root of the C: drive, further scripts are written based on the assumption that the VHD is located in the root of the NTFS partition.

# 5. Configuring grub4dos

First you need to download the latest version of grub4dos. Then create menu.lst file and fill it with following content:
```grub
title debian
find --set-root --ignore-floppies --ignore-cd /debian.vhd
map /debian.vhd (hd3)
map --hook
root (hd3,0)
kernel /boot/vmlinuz-*version*-amd64 root=/dev/loop0p1 rw loop_file_path=/debian.vhd loop_dev_path=/dev/sda2
initrd /boot/initrd.img-*version*-amd64
```
# 6. Configuring the bootmgr

Please note: depending on the Windows version and OS installation features, minor differences may occur.

The first thing to do is to connect the hidden partition with bootmgr, in the example below I connect the hidden partition "System Reserved" to the C directory:\mnt (the directory must be pre-created). 
Commands are executed in diskpart.exe:
```
list volume
  Volume 1         System Rese  NTFS   Partition    549 MB  Healthy    System
  Volume 2     C                NTFS   Partition     49 GB  Healthy    Boot

select volume 1

assign mount=C:\mnt
```

After that, you need to unpack it into a directory C:\mnt \ files from the archive with grub4dos: grldr and grldr.mbr. 
To the same directory, copy the menu.lst file created in the previous step. 
After that, the section can be disabled in diskpart.exe:
```
remove mount=C:\mnt
```
To configure the display of the menu item when Windows boots, you need to do the following:

bcdedit.exe -create -d grub4dos -application bootsector
*The entry {GUID} was successfully created.

In response, the GUID of the new menu item will be reported. The resulting GUID is used in the following commands:

bcdedit.exe -set {GUID} device boot
bcdedit.exe -set {GUID} path \grldr.mbr
bcdedit.exe -displayorder {GUID} -addlast

## Now you can reboot pc and boot into linux
