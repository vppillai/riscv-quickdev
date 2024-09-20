# Enabling and mounting USB on WSL

I plan to use an `iCEstick Evaluation Kit` and a RP2350 with RISC-V cores for some of the RISC-V experiments. While we can switch between Windows and WSL to copy files from wSL into the device, I plan to stick to WSL exclusively. But, the default WSL kernel (`5.10.60.1`) does not support some of the required drivers. So the steps below are to compile a cusotm kernel and boot WSL with it. We then use `USBIPD` to  for sharing locally connected USB devices to WSL. 


## OPTIONAL: Backup the current WSL instance (in case something goes wrong)

Since we are about to pull in custom kernel releases, it is a good idea to backup the current WSL instance in case you have important data in it.

1. Make a backup of the current WSL instance
   ```bash
   wsl --export <Distro> <BackupLocation>
   ```
2. Get current distro names and install location
   ```powershell
   Get-ChildItem "HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss" | ForEach-Object {
      $id = (Get-ItemProperty $_.PSPath).DistributionName
      $path = (Get-ItemProperty $_.PSPath).BasePath
      [PSCustomObject]@{
         Distribution = $id
         Path = $path
      }
   }
   ```
2. Import the backup if needed
   ```powershell
   wsl --import <Distro> <InstallLocation> <BackupLocation> --version 2
   ```


## Booting with a custom Kernel

1. Download the WSL Kernel, unpack it and switch to the latest release version .
   ```bash
   sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses-dev autoconf libudev-dev libtool
   git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
   cd WSL2-Linux-Kernel
   git checkout linux-msft-wsl-6.6.36.6
   ```
2. Apply the current configuration
   ```bash
   cp /proc/config.gz config.gz
   gunzip config.gz
   mv config .config
   ```
3. Ensure that the following configurations are enabled under `Device Drivers -> USB Support` (`CONFIG_USB_SUPPORT`)
   ```bash
   USB announce new devices (`CONFIG_USB_ANNOUNCE_NEW_DEVICES:`)
   USB Modem (CDC ACM) support (`CONFIG_USB_ACM`)
   USB/IP (`CONFIG_USBIP_CORE`)
   USB/IP -> VHCI HCD (`CONFIG_USBIP_VHCI_HCD`)
   USB Serial Converter Support (`CONFIG_USB_SERIAL`)
   USB Serial Converter Support -> USB CP210x family of UART Bridge Controllers (`CONFIG_USB_SERIAL_CP210X`)
   USB Serial Converter Support ->  USB FTDI Single Port Serial Driver (`CONFIG_USB_SERIAL_FTDI_SIO`)
   USB Mass Storage support (`USB_STORAGE`)
   USB Mass Storage support -> Disable USB Mass Storage verbose debug (`USB_STORAGE_DEBUG` [=n])
   USB Mass Storage support -> USB Attached SCSI (USB_UAS [=y])
   ```
4. Build the kernel
   ```bash
   sudo make -j $(nproc) && sudo make modules_install -j $(nproc) && sudo make install -j $(nproc)
   ```

   **Note**: The install step is for competness. WSL relies on `.wslconfig` at `%USERPROFILE%` for kernel selection. We will update this in the next step.
   
6. Copy the new kernel to windows and edit `%USERPROFILE%\`
   ```bash
   WIN_USERNAME=$(cmd.exe /c "echo %USERNAME%" | tr -d '\r')
   WSL_CONFIG_PATH="/mnt/c/Users/$WIN_USERNAME/.wslconfig"
   KERNEL_PATH="C:\\\\\\\\Users\\\\\\\\$WIN_USERNAME\\\\\\\\linux-msft-wsl-6.6.36.6"
   KERNEL_SETTING="kernel=${KERNEL_PATH}"
   cp arch/x86/boot/bzImage $KERNEL_PATH
   # Check if the .wslconfig file exists
   if [ -f "$WSL_CONFIG_PATH" ]; then
      # Check if the kernel setting is already in the file
      if grep -q "^kernel=" "$WSL_CONFIG_PATH"; then
         echo "Kernel setting already exists in .wslconfig. Please update it manually to $KERNEL_PATH"
      else
         echo "Updating .wslconfig with the kernel setting."
         echo "$KERNEL_SETTING" >> "$WSL_CONFIG_PATH"
      fi
   else
      # Create the .wslconfig file and add the kernel setting
      echo "Creating .wslconfig and adding the kernel setting."
      echo -e "[wsl2]\n$KERNEL_SETTING" > "$WSL_CONFIG_PATH"
   fi

   echo "Restart WSL with 'wsl --shutdown' and reopen your WSL terminal to apply the new kernel."
   ```

7. Restart WSL from powershell in windows and check the kernel version
   ```powershell
   wsl --shutdown
   wsl uname -r
   ```
   The output should be `6.6.36.6-microsoft-standard-WSL2+`

## Mounting USB devices

1. Install USBIPD-WIN from https://github.com/dorssel/usbipd-win/releases
   ```powershell
   winget install usbipd
   ```
2. List and bind the USB devices from an admin powershell
   ```powershell
   usbipd list
   usbipd bind --busid <busid>
   ```

   **Note**: busid looks like `5-1` or `5-1.1` etc from the list command.

3. Bind the required USB device in the admin powershell
   ```powershell
   usbipd bind --busid <busid> --force
   ```
4. Check if the device is bound
   ```powershell
   usbipd list
   ```
   The output should show the device as `shared` under the `STATE` column.

5. From a non-admin powershell, attach the USB device to WSL
   ```powershell
   usbipd attach --wsl --busid <busid>
   ```
   **Note**: Make sure that you hav the WSL terminal open before running this command to make sure the device is attached to the correct WSL instance.

6. Check for the device in wsl 
   ```bash
   lsusb
   ```
   If the device is a USB storage device like the boot disk of a RP2350, you should see the device listed with `lsblk` as well.
7. Mount the device
   ```bash
   sudo mkdir /mnt/rpi
   sudo mount /dev/sdXN /mnt # Replace X and N with the correct device and partition number. e.g. /dev/sdd1
   ls /mnt/rpi
   cat /mnt/rpi/INFO_UF2.TXT
   ```
   **Note**: Replace `sdX` with the device name from `lsblk` output.
