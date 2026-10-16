# Step 1: 安装依赖
sudo apt update
sudo apt install -y qemu-system-arm gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi gdb-multiarch gcc make flex bison libncurses-dev libssl-dev bc cpio bzip2 wget rsync

# Step 2: 创建目录
mkdir -p ~/arm-dev/{kernel,busybox,images}
cd ~/arm-dev

# Step 3: 编译内核
cd ~/arm-dev/kernel
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.9.tar.xz
tar -xvf linux-7.0.9.tar.xz
cd linux-7.0.9
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- versatile_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig

    % 启用调试信息
    % 在 menuconfig 界面中，按以下路径开启调试选项：
    <1> 进入 Kernel hacking 菜单。
    <2> 在 Kernel hacking 菜单的顶部附近，找到并选中 [ ] Kernel debugging 选项。按 SPACEBAR 键将它设置为 [*]。
    <3> 进入 Compile-time checks and compiler options 子菜单。
    <4> 进入 Debug information (Rely on the toolchain's implicit default DWARF version) 子菜单。
    <5> 找到并选中 ( ) Rely on the toolchain's implicit default DWARF version，按 SPACEBAR 键将它设置为 (X)。这会生成完整的 DWARF 调试信息。

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j$(nproc) zImage modules dtbs

# Step 4: 编译 BusyBox
cd ~/arm-dev/busybox
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar -xjf busybox-1.36.1.tar.bz2
cd busybox-1.36.1
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config  // 开启静态编译
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig

    # 开启静态编译（让 busybox 不依赖共享库，方便在 QEMU 中运行）
    在 menuconfig 界面中：
    1）进入 Settings → Build static binary (no shared libs)，按空格选中
    2）多次按 Exit 退出并保存

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j$(nproc)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
file _install/bin/busybox

# Step 5: 制作 initrd
cd ~/arm-dev/busybox/busybox-1.36.1/_install
mkdir -p proc sys dev etc/init.d tmp

cat > init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t debugfs nodev /sys/kernel/debug
mount -t tracefs nodev /sys/kernel/debug/tracing
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

# Step 6: 启动 QEMU（退出 QEMU：先按 [Ctrl + A] ，松开后再按 [X] ）
qemu-system-arm -M versatilepb \
    -kernel ~/arm-dev/kernel/linux-7.0.9/arch/arm/boot/zImage \
    -dtb ~/arm-dev/kernel/linux-7.0.9/arch/arm/boot/dts/arm/versatile-pb.dtb \
    -initrd ~/arm-dev/busybox/busybox-1.36.1/initrd.img \
    -append "root=/dev/ram0 console=ttyAMA0,115200 nokaslr" \
    -nographic -s

    /*
        -s：在TCP 1234端口开启GDB调试服务
        -S：让CPU在启动时暂停，等待GDB连接
        nokaslr：关闭内核地址空间随机化，确保断点稳定
    */