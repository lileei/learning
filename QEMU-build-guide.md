# Step 1: 安装依赖
sudo apt update
sudo apt install -y qemu-system-arm arm-linux-gnueabi-gcc arm-linux-gnueabi-binutils gcc make flex bison libncurses-dev libssl-dev bc cpio bzip2 wget rsync

# Step 2: 创建目录
mkdir -p ~/arm-dev/{kernel,busybox,images}
cd ~/arm-dev

# Step 3: 编译内核
cd ~/arm-dev/kernel
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.44.tar.xz
tar -xvf linux-6.1.44.tar.xz
cd linux-6.1.44
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- versatile_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j$(nproc) zImage dtbs

# Step 4: 编译 BusyBox
cd ~/arm-dev/busybox
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- oldconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j$(nproc)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install

# Step 5: 制作 initrd
cd ~/arm-dev/busybox/busybox-1.36.1/_install
mkdir -p proc sys dev etc/init.d
cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/bin/sh
EOF
chmod +x init
cat > etc/inittab << 'EOF'
::sysinit:/etc/init.d/rcS
::askfirst:/bin/sh
EOF
touch etc/init.d/rcS
chmod +x etc/init.d/rcS
cd dev
sudo mknod console c 5 1
sudo mknod null c 1 3
sudo mknod ttyS0 c 4 64
cd ..
find . -print0 | cpio --null -o -H newc --owner=root:root | gzip -9 > ../initrd.img

# Step 6: 启动 QEMU
qemu-system-arm -M versatilepb \
    -kernel ~/arm-dev/kernel/linux-6.1.44/arch/arm/boot/zImage \
    -dtb ~/arm-dev/kernel/linux-6.1.44/arch/arm/boot/dts/versatile-pb.dtb \
    -initrd ~/arm-dev/busybox/busybox-1.36.1/initrd.img \
    -append "root=/dev/ram0 console=ttyAMA0,115200" \
    -audiodev none,id=noaudio \
    -nographic