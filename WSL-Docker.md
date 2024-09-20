## Setting up WSL/Ubuntu for quickdev

1. Ensure `/etc/wsl.conf` has `systemd` enabled
   ```bash
    # Check if /etc/wsl.conf contains [boot]
    if grep -q "\[boot\]" /etc/wsl.conf; then
        # Ensure systemd=true is set
        sudo sed -i '/\[boot\]/!b;n;c\systemd=true' /etc/wsl.conf
        echo "systems is enabled in /etc/wsl.conf. Continue."
    else
        # Append [boot] and systemd=true if not found
        echo -e "\n[boot]\nsystemd=true" | sudo tee -a /etc/wsl.conf
    
        # Print next steps
        echo "Next steps:"
        echo "1. Run: wsl --shutdown"
        echo "2. Run: wsl --distribution Ubuntu"
    fi
    ```
2. Installing docker
    ```bash
      sudo apt update && sudo apt upgrade -y
    
      for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
         
      sudo apt-get install ca-certificates curl
      sudo install -m 0755 -d /etc/apt/keyrings
      sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
      sudo chmod a+r /etc/apt/keyrings/docker.asc
      
      # Add the repository to Apt sources:
      echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
        $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      sudo apt-get update
    
      sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

      sudo docker run hello-world
    ```

3. Bring up an ubuntu container 
    ```bash
       mkdir artifacts
       sudo docker run --cap-add SYS_ADMIN  --device /dev/loop0 --device /dev/loop-control -v $(pwd)/artifacts:/artifacts -it ubuntu:22.04 /bin/bash
    ```

4. Install dependencies
    ```bash
      apt update
      apt upgrade
      ln -fs /usr/share/zoneinfo/America/Vancouver /etc/localtime
      DEBIAN_FRONTEND=noninteractive apt install -y build-essential wget libmpc-dev flex bison bc libncurses-dev file libglib2.0-dev libfdt-dev libpixman-1-dev git zlib1g-dev ninja-build python3-venv e2fsprogs
   ```
5. Setup the toolchain from : https://github.com/riscv-collab/riscv-gnu-toolchain/releases
    ```bash
      cd ~
      mkdir -p /opt
      TOOLCHAIN_VERSION=nightly-2024.09.03-nightly
      OS_VERSION=ubuntu-22.04
      
      # elf toolchain:
      wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2024.09.03/riscv64-elf-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz
      tar xvf riscv64-elf-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz -C /opt
      rm riscv64-elf-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz
      
      # Linux toolchain:
      wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2024.09.03/riscv64-glibc-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz
      tar xvf riscv64-glibc-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz -C /opt
      rm riscv64-glibc-${OS_VERSION}-llvm-${TOOLCHAIN_VERSION}.tar.gz
      
      # Add the toolchain to the PATH
      export PATH=$PATH:/opt/riscv/bin 
    ```

6. Download linux kernel and build it
    ```bash
        cd ~
        KERNEL_VERSION=6.11
        wget https://github.com/torvalds/linux/archive/refs/tags/v${KERNEL_VERSION}.tar.gz
        tar xvf v${KERNEL_VERSION}.tar.gz
        cd linux-${KERNEL_VERSION}
        
        make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
        make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j $(nproc)
    ```

7. Download and cross-compile busybox
    ```bash
        cd ~
        BUSYBOX_VERSION=1.36.1
        wget https://busybox.net/downloads/busybox-${BUSYBOX_VERSION}.tar.bz2
        tar xvf busybox-${BUSYBOX_VERSION}.tar.bz2
        cd busybox-${BUSYBOX_VERSION}
        CROSS_COMPILE=riscv64-unknown-linux-gnu- make defconfig
        CROSS_COMPILE=riscv64-unknown-linux-gnu- make CONFIG_STATIC=y -j $(nproc)

    ```

8. Create a minimal root filesystem
    ```bash
      cd ~ 
      dd if=/dev/zero of=root.bin bs=1M count=4 
      mkfs.ext2 -F root.bin 
      debugfs -w root.bin -R "mkdir /bin" 
      debugfs -w root.bin -R "write busybox-${BUSYBOX_VERSION}/busybox /bin/busybox" 
      debugfs -w root.bin -R "mkdir /sbin" 
      debugfs -w root.bin -R "mkdir /etc" 
      debugfs -w root.bin -R "mkdir /etc/init.d" 
      debugfs -w root.bin -R "mkdir /dev" 
      debugfs -w root.bin -R "mkdir /lib" 
      debugfs -w root.bin -R "mkdir /proc" 
      debugfs -w root.bin -R "mkdir /tmp" 
      debugfs -w root.bin -R "mkdir /usr" 
      debugfs -w root.bin -R "mkdir /usr/bin" 
      debugfs -w root.bin -R "mkdir /usr/lib" 
      debugfs -w root.bin -R "mkdir /usr/sbin" 
      debugfs -w root.bin -R "ln /bin/busybox /sbin/init" 
      debugfs -w root.bin -R "symlink /bin/busybox /bin/sh" 
      echo "#!/bin/busybox sh\n/bin/busybox --install -s \nexit 0" > /tmp/rcS 
      chmod 755 /tmp/rcS
      debugfs -w root.bin -R "write /tmp/rcS /etc/init.d/rcS"
    ```

8. Build and install latest QEMU
    ```bash
        cd ~
        QEMU_VERSION=9.1.0
        wget https://github.com/qemu/qemu/archive/refs/tags/v${QEMU_VERSION}.tar.gz
        tar xvf v${QEMU_VERSION}.tar.gz
        cd qemu-${QEMU_VERSION}
        ./configure --target-list=riscv64-softmmu,riscv64-linux-user
        make -j $(nproc)
        export PATH=$PATH:$(pwd)/build
    ```

9. Boot the kernel and RFS
    ```bash
        cd ~
        qemu-system-riscv64 -nographic -machine virt \
                    -kernel linux-${KERNEL_VERSION}/arch/riscv/boot/Image \
                    -append "root=/dev/vda rw console=ttyS0" \
                    -drive file=root.bin,format=raw,id=drive0     
   ```
