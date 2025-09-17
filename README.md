# jetson-orin-nano-ch34x-usb-driver
This guide explains how to build and install the in-tree CH341 USB serial driver on Jetson boards running JetPack/L4T so that devices such as CH340/CH341 USB-UART converters are recognized as `/dev/ttyUSBx`.

## ⚠️IMPORTANT: Remove `brltty` first (avoids ttyUSB getting grabbed)**
On some Ubuntu-based Jetson images, **brltty** (Braille display service) may automatically claim CH340/CH341 USB–UART adapters as Braille devices and immediately disconnect `/dev/ttyUSB0`. If your device keeps disappearing or kernel logs show `brltty sets config #1`, remove brltty first:

```bash
sudo apt purge -y 'brltty*'
sudo apt autoremove -y
```

*Explanation:* `brltty` can bind to CH34x adapters and steal the interface,
causing `/dev/ttyUSB0` to vanish right after it appears. Removing it avoids
conflicts before building/installing the driver.

## 📋 Prerequisites
1. Check your current kernel and L4T version

    ```bash
    uname -r                    # e.g., 5.15.148-tegra
    cat /etc/nv_tegra_release   # e.g., R36.3
    ```

2. Install required packages

    ```bash
    sudo apt update
    sudo apt install build-essential bc libncurses5-dev libncursesw5-dev libssl-dev
    ```

3. Download the matching Kernel Source

    - Go to [NVIDIA Jetson Linux Downloads](https://developer.nvidia.com/embedded/jetson-linux)
    - Download `Driver Package (BSP) Sources` for your exact L4T/JetPack version.
    - This download will give you a file named `public_sources.tbz2`.

## 📦 Prepare Kernel Source
1. Create a workspace folder

    ```bash
    mkdir -p ~/jetson_kernel
    ```

2. Place `public_sources.tbz2` inside `~/jetson_kernel`

    Move or copy the downloaded file into this folder for easier organization.

3. Extract the BSP sources and kernel source

    ```bash
    cd ~/jetson_kernel
    tar -xf public_sources.tbz2
    cd Linux_for_Tegra/source
    tar -xf kernel_src.tbz2
    ```

4. Enter the kernel source directory

    ```bash
    cd kernel/kernel-jammy-src   # Folder name may vary slightly by version
    ```

## ⚙ Configure the Kernel
1. Import your current kernel configuration

    ```bash
    zcat /proc/config.gz > .config
    make olddefconfig
    ```

2. Enable CH341 support using menuconfig

    ```bash
    make menuconfig
    ```

    - Navigate:
        `Device Drivers → USB support → USB Serial Converter support`

    - Mark as modules (M):
        - **USB Winchiphead CH341 Single Port Serial Driver**

    - Save and exit.

## 🏗 Build the Driver Modules

```bash
make modules_prepare
make M=drivers/usb/serial clean
make M=drivers/usb/serial modules -j"$(nproc)"
```

Verify the compiled modules:

```bash
find drivers/usb/serial -maxdepth 1 -name "*.ko"
# Should show usbserial.ko and ch341.ko
```

## 📥 Install and Load
1. Copy the modules to the system

    ```bash
    sudo cp drivers/usb/serial/usbserial.ko /lib/modules/$(uname -r)/kernel/drivers/usb/serial/
    sudo cp drivers/usb/serial/ch341.ko     /lib/modules/$(uname -r)/kernel/drivers/usb/serial/
    sudo depmod -a
    ```

2. Load the modules

    ```bash
    sudo modprobe usbserial
    sudo modprobe ch341
    ```

3. Plug in the CH340/CH341 device and verify

    ```bash
    sudo dmesg | egrep -i "ch34|ttyUSB|usbserial"
    ls -l /dev/ttyUSB*
    ```
    You should see `/dev/ttyUSB*`.

## 🔁 Auto-Load on Boot (Optional)

```bash
echo -e "usbserial\nch341" | sudo tee /etc/modules-load.d/ch341.conf
```

## 👤 User Permissions
Allow your user to access serial ports without root:

```bash
sudo usermod -aG dialout $USER
# Log out and log back in for the change to take effect
```

## ✅ Result
After completing these steps, your Jetson board will correctly recognize CH340/CH341 USB-Serial adapters as `/dev/ttyUSBx`, ready for use with Arduino, ESP32, STM32, and other serial devices.