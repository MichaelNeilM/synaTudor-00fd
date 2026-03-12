# synaTudor — Synaptics SYN 79A Fingerprint Driver for Linux
### (USB 06cb:00fd — Lenovo ThinkBook 14 G3 ACL and similar)

> **Hey!** If you have a Lenovo ThinkBook 14 G3 ACL (or any laptop with a USB `06cb:00fd` fingerprint sensor) and your fingerprint scanner just... doesn't work on Linux — this is for you. I was in the same boat. I got it working and I'm sharing everything here so you don't have to go through what I went through.

---

## What even is this?

OK so imagine your fingerprint scanner is a little kid who only speaks Windows. Linux doesn't speak Windows. Normally that means your fingerprint scanner just sits there doing nothing and everyone ignores it.

What this project does is basically hire a translator. It takes the official Synaptics Windows fingerprint driver (the thing that actually knows how to talk to the scanner), teaches it just enough Linux to run here, and then connects it to `fprintd` — the thing Linux uses to handle fingerprints for login and `sudo` and stuff like that.

This is a fork of the amazing [synaTudor project by Popax21](https://github.com/Popax21/synaTudor). All the hard work of building that translator was done there. I just added support for the specific driver version that the `06cb:00fd` sensor needs, fixed a bunch of crashes, and made it actually work end-to-end.

---

## Does this work for my laptop?

Check your fingerprint sensor's USB ID. Run this in your terminal:

```sh
lsusb | grep -i 06cb
```

If you see `06cb:00fd` in the output, **yes, this is for you.**

This was specifically developed and tested on a **Lenovo ThinkBook 14 G3 ACL** running **Arch Linux**. It may work on other distros and other laptops with the same sensor — if it works for you, let me know!

---

## What you need before you start

Install these packages first. On Arch Linux:

```sh
sudo pacman -S libusb innoextract wget
```

You also need **libfprint-tod** instead of the regular libfprint. They conflict, so you have to swap them out:

```sh
sudo pacman -Rdd libfprint --noconfirm
```

Then install `libfprint-tod-git` from the AUR:

```sh
yay -S libfprint-tod-git
# or however you install AUR packages
```

You also need `fprintd`:

```sh
sudo pacman -S fprintd
```

---

## How to build and install

Clone this repo:

```sh
git clone https://github.com/MichaelNeilM/synaTudor-00fd.git
cd synaTudor-00fd
```

Build it (the first build will automatically download the Windows driver from Lenovo — so you need internet access):

```sh
arch-meson build
ninja -C build
sudo ninja -C build install
```

> If you're not on Arch Linux, use `meson build` instead of `arch-meson build`.

Reload udev rules so your system recognizes the sensor:

```sh
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Enable and start the services:

```sh
sudo systemctl enable --now tudor-host-launcher
sudo systemctl enable --now fprintd
```

---

## Enroll your fingerprint

Once everything is installed and running, enroll a finger:

```sh
fprintd-enroll -f right-index-finger
```

Follow the prompts — keep moving your finger around on the sensor until it says `enroll-completed`. It usually takes about 10-12 scans.

Then verify it works:

```sh
fprintd-verify
```

You should see `verify-match` if everything is working.

To use your fingerprint for `sudo` and login, add this to `/etc/pam.d/sudo` (and/or your login manager's PAM config):

```
auth sufficient pam_fprintd.so
```

---

## If something goes wrong

The most common issue is the tudor host getting into a bad state (especially if enrollment gets interrupted). Just restart the services:

```sh
sudo systemctl restart tudor-host-launcher fprintd
```

Then try again. That fixes it 99% of the time.

If you're seeing crashes or weird errors, check the logs:

```sh
journalctl -u tudor-host-launcher -n 50
```

---

## How does it actually work? (the nerd version)

The Synaptics Tudor fingerprint sensor uses a Windows UMDF 2.21 driver (`synaWudfBioUsb139.dll`) that talks USB to the sensor and handles all the fingerprint matching and enrollment logic. On Linux, none of that exists natively.

synaTudor solves this by:
1. Loading the Windows PE DLL directly into memory
2. Resolving all its Windows API imports against a custom Linux implementation (basically a tiny fake Windows environment)
3. Emulating the Windows Driver Framework (WDF) function table so the driver thinks it's running on Windows
4. Connecting the driver's WINBIO interfaces to libfprint-tod so Linux fingerprint tools can use it

My additions to make `06cb:00fd` work specifically:

- Switched to the correct Lenovo driver package for the ThinkBook 14 G3 ACL (version 6.0.506.1139)
- Stubbed 10 additional WDF functions (indices 42-56) that the newer UMDF 2.21 driver requires
- Fixed a null-pointer crash in `GetModuleHandleW(NULL)` — valid Windows API call the newer DLL makes
- Added stubs for USER32 and WTSAPI32 functions the driver calls during initialization (window class registration, power notifications, terminal services session notifications — none of which are actually used by a USB fingerprint driver, but the DLL tries to call them anyway)
- Added missing registry API functions (`RegQueryInfoKeyW`, `RegEnumValueW`, etc.)
- Increased the IPC buffer sizes from 96KB to 256KB — the new driver generates larger fingerprint templates than the old one and was hitting the limit

---

## Credits

- [Popax21](https://github.com/Popax21) — built the entire synaTudor framework that makes all of this possible
- Everyone who contributed to the original synaTudor project
- This fork was developed with help from [Claude Code](https://claude.ai/code)

---

## Disclaimer

THIS PROJECT IS PROVIDED AS IS, WITHOUT WARRANTY OR LIABILITY OF ANY KIND. USE AT YOUR OWN RISK. I'm not responsible if something breaks. That said — it works great on my machine and I use it every day.

If it works for you, drop a star. If it doesn't, open an issue and let me know what laptop/sensor you have and what errors you're seeing. The more people who contribute, the more devices we can get working.
