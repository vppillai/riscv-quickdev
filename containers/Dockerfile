FROM debian:12.7-slim

ENV TOOLCHAIN_VERSION=nightly-2024.09.03-nightly
ENV OS_VERSION=ubuntu-22.04
ENV KERNEL_VERSION=6.11
ENV BUSYBOX_VERSION=1.36.1
ENV QEMU_VERSION=9.1.0
ENV BUILDROOT_VERSION=2024.02.6

ENV DEBIAN_FRONTEND=noninteractive
ENV PATH=$PATH:/opt/riscv/bin 
ENV PATH=$PATH:/opt/qemu/bin


# install dependencies
RUN apt update && apt upgrade -y && RUN apt upgrade && \
    ln -fs /usr/share/zoneinfo/America/Vancouver /etc/localtime && \
    apt install -y build-essential wget libmpc-dev \
                   flex bison bc libncurses-dev file libglib2.0-dev libfdt-dev \
                   libpixman-1-dev git zlib1g-dev ninja-build python3-venv e2fsprogs

# Setup the elf toolchain
RUN cd ~ && mkdir -p /opt &&\
    wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2024.09.03/riscv64-elf-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz &&\
    tar xvf riscv64-elf-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz -C /opt &&\
    rm riscv64-elf-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz

# Setup the Linux toolchain:
RUN cd ~ && mkdir -p /opt &&\
    wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2024.09.03/riscv64-glibc-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz &&\
    tar xvf riscv64-glibc-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz -C /opt &&\
    rm riscv64-glibc-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz

# Download Linux kernel source and build it
RUN cd ~ && wget https://github.com/torvalds/linux/archive/refs/tags/v${KERNEL_VERSION}.tar.gz &&\
    tar xvf v${KERNEL_VERSION}.tar.gz &&\
    rm v${KERNEL_VERSION}.tar.gz &&\
    cd linux-${KERNEL_VERSION} &&\
    make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig &&\
    make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j $(nproc)

# Download and build busybox
RUN cd ~ && wget https://busybox.net/downloads/busybox-${BUSYBOX_VERSION}.tar.bz2 &&\
    tar xvf busybox-${BUSYBOX_VERSION}.tar.bz2 &&\
    rm busybox-${BUSYBOX_VERSION}.tar.bz2 &&\
    cd busybox-${BUSYBOX_VERSION} &&\
    CROSS_COMPILE=riscv64-unknown-linux-gnu- make defconfig &&\
    CROSS_COMPILE=riscv64-unknown-linux-gnu- make CONFIG_STATIC=y -j $(nproc)

# Download and build QEMU
RUN cd ~  &&\
    wget https://github.com/qemu/qemu/archive/refs/tags/v${QEMU_VERSION}.tar.gz &&\
    tar xvf v${QEMU_VERSION}.tar.gz &&\
    cd qemu-${QEMU_VERSION} &&\
    ./configure --target-list=riscv64-softmmu,riscv64-linux-user --prefix=/opt/qemu &&\
    make -j $(nproc) install &&\
    rm -rf ~/qemu-${QEMU_VERSION} 

# Create a minimal rootfs
RUN cd ~ && \
    dd if=/dev/zero of=root.bin bs=1M count=4 && \
    mkfs.ext2 -F root.bin && \
    debugfs -w root.bin -R "mkdir /bin" && \
    debugfs -w root.bin -R "write busybox-${BUSYBOX_VERSION}/busybox /bin/busybox" && \
    debugfs -w root.bin -R "mkdir /sbin" && \
    debugfs -w root.bin -R "mkdir /etc" && \
    debugfs -w root.bin -R "mkdir /etc/init.d" && \
    debugfs -w root.bin -R "mkdir /dev" && \
    debugfs -w root.bin -R "mkdir /lib" && \
    debugfs -w root.bin -R "mkdir /proc" && \
    debugfs -w root.bin -R "mkdir /tmp" && \
    debugfs -w root.bin -R "mkdir /usr" && \
    debugfs -w root.bin -R "mkdir /usr/bin" && \
    debugfs -w root.bin -R "mkdir /usr/lib" && \
    debugfs -w root.bin -R "mkdir /usr/sbin" && \
    debugfs -w root.bin -R "ln /bin/busybox /sbin/init" && \
    debugfs -w root.bin -R "symlink /bin/busybox /bin/sh" && \
    # Create rcS script content && Add rcS script to the image
    echo "#!/bin/busybox sh\n/bin/busybox --install -s \nexit 0" > /tmp/rcS && \
    chmod 755 /tmp/rcS &&\
    debugfs -w root.bin -R "write /tmp/rcS /etc/init.d/rcS"

# Create a quickstart script
RUN cat <<EOF > ~/quickstart.sh
#!/bin/sh
cd ~
qemu-system-riscv64 -nographic -machine virt \
    -kernel linux-${KERNEL_VERSION}/arch/riscv/boot/Image \
    -append "root=/dev/vda rw console=ttyS0" \
    -drive file=root.bin,format=raw,id=drive0
EOF
RUN chmod 755 ~/quickstart.sh

# download and build buildroot
RUN cd ~ && wget https://buildroot.org/downloads/buildroot-${BUILDROOT_VERSION}.tar.gz &&\
    tar xvf buildroot-${BUILDROOT_VERSION}.tar.gz &&\
    rm buildroot-${BUILDROOT_VERSION}.tar.gz &&\
    cd buildroot-${BUILDROOT_VERSION} &&\
    make qemu_riscv64_virt_defconfig &&\
    make -j $(nproc)
