# How to Update Acer Laptop BIOS from Linux — Ventoy + HBCD PE Method

## Problem
Acer distributes BIOS updates as Windows self-extracting executables (SFX). These can't be run on Linux, and the embedded BIOS image is encrypted with Insyde's proprietary algorithm — you cannot extract it on Linux.

## Solution (Updated 2026-08-09)
Use **Ventoy** with **Hiren's BootCD PE (Windows PE)** to run the BIOS updater. This works with UEFI firmware, unlike FreeDOS which has known issues with UEFI boot on modern Acer laptops.

---

## Step 0: Identify your exact model number

The name on the front of your laptop (e.g. "Acer Aspire 14 AI") is NOT the model number. You need the exact model for the Acer support site.

### Method 1: fwupd (recommended)
```bash
sudo fwupdmgr get-updates
```
This shows your device name (e.g. "Acer Aspire A14-52MT") and whether LVFS has a BIOS update available. If no BIOS update appears, proceed to Step 1.

### Method 2: dmidecode (no root required for basic info)
```bash
sudo dmidecode -s system-product-name
sudo dmidecode -s system-version
sudo dmidecode -s system-serial-number
```

### Method 3: /sys/class/dmi
```bash
cat /sys/class/dmi/id/product_name
cat /sys/class/dmi/id/product_version
cat /sys/class/dmi/id/sys_vendor
```

### Method 4: lshw (if installed)
```bash
sudo lshw -class system | grep "product"
```

Install required packages:
```bash
sudo apt install fwupd fwupd-doc fwupd-tests p7zip-full dmidecode lshw
```

---

## Step 1: Download and extract the BIOS update from Acer

Go to your Acer support page (e.g. `https://www.acer.com/us-en/support/product-support/<YOUR_MODEL>`), find your model, and download the latest BIOS ZIP.

```bash
cd ~/Downloads/ACER
7z x BIOS_Acer_1.32_A_A.zip -oacer_bios_update
```

This gives you `ZPN_V1_32.exe` — a 7-Zip SFX stub (PE32+ Windows executable).

**Key insight:** Running `7z x ZPN_V1_32.exe` on Linux extracts the PE sections (`.text`, `.rdata`, etc.) and a `[0]` file (9.8MB), but the `[0]` file is encrypted/compressed with Insyde's proprietary algorithm. You CANNOT extract the raw BIOS image on Linux. The SFX is designed to run on Windows and flash the BIOS there.

---

## Step 2: Get Ventoy and Hiren's BootCD PE

### Download Ventoy (latest version)
```bash
cd /tmp
curl -L -o ventoy-1.1.17-linux.tar.gz "https://github.com/ventoy/Ventoy/releases/download/v1.1.17/ventoy-1.1.17-linux.tar.gz"
tar xzf ventoy-1.1.17-linux.tar.gz
```

### Download Hiren's BootCD PE (~3.06 GB)
```bash
curl -L --insecure -o HBCD_PE_x64.iso "https://download.hirensbootcd.org/files/PE/HBCD_PE_x64.iso"
# SHA-256: 8c4c670c9c84d6c4b5a9c32e0aa5a55d8c23de851d259207d54679ea774c2498
```

**Why HBCD PE instead of FreeDOS?**
- FreeDOS does NOT work correctly with UEFI BIOS on modern Acer laptops
- HBCD PE is a full Windows PE environment that boots natively in UEFI mode
- The PE environment can run the Windows SFX BIOS updater without issues
- Ventoy handles both BIOS and UEFI boot transparently

---

## Step 3: Install Ventoy and copy files (RUN MANUALLY)

The agent is **blocked from executing `dd` or `mkfs` on raw block devices**. Run these commands in your terminal:

```bash
# 1. Identify your USB device (DO NOT use your system drive!)
lsblk

# 2. Unmount any existing partitions on the USB
sudo umount /dev/sda1 2>/dev/null

# 3. Wipe the partition table (critical — Ventoy needs a clean device)
sudo dd if=/dev/zero of=/dev/sda bs=1M count=10

# 4. Install Ventoy to the USB (replaces the partition table with Ventoy's)
cd /tmp/ventoy-1.1.17
sudo ./Ventoy2Disk.sh -I /dev/sda

# 5. Mount the Ventoy data partition
sudo mkdir -p /mnt/ventoy-usb
sudo mount /dev/sda1 /mnt/ventoy-usb

# 6. Copy the HBCD PE ISO to the USB
cp /tmp/HBCD_PE_x64.iso /mnt/ventoy-usb/

# 7. Copy the BIOS executable to the USB
cp ~/Downloads/ACER/acer_bios_update/ZPN_V1_32.exe /mnt/ventoy-usb/

# 8. Unmount
sudo umount /mnt/ventoy-usb
```

**Note:** The USB will appear as a FAT32 partition (~29 GB) after Ventoy installation. The ISO files sit on this partition and Ventoy loads them at boot time.

---

## Step 4: Boot and flash

1. Plug USB into the Acer laptop
2. Plug in AC power (critical — power loss bricks the BIOS)
3. Power on, press **F12** repeatedly to open the boot menu
4. Select the USB drive from the boot menu
5. Ventoy will present a menu — select `HBCD_PE_x64.iso`
6. In the Windows PE environment, open Command Prompt
7. Navigate to the USB drive (usually `X:` or `D:`) and run:
   ```
   ZPN_V1_32.exe
   ```
8. The Insyde Flash Tool extracts and flashes the BIOS automatically
9. The laptop will reboot on its own when done. Let it complete fully.

---

## Important Notes

- **fwupd does NOT have a BIOS capsule for most Acer models** — it only offers Secure Boot dbx (forbidden signature) updates, not actual BIOS version updates.
- The SFX file is a 7-Zip SFX stub (PE32+ x86-64). The embedded 7z archive is encrypted with Insyde's proprietary algorithm — no Linux tool can extract it.
- **FreeDOS does NOT work correctly with UEFI BIOS** on modern Acer laptops. Use HBCD PE instead.
- Ventoy 1.1.17 is the latest version (released 2026-07-24).
- Required packages: `fwupd`, `fwupd-doc`, `fwupd-tests`, `p7zip-full`, `dmidecode`, `lshw`
- The USB must be wiped (`dd if=/dev/zero`) before installing Ventoy, or Ventoy2Disk.sh will fail with `/dev/sda2 not exist`.
- HBCD PE download URL: `https://download.hirensbootcd.org/files/PE/HBCD_PE_x64.iso` (site has SSL cert issues — use `curl --insecure`)
