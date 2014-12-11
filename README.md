# fastmodel-on-kvm
!! Ref
http://www.virtualopensystems.com/en/solutions/guides/kvm-on-arm/
https://wiki.linaro.org/PeterMaydell/KVM/HowTo/KVMHostSetup
 
## The host Linux kernel

$ cp hostConfig .config
$ CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm make menuconfig
$ make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-

## Get the boot wrapper

$ cd ..
$ git clone git://git.linaro.org/arm/models/boot-wrapper.git
$ cd boot-wrapper

Previous versions of the boot-wrapper required you to bake a particular kernel uImage and command line arguments into it, so you would need to rebuild it every time you changed the kernel. That is no longer required, so these instructions document how to build a single linux-system-semi.axf which you can then use for any kernel (zImage or uImage) and command line. You can build this with:

$ make CROSS_COMPILE=arm-linux-gnueabi- linux-system-semi.axf


## Host file system

We will boot our filesystem from a NFS share exported by the machine we use for development. Of course any machine accessible from the local network could be used instead. First we need to make sure NFS is installed, with the help of our distribution's tools. E.g. for Debian and Ubuntu:

$ sudo apt-get install nfs-kernel-server nfs-common

Make sure an appropriate directory is exported in /etc/exports with the right settings. For example:
/srv/nfsroot 192.168.0.0/255.255.0.0(rw,sync,no_root_squash,no_subtree_check,insecure)

$ sudo /etc/init.d/nfs-kernel-server restart

$ wget http://www.virtualopensystems.com/downloads/guides/kvm_on_arm/fs-alip-armel.cramfs

$ sudo mount -o loop -t cramfs fs-alip-armel.cramfs /mnt
$ sudo cp -a /mnt/* /srv/nfsroot/
$ sudo umount /mnt


## Preparing the system to boot a guest

$ wget http://www.virtualopensystems.com/downloads/guides/kvm_on_arm/qemu-system-arm

$ cp qemu-system-arm /srv/nfsroot/bin


## Guest file system and kernel

$ dd if=/dev/zero of=./disk.img bs=1MiB count=512
$ mkfs.ext3 ./disk.img

$ mkdir mnt.cramfs mnt.disk
$ sudo mount -o loop -t cramfs fs-alip-armel.cramfs mnt.cramfs
$ sudo mount -o loop disk.img mnt.disk
$ sudo cp -a mnt.cramfs/* mnt.disk/
$ sudo umount mnt.cramfs mnt.disk
$ sudo cp disk.img /srv/nfsroot/root

## guest kernel

$ cp guestConfig .config
$ CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm make menuconfig
$ make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
$ mv arch/arm/boot/zImage /srv/nfsroot/root/
$ mv guest-a15.dtb /srv/nfsroot/root/


## edit params
$ vi params
motherboard.smsc_91c111.enabled=1
motherboard.hostbridge.userNetworking=1
cluster.cpu0.semihosting-cmd_line="--kernel arch/arm/boot/zImage --dtb host-a15.dtb -- earlyprintk console=ttyAMA0 mem=2048M root=/dev/nfs nfsroot=192.168.0.81:/srv/nfsroot/ rw ip=dhcp"
motherboard.mmc.p_mmc_file="../disk.img"


## Running the model

model_shell cadi_system_Linux-Relase-GCC-4.1.so -f params \
             ~/ARM/boot-wrapper/linux-system-semi.axf




## Running Guest system in host

$ qemu-system-arm \
      -enable-kvm -nographic \
      -kernel zImage -dtb ./guest-a15.dtb \
      -m 512 -M vexpress-a15 -cpu cortex-a15 \
      -drive if=none,file=disk.img,id=fs \
      -device virtio-blk-device,drive=fs \
      -netdev type=user,id=net0 \
      -device virtio-net-device,netdev=net0,mac="52:54:00:12:34:50" \
      -append "console=ttyAMA0 mem=512M root=/dev/vda rw ip=dhcp"


