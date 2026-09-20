# How to Create a Bootable Armbian Image for Rockchip RK3318

This guide explains how to prepare and flash an **Armbian image** to a Rockchip RK3318/RK3228 TV Box using **Multitool**.

---

## 🧰 Requirements

- A **MicroSD** (at least 4 GB)
- A **computer** with Windows, Linux, or macOS
- Tools for flashing images, such as:
  - [Balena Etcher](https://etcher.balena.io/)
  - [Rufus](https://rufus.ie/)
  - [Win32 Disk Imager](https://sourceforge.net/projects/win32diskimager/)
- Files to download:
  - `multitool.xz`
  - The **Armbian image** for Rockchip RK3318 (e.g., `Armbian_XX.xx_Rk3318.img.xz`)
  - Find Here [Release Page](https://github.com/txq92/armbian-rk3318/releases)

---

## ⚙️ Steps

### 1. Download the Required Files
Download both:
- `multitool.xz`
- Your preferred **Armbian** image for **Rockchip RK3318**.

---

### 2. Create a Bootable Multitool in MicroSD
1. Open **Balena Etcher**, **Rufus**, or **Disk Imager**.
2. Select the downloaded file `multitool.xz` as the source.
3. Choose your **MicroSD** as the target.
4. Start the flashing process.

<img width="480" alt="Rufus Flash Multitool" src="https://psenyukov.ru/wp-content/uploads/2023/11/h96max2.jpg" />

5. Once the flashing process completes, insert the MicroSD into your TV Box and turn it on once so Multitool automatically expands the partition to use all unallocated SD space.
6. Insert the MicroSD back into your PC. If the **MULTITOOL** drive does not appear, open **Disk Management**, right-click the `MULTITOOL` partition, and assign it a drive letter.

<img width="600" alt="Assign Drive Letter in Disk Management" src="https://psenyukov.ru/wp-content/uploads/2023/11/h96max5.jpg" />

---

### 3. Copy the Armbian Image
1. Open the **MULTITOOL** partition.
2. Inside, open the `images` folder.
3. Copy the downloaded **Armbian image file** (e.g., `Armbian_XX.xx_Rk3318.img.xz`) into the `images` folder.

<img width="600" alt="Copy Armbian image to images folder" src="https://psenyukov.ru/wp-content/uploads/2023/11/h96max6.jpg" />

---

### 4. Safely Eject the MicroSD
After the copy process finishes:
- Eject the MicroSD safely from your computer.

---

### 5. Boot the TV Box & Flash to eMMC
1. Insert the prepared MicroSD into your **RK3318 TV Box**.
2. Power on the TV Box.
3. The **Multitool** menu will appear:
   - *(Optional)* **Backup flash** to back up your stock firmware.
   - **Erase flash** (eMMC).
   - Choose **Burn Image** to flash.
   - Select the Armbian image you copied (e.g. `Armbian_XX.xx_Rk3318.img.xz`) and press **OK**.
   - Wait until the flashing process is complete, then press **OK**.
   - Return to the main menu and select **Shutdown**.
   - Eject the MicroSD card.
   - Power on the TV Box to boot into Armbian from eMMC.
   - Complete initial setup (username, password, etc.).

<img width="432" height="396" alt="Multitool Menu" src="https://github.com/user-attachments/assets/8316e5fc-6fb1-48d7-a9c1-4b768ae00d70" />

---

### 6. Troubleshooting & Post-Install Setup

#### 1. Fix Wi-Fi Connection
If Wi-Fi networks are visible but connection fails, replace the Broadcom firmware configuration file:

1. Remove the default firmware configuration:
   ```bash
   sudo rm -f /usr/lib/firmware/brcm/brcmfmac4334-sdio.rockchip,rk3318-box.txt
   ```
2. Create and edit the file, pasting the contents from `rk3318_fix_wifi.txt`:
   ```bash
   sudo nano /usr/lib/firmware/brcm/brcmfmac4334-sdio.rockchip,rk3318-box.txt
   ```
3. Reboot the TV Box:
   ```bash
   sudo reboot
   ```
4. Connect to Wi-Fi via Armbian config tool:
   ```bash
   sudo armbian-config
   ```
   *(Navigate to **Network** > **WiFi**)*

---

#### 2. Fix Bluetooth (SP2734C / AP6334 Chips)
Many RK3318 TV Boxes use the **SP2734C** Wi-Fi/Bluetooth chip (compatible with **AP6334**). To enable Bluetooth support:

1. Open the Armbian configuration utility:
   ```bash
   sudo armbian-config
   ```
2. Navigate to **System** > **Hardware**.
3. Enable the hardware overlay: `rk3318-box-wlan-ap6334` (press Space to select, then Save).
4. Reboot the TV Box:
   ```bash
   sudo reboot
   ```
5. Check Bluetooth status:
   ```bash
   sudo systemctl status bluetooth
   # or scan devices with:
   bluetoothctl
   ```

---

#### 3. Working LED Control
Manage the blue status LED behavior via `sysfs`:

- **Turn off the blinking LED:**
  ```bash
  echo 0 | sudo tee /sys/devices/platform/gpio-leds/leds/working/brightness
  ```
- **Turn the LED permanently on:**
  ```bash
  echo 1 | sudo tee /sys/devices/platform/gpio-leds/leds/working/brightness
  ```
- **Set LED to blink with system activity (CPU load):**
  ```bash
  echo activity | sudo tee /sys/devices/platform/gpio-leds/leds/working/trigger
  ```

---

## ✅ Done!
You’ve successfully created a **bootable Armbian image** for your **Rockchip RK3318 TV Box** using **Multitool**.  
You can now proceed with installing or testing Armbian on your device.

---

### 🔗 Useful Links
- [Armbian Official Website](https://www.armbian.com/)
- [Armbian Forum (TV Box Section)](https://forum.armbian.com/forum/24-tv-box/)
- [Balena Etcher Download](https://etcher.balena.io/)

---
