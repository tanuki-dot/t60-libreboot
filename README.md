# Libreboot on the ThinkPad T60 (14.1" XGA, Intel GPU)

A complete, walkthrough of putting Libreboot on a T60, written while I was actually doing it. It covers the happy path (external flash) and the part where things went sideways and I had to recover over external SPI. I've left my mistakes in on purpose, because they're the parts that cost me real time, and they're probably the parts you'll hit too.

![Libreboot](https://img.shields.io/badge/Libreboot-26.01-blue)
![Machine](https://img.shields.io/badge/ThinkPad-T60-black)
![GPU](https://img.shields.io/badge/GPU-Intel%20i945-lightgrey)
![Panel](https://img.shields.io/badge/Panel-14.1%22%20XGA-green)
![Status](https://img.shields.io/badge/status-working-success)

> [!NOTE]
> The single biggest time sink for me was SPI bus contention during the external flash. If your programmer says it "never sees the chip," skip ahead to [section 6](#6-rescue-via-external-spi-raspberry-pi-pico-serprog). That one insight would have saved me a whole evening.

> [!CAUTION]
> Flashing firmware can temporarily brick the machine. On the T60 a brick is always recoverable over external SPI, but only if you already have a programmer and a SOIC-8 clip on hand. Set up and test your external rig before you flash internally. Your hardware is your responsibility, not mine.

---

## Contents

1. [Is the XGA panel supported?](#1-is-the-xga-panel-supported)
2. [What you'll need](#2-what-youll-need)
3. [Choosing and downloading the ROM image](#3-choosing-and-downloading-the-rom-image)
4. [Internal flash (bucts method)](#4-internal-flash-bucts-method)
5. [Problem: black screen after the internal flash](#5-problem-black-screen-after-the-internal-flash)
6. [Rescue via external SPI (Raspberry Pi Pico serprog)](#6-rescue-via-external-spi-raspberry-pi-pico-serprog)
7. [Finishing up: reset bucts and boot](#7-finishing-up-reset-bucts-and-boot)
8. [Post-install issues and fixes](#8-post-install-issues-and-fixes)
9. [Summary: my main problems and solutions](#9-summary-my-main-problems-and-solutions)
10. [Resources](#10-resources)

---

## 1. Is the XGA panel supported?

Yes, it is. You'll still find plenty of pages saying XGA (1024x768) doesn't work on a T60 with the Intel GPU, but that information is simply out of date. It dates back to when Libreboot relied on a reverse-engineered VBIOS, and it lives on in old wikis and a release note from 2014.

What changed is that for the i945 (X60 and T60), Libreboot switched to libgfxinit, which is coreboot's own native graphics init. That took care of the XGA panels. The current official T60 install page no longer mentions any resolution limit at all. The only caveat left is that you must have the Intel GPU, not the ATI one, and you should never flash the ATI models.

So if you're on a 14.1" XGA panel, you're fine. Upgrading to SXGA+ is a nice to have but not a requirement. Before you start, it's worth dumping the panel's EDID just to confirm it reports valid data:

```bash
cat /sys/class/drm/card0-LVDS-1/edid | edid-decode
```

When you pick the ROM later, make sure the filename contains libgfxinit (with display type corebootfb).

---

## 2. What you'll need

Software and files:

- A ROM image from the current Libreboot release (see [section 3](#3-choosing-and-downloading-the-rom-image)).
- The Libreboot 20160907 utils, which contain bucts, flashrom_lenovobios_sst, flashrom_lenovobios_macronix and flashrom.
- flashprog, the modern fork of flashrom, for the external flash.

Hardware for the internal flash:

- A 64-bit Linux running on the T60 itself. That means a Core 2 Duo, not a 32-bit Core Duo.
- A healthy CR2032 coin cell, which holds the BUC.TS bit.
- No BIOS password set.

Hardware for the external flash (your rescue path, and honestly your insurance policy):

- A Raspberry Pi Pico, or an RPi 3B or Zero 2 W used as linux_spi.
> [!CAUTION]
> You can also use a CH341a proggramer but watch out, sometimes the old ver comes in 5v.
- A SOIC-8 clip. A Pomona 5250 is the nice one.
- Thin dupont wires, kept under 10 cm.
- A multimeter.
- Optionally a bulk cap (10 to 47 uF) plus a 100 nF, to steady the chip's power during writes.

---

## 3. Choosing and downloading the ROM image

The current stable release is Libreboot 26.01 (26.01rev1). The ROMs live in the `roms/` subfolder of the release.

A stock T60 has a 2 MB SPI flash, so take the variant without `_16mb` in the name:

```
libreboot-26.01rev1_t60_intelgpu.tar.xz
libreboot-26.01rev1_t60_intelgpu.tar.xz.sha512
```

Only reach for `t60_16mb_intelgpu` if you've physically swapped the flash chip for a 16 MB one.

Inside the tarball you'll find several ROM variants. The naming scheme is `payload_board_inittype_displaytype_keymap.rom`. For XGA on the Intel GPU you want:

```
seagrub_t60_intelgpu_libgfxinit_corebootfb_usqwerty.rom
```

Here's what those tokens mean:

| Token | Meaning |
|-------|---------|
| seagrub | SeaBIOS is primary and auto-loads GRUB. You can still reach the SeaBIOS menu with ESC. |
| libgfxinit + corebootfb | Native graphics with a framebuffer. This is the one you want. |
| (no nomicrocode) | Keep the microcode build. It's more stable. |

> [!TIP]
> The i945 is blob-free, so there's nothing to inject. Download, extract, flash.

Check the download before trusting it:

```bash
sha512sum -c libreboot-26.01rev1_t60_intelgpu.tar.xz.sha512
```

---

## 4. Internal flash (bucts method)

The internal flash works by exploiting a bug in the stock Lenovo BIOS. The top 64 KB bootblock is protected by SPI lockdown, so you flash in two passes and use bucts to set the Top Swap bit, which makes the machine boot from the backup bootblock in between.

Before you start, confirm your userland is 64-bit and boot with the right kernel parameter:

```bash
uname -m          # must print x86_64
```

Boot with `iomem=relaxed` on the kernel command line, otherwise `-p internal` will fail trying to reach /dev/mem. Have a healthy CR2032 in, the laptop on AC with the battery installed, and no BIOS password.

Then:

```bash
# 1. permissions
chmod +x bucts flashrom_lenovobios_sst flashrom_lenovobios_macronix flashrom

# 2. rename the ROM
mv seagrub_t60_intelgpu_libgfxinit_corebootfb_usqwerty.rom coreboot.rom

# 3. set the Top Swap bit
sudo ./bucts 1

# 4. first flash, run BOTH binaries (one matches your chip, SST or Macronix)
sudo ./flashrom_lenovobios_sst       -p internal -w coreboot.rom
sudo ./flashrom_lenovobios_macronix  -p internal -w coreboot.rom
```

> [!WARNING]
> The binary names don't match the docs. The official docs call them flashrom_i945_sst and _mx, but the real archive names them flashrom_lenovobios_sst and flashrom_lenovobios_macronix.

What you'll see next looks alarming but is actually the sign of success:

```
spi_block_erase_XX failed ...
No usable erase functions left.
FAILED!
Apparently at least some data has changed.
Your flash chip is in an unknown state.
```

That's exactly what's supposed to happen. The lockdown blocked writing the top block, but everything else, including the backup bootblock, went in fine. The `DO NOT REBOOT OR POWEROFF!` line that flashrom prints does not apply to the bucts method. Here you do want to power off now.

```bash
# 5. power off (not reboot), wait a few seconds, power back on
sudo poweroff
```

Once Libreboot comes up, do the second flash from within running Libreboot. This pass writes the whole chip, top block included, using plain flashrom:

```bash
sudo ./flashrom -p internal -w coreboot.rom     # must end with VERIFIED
sudo ./bucts 0
```

---

## 5. Problem: black screen after the internal flash

Here's where my first attempt went quiet. After the first flash, a poweroff and a power-on, I got a black screen. NumLock and CapsLock blinked once, the fan ramped up, and then nothing.

What's going on is that with bucts set to 1 the machine boots from the backup bootblock, which is now coreboot, but it didn't finish coming up. A black screen with blinking keyboard LEDs is an early-init hang. You can't fix it from the inside, because you can't reach the OS to run flashrom again.

Work through it in this order. First, try a few cold starts: hold power for 10 seconds, disconnect AC and the battery, hold power for about 30 seconds to drain the flea power, reconnect AC and power on. Do that two or three times, because coreboot sometimes needs two boots. If the LEDs never blink and it just keeps hanging, move to the external SPI flash in [section 6](#6-rescue-via-external-spi-raspberry-pi-pico-serprog). That's the reliable way out.

> [!NOTE]
> This is not a hard brick. The top bootblock is still the untouched Lenovo BIOS, so in the worst case you're right back where you started.

---

## 6. Rescue via external SPI (Raspberry Pi Pico serprog)

This is the heart of the guide, and the part where I lost the most time, so I'm going to be thorough. The nice thing about the Pico is that it runs at 3.3 V natively, which is exactly what the SST25VF016B wants, so you don't need a level shifter.

### 6.1 Flash the serprog firmware onto the Pico

Grab the .uf2 from libreboot-26.01rev1_serprog_pico.tar.xz. Hold BOOTSEL, plug in USB, and the Pico shows up as a drive. Copy the .uf2 onto it.

### 6.2 Pinout

The SST25VF016B is a SOIC-8, and the dot marks pin 1:

| Pin | Signal | | Pin | Signal |
|-----|--------|-|-----|--------|
| 1 | CE# (CS) | | 8 | VDD (VCC) |
| 2 | SO (MISO) | | 7 | HOLD# |
| 3 | WP# | | 6 | SCK (CLK) |
| 4 | VSS (GND) | | 5 | SI (MOSI) |

Wiring from the Pico to the chip. Note that the serprog firmware's table refers to the Pico by physical pin number, not GPIO number:

| Signal | Pico (physical pin / GPIO) | Chip (SOIC-8 pin) |
|--------|----------------------------|-------------------|
| CS | 7 / GP5 | 1 |
| MISO | 6 / GP4 | 2 |
| MOSI | 5 / GP3 | 5 |
| SCK | 4 / GP2 | 6 |
| GND | 3 (or 8) | 4 |

> [!WARNING]
> Keep pins 3 (WP#) and 7 (HOLD#) at VCC, either through a 10k resistor or tied straight to 3.3 V. Never put them on GND. If HOLD# is low, the chip's MISO output goes high-impedance and it will never answer you.

### 6.3 The key problem: SPI bus contention

This is the most common and most frustrating failure on the T60, so read this bit even if everything looks right. The symptom is maddening:

```
serprog: Programmer name is "pico-serprog"    # programmer OK
...
No SFDP signature found.                       # normal for SST
No EEPROM/flash device found.                  # chip NEVER answers
```

The programmer is clearly fine, you measure around 3.3 V on VCC, continuity beeps out on every wire, HOLD# and WP# are high, and the chip still refuses to respond.

The cause is contention. When you power the flash rail through the AC adapter in the charging port, which is what the Libreboot docs suggest for current reasons, the ICH7M southbridge is partly alive and keeps hold of the SPI bus. The chip never gets a chance to answer.

> [!IMPORTANT]
> The fix is to power only the chip, from the Pico's 3V3, with the rest of the board completely dead. Disconnect AC and the main battery, and pull the CMOS cell, so nothing on the board has power. Then run Pico 3V3 (physical pin 36) to chip pin 8 (VCC). An isolated chip draws only a few microamps, so the Pico handles it easily. With the southbridge dead, the contention disappears and the chip answers on the first try.

To be clear about why this doesn't contradict the official advice: the "don't connect VCC from the Pico" rule is only about the current draw of the whole rail. Once the board is otherwise completely dead, powering just the chip from the Pico is exactly what clears the contention.

### 6.4 Flash procedure

```bash
# find the device
dmesg -wH            # plug in the Pico, note ttyACMx

# 1. detection test (writes nothing)
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=512k -V
#    it must report: Found SST flash chip "SST25VF016B"
#    if not, drop to spispeed=100k, press the clip, and power the chip from the Pico ONLY

# 2. confirm stable reads (twice, then compare)
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=512k -r dump1.bin
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=512k -r dump2.bin
sha256sum dump1.bin dump2.bin        # matching hashes means it's safe to write

# 3. write
sudo flashprog -p serprog:dev=/dev/ttyACM0,spispeed=512k -w coreboot.rom
#    must end with: Erase/write done. Verifying flash... VERIFIED.
```

> [!TIP]
> Erase has current spikes, and the Pico's 3V3 (around 300 mA) can sag under them. A bulk cap (10 to 47 uF) plus a 100 nF between pin 8 and pin 4 at the chip smooths that out. Keep an eye on `dmesg -w`. If ttyACM drops out, that's a power sag, so add the cap, lower the speed, or use a bench 3.3 V supply.

<details>
<summary><b>6.5 Debugging checklist (when detection fails)</b></summary>

<br>

| Symptom | Cause | Fix |
|---------|-------|-----|
| Could not open /dev/ttyACM0 | serial or firmware | try another ACM number, use a data USB cable (not charge-only), re-flash the .uf2 |
| Programmer name is "pico-serprog" but No device found | contention, or poor clip contact | power only from the Pico with the board dead, press the clip, lower spispeed |
| No SFDP signature found | normal for SST | ignore it, it isn't an error |
| HOLD#/WP# on GND | chip held in Hold state | move them to VCC |
| intermittent, or fails during write | power sag or contact | bulk cap, hold the clip, bench 3.3 V supply |

If you want to know whether the problem is your rig or the board, probe a different SPI flash on a different, unpowered board (read-only). If it reports Found there, your rig is fine and the T60 contention was the culprit.

</details>

---

## 7. Finishing up: reset bucts and boot

```bash
# 1. take the rig apart safely: unplug Pico 3V3 from pin 8 first, then remove the clip
# 2. reset bucts. Full coreboot now lives in BOTH bootblocks, so either:
#    pull the CMOS cell for about 30 seconds (bucts goes to 0), then put it back
#    or, from Linux: ./bucts 0
# 3. reassemble, put the battery and CMOS back, connect AC, power on
```

Libreboot should come up, penguin then GRUB, on your XGA panel. That's the moment it all pays off.

---

## 8. Post-install issues and fixes

### 8.1 thinkfan spams errors

```
thinkfan: ERROR: Initializing fan control in /proc/acpi/ibm/fan: No such file or directory
```

The reason is that coreboot doesn't ship Lenovo's ACPI/DSDT, so thinkpad_acpi doesn't expose the fan node. Sometimes it's just a first-boot race and it's gone after a reboot.

> [!IMPORTANT]
> On a libre T60 the EC cools the fan by itself, independent of the OS. A `status: disabled` in /proc/acpi/ibm/fan only means software isn't driving it. It does not mean the fan is off.

Confirm the EC is actually cooling:

```bash
cat /proc/acpi/ibm/fan        # look at speed: for a non-zero RPM
sensors                        # idle should sit around 45 to 55 C
# load test:
yes > /dev/null & yes > /dev/null &   # the fan should audibly ramp
kill %1 %2
```

If the fan reacts, just disable thinkfan and move on:

```bash
sudo rm /var/service/thinkfan        # Void / runit
```

### 8.2 TLS and certificates fail after pulling CMOS

```
Certificate verification failed for ... ISRG Root X1
SSL_connect returned 1
```

This one is a classic chicken and egg. After you pull the CMOS cell, the RTC is wildly off, so TLS rejects every connection because the certificate looks outside its validity window. You can't download an NTP daemon, because HTTPS is down, because the clock is wrong.

Break the loop by setting the clock by hand, no network needed. The format is MMDDhhmmYYYY:

```bash
sudo date 101522452026        # 15 Oct 2026 22:45, adjust to the real time
date                           # confirm the year is right
# HTTPS and xbps work again now
```

Then set up proper time sync. On Void with runit you'll use openntpd, and note that the old `-s` and `-q` flags no longer work:

```bash
sudo xbps-install -S openntpd
sudo ln -sf /etc/sv/openntpd /var/service/     # the daemon fixes the clock itself

# for an immediate one-shot sync:
sudo xbps-install -S sntp
sudo sntp -sS pool.ntp.org
sudo hwclock --systohc                          # write it back to the RTC
```

> [!NOTE]
> If the clock resets on every normal shutdown, not just when you pull CMOS, your CR2032 is weak or dead. Replace it.

### 8.3 Wi-Fi card (for example Atheros AR5B22 / AR9462)

One of the quiet wins of Libreboot is that it removes the Wi-Fi whitelist, so cards the stock BIOS used to reject with `1802: Unauthorized network card` now just work.

- The Wi-Fi side (AR9462) uses ath9k, with no firmware blob, fully in-kernel.
- The Bluetooth side (AR3012) uses ath3k and needs a blob (ath3k-1.fw), but only if you want Bluetooth.
- Check it landed: `lspci -nnk | grep -iA3 atheros` should show `Kernel driver in use: ath9k`.
- If you run a custom kernel, you need CONFIG_ATH9K=m, CONFIG_ATH=m, CONFIG_MAC80211=m and CONFIG_CFG80211=m.
- Connect both antennas (MAIN and AUX).

### 8.4 Display upgrade (14.1")

- The ceiling for a 14.1" 4:3 is SXGA+ 1400x1050 (TN). IPS/FlexView was 15" only.
- Compatible panels include the Samsung LTN141P4-L01/L02, the BOE-Hydis HT14P12, and the TMD-Toshiba LTD141EN9B (FRU 13N7063 or 42T0373).
- The T60 uses the same LCD cable for XGA and SXGA+. Both LVDS channels are already wired, and the panel type is read from an EDID EEPROM on the panel itself, so you swap only the panel, not the cable or inverter.
- A Libreboot bonus: libgfxinit just reads the EDID and drives the panel, so you aren't locked to a blessed Lenovo FRU the way the stock BIOS was.

> [!WARNING]
> Watch the listings. The HT14P12 is an SXGA+ panel even when an ad claims it's XGA, so verify the label on the back of the panel before you buy.

### 8.5 More usable RAM

coreboot is leaner than the Lenovo BIOS, so it frees up a couple hundred MB through smaller firmware reservations. The 3.1 to 3.3 GB ceiling stays, though, because of the MMIO hole below 4 GB. The i945GM can't do memory remap. You can see the map with `cat /proc/meminfo | head -1` and `dmesg | grep e820`.

---

## 9. Summary: my main problems and solutions

| # | Problem | Cause | Fix |
|---|---------|-------|-----|
| 1 | "XGA isn't supported" (stale info) | outdated wiki from the VBIOS era | libgfxinit handles XGA, and the current docs have no restriction |
| 2 | Wrong binary names in the docs | the archive uses different names | flashrom_lenovobios_sst and _macronix, not _i945_* |
| 3 | Error storm on the first internal flash | SPI lockdown protects the top block | that's success, FAILED! is expected, power off and on |
| 4 | Black screen after the flash | half-flash plus bucts, early hang | cold starts, otherwise external SPI |
| 5 | External flash "never sees the chip" | contention, the ICH7M holds the bus via the powered rail | power only the chip from Pico 3V3, with the board completely dead (AC, battery and CMOS all out) |
| 6 | I tied HOLD#/WP# to GND | HOLD# low means MISO goes high-impedance | HOLD# and WP# go to VCC |
| 7 | thinkfan spam | coreboot has no Lenovo ACPI | the EC cools on its own, so disable thinkfan |
| 8 | TLS fails after a CMOS pull | the RTC is off, so the cert looks invalid | set date by hand, then openntpd plus hwclock --systohc |
| 9 | openntpd -s and -q don't work | different syntax from the systemd ntpd | ln -sf /etc/sv/openntpd /var/service/ plus sntp -sS |

Three things I'd do differently next time:

1. Test the external SPI rig before the internal flash. It saves a lot of grief when something goes wrong.
2. On the T60, go straight to the "chip only from the Pico, board dead" approach for external flashing. Don't even bother trying it with the rail powered.
3. After pulling CMOS, expect a broken clock, and set date by hand right after the first boot, before touching anything over HTTPS.

---

## 10. Resources

- Libreboot, official docs: <https://libreboot.org/docs/install/t60_unbrick.html>
- Libreboot, SPI and external flash: <https://libreboot.org/docs/install/spi.html>
- Libreboot pico-serprog: <https://codeberg.org/libreboot/pico-serprog>
- flashprog: <https://flashprog.org>
- SST25VF016B datasheet (Microchip DS20005044)
- Coreboot-ThinkPads wiki (bibanon), which is useful but partly outdated
