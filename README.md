# Build Real-Time kernels
This repo outlines the steps needed to build a real-time linux kernel for a Raspberry Pi device.

## 1. Prepare the environment

On your Ubuntu/Debian host computer, install the required build tools and the ARM64 cross-compiler.
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev crossbuild-essential-arm64
```

Next, create a workspace directory:
```bash
mkdir rpi-kernel && cd rpi-kernel
```

## 2. Download the Kernel and RT Patch

You need to match the Raspberry Pi Linux kernel version exactly with the corresponding Real-Time patch version.
To find out which kernel version is running on the Raspberry device, run the following:
```bash
uname -r
```

For this guide, we are using 6.12.y:
```bash
git clone --depth=1 --branch rpi-6.12.y https://github.com/raspberrypi/linux.git
```

Find and download the corresponding RT patch from the [Linux Foundation RT wiki](https://wiki.linuxfoundation.org/realtime/start).
```bash
# Download the patch (replace with the exact matching version found)
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/6.12/patch-6.12.89-rt18.patch.gz
gunzip patch-6.12.89-rt18.patch.gz
```

## 3. Apply the Real-Time Patch

Navigate into the Linux source directory and apply the patch:
```bash
cd linux
patch -p1 < ../patch-6.12.89-rt18.patch
```

If successful, you will see a long list of files being successfully patched without errors.
If not, brace...
Go back to the linux directory and undo the patch:
```bash
git reset --hard HEAD && git clean -fd
```

Checkout the exact point in time that matches the patch.
Then, force it.
The initial clone does not bring everything, so this needs taken care of:
```bash
git fetch --tags
```

Run the following to find the right tag:
```bash
git tag --list
```

Force it with:
```bash
git checkout <tag>
```

The process of finding the right tag/commit may prove difficult.
So, the alternative is to look at the "rejection" file(s) and make the edits manually!

Once everything is in order, proceed with the configuration.

**The rest of the guide focuses on RPi Zero 2 W.**

## 4. Configure the Kernel for Pi Zero 2 W

The Pi Zero 2 W uses a 64-bit capable BCM2710 chip. We need to set up the default 64-bit configuration (`bcmrpi3_defconfig`).
1. Initialize the base configuration:
   ```bash
   ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make bcm2711_defconfig
   ```
2. Enable Full Real-Time Preemption by opening the graphical configuration menu to activate the RT options:
   ```bash
   ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make menuconfig
   ```
   Inside the menu interface, use your arrow keys to navigate to and change these settings:
      - Navigate to General setup -> Preemption Model -> Select Fully Preemptible Kernel (Real-Time).
      - (Optional) Navigate to Kernel features -> Timer frequency -> Change to 1000 Hz (standard for low-latency tasks).
   Select Save, keep the filename .config, and Exit.

## 5. Compile the Kernel

Now, initiate the compile process. Use the -j flag to match the number of CPU cores on the host computer to speed things up (e.g., -j8 for an 8-core machine).
```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc) Image modules dtbs
```

This process can take anywhere from 5 to 30 minutes depending on the computer's speed.
A Ryzen 9 Pro 6950H took about 11 minutes with all 16 cores engaged.

## 6. Install the Files

Once the build finishes, there are two ways to transfer the kernel to the Pi.

### 6.1. By plugging the Raspberry Pi's MicroSD card into the build machine. 
Identify the mount points:
   - Boot partition: (often mounted at `/media/$USER/boot-fs/` or `/media/$USER/firmware/`)
   - Root partition: (often mounted at `/media/$USER/rootfs/`)
 Run the following commands to copy the newly built modules and kernel images over (make sure to replace paths with the actual mount paths):
 ```bash
 # Install Kernel Modules (to the root partition):
 sudo ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make modules_install INSTALL_MOD_PATH=/media/$USER/rootfs/

 # Copy the Kernel Image and Device Tree Blobs (to the boot partition):
 # Copy the main kernel image under our custom name
 sudo cp arch/arm64/boot/Image /media/$USER/firmware/kernel8-rt.img

 # Copy overlays and base hardware DTBs
 sudo cp arch/arm64/boot/dts/broadcom/*.dtb /media/$USER/firmware/
 sudo cp arch/arm64/boot/dts/overlays/*.dtbo /media/$USER/firmware/overlays/
 ```

### 6.2. By copying over SSH.
Open a terminal in the `rpi-kernel/linux` directory and copy the files over to the Pi's temporary directory (`/tmp`). 
Then move them into place on the Pi.
```bash
# Copy the kernel image
scp arch/arm64/boot/Image ${USER}$@raspberrypi.local:/tmp/kernel8-rt.img

# Copy the device tree blobs
scp arch/arm64/boot/dts/broadcom/*.dtb ${USER}@raspberrypi.local:/tmp/
```

Because overlays go into a specific subfolder, it's easiest to compress them first, send them over, and extract them on the Pi.
```bash
# 1. Move directly into the directory where the compiled overlays are
cd arch/arm/boot/dts/

# 2. Create the tarball from right inside this folder
tar -czf ../../../../overlays.tar.gz overlays/

# 3. Jump back to your main linux build root directory
cd ../../../../

# Send the tarball to the Pi
scp overlays.tar.gz ${USER}@raspberrypi.local:/tmp/
```

Kernel modules consist of hundreds of tiny files. Copying them one by one over SSH is incredibly slow. 
Instead, install them into a temporary local folder on the build PC, zip them up, and transfer them as one file.
```bash
# 0. Define vars for convenience
cwd=`(pwd)`
target_dir=/tmp/pi-modules

# 1. Create a temporary folder on your PC
mkdir -p ${target_dir}

# 2. Install modules locally into that folder
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules_install INSTALL_MOD_PATH=${target_dir}

# 3. Compress them
cd ${target_dir}/lib
tar -czf ${cwd}/modules.tar.gz modules/
cd ${cwd}

# 4. Send the tarball to the Pi
scp modules.tar.gz ${USER}@raspberrypi.local:/tmp/

# 5. clean up
rm -rf /tmp/pi-modules
```
--------
Log into the Raspberry Pi via SSH or insert the MicroSD card and power up the device.
Then, proceed with unpacking and moving everything to their final production destinations.
```bash
# 1. Move the kernel image to the boot firmware directory
sudo mv /tmp/kernel8-rt.img /boot/firmware/
   
# 2. Move the base DTB files
sudo cp /tmp/*.dtb /boot/firmware/

# 3. Extract the overlays into the firmware folder
sudo tar -xzf /tmp/overlays.tar.gz -C /boot/firmware/

# 4. Extract the kernel modules into the system modules folder
sudo tar -xzf /tmp/modules.tar.gz -C /lib/

# 5. Clean up the temporary files on the Pi
rm /tmp/*.dtb /tmp/overlays.tar.gz /tmp/modules.tar.gz
```

Now, update `config.txt` and reboot.
```bash
sudo vi /boot/firmware/config.txt
```

Add the line pointing to the new kernel:
```
kernel=kernel8-rt.img
```
Save, exit, and safely trigger the reboot.

After rebooting, confirm the new kernel is active with:
```bash
uname -r

# before
Linux box 6.18.33+rpt-rpi-v8 #1 SMP PREEMPT Debian 1:6.18.33-1+rpt1 (2026-06-01) aarch64 GNU/Linux

#after
Linux box 6.12.92-rt18-v8+ #1 SMP PREEMPT_RT Mon Jun  8 19:02:46 BST 2026 aarch64 GNU/Linux
```

###### Notes: see Raspberry Pi Linux kernel on [Github](https://github.com/raspberrypi/linux).
