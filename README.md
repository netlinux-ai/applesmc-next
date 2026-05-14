# applesmc-next

Kernel patches that expose battery charge thresholds on Intel Apple laptops
running Linux. Adds `charge_control_end_threshold` (and friends) to
`/sys/class/power_supply/BAT0/`, mapping writes to the SMC `BCLM` key so the
limit is stored in SMC NVRAM and survives reboots — even back into macOS.

This is the **netlinux-ai** fork of [`c---/applesmc-next`](https://github.com/c---/applesmc-next).
It carries fixes and packaging for recent Debian / Ubuntu kernels in addition
to the upstream features.

## What it does

Once the modules are loaded, the kernel battery exposes:

```
/sys/class/power_supply/BAT0/charge_control_end_threshold     # BCLM
/sys/class/power_supply/BAT0/charge_control_full_threshold    # BFCL (cosmetic)
/sys/class/power_supply/BAT0/charge_control_start_threshold   # stock kernel; not driven by BCLM
```

Writing an integer 20–100 to `charge_control_end_threshold` makes the SMC stop
charging at that percentage. The setting is persistent in SMC NVRAM across
reboots and operating systems unless the SMC is reset.

```sh
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

`charge_control_full_threshold` only controls when the charging LED turns
green and is cosmetic. For it to work, set it at least 2% **below** the end
threshold, because the battery floats slightly under its target.

## What's in this fork

* **Two-module DKMS package** (`applesmc-next-dkms`) — replaces stock
  `applesmc` and `sbs` with patched copies in `/lib/modules/.../updates/dkms/`.
  `AUTOINSTALL=yes` means it rebuilds on every kernel upgrade.
* **GitHub Actions** that build `.deb` packages for **Debian Bookworm** and
  **Ubuntu Resolute (26.04)** on every release.
* **Fix for kernels where `sbshc` is a `platform_driver`** (notably the Ubuntu
  7.0 kernel in Resolute). On those kernels the host controller pointer lives
  in the parent's *platform device* drvdata rather than its ACPI driver_data;
  without this fix the probe oopses on a NULL `acpi_smb_hc` pointer and the
  module is unrecoverable until reboot.
* **TLP integration** — drop `45-apple` into `/usr/share/tlp/bat.d/` for
  charge-threshold control via TLP profiles.

## Install

### From the `.deb` (Debian / Ubuntu)

Grab the `.deb` for your release from the [releases page](https://github.com/netlinux-ai/applesmc-next/releases),
then:

```sh
sudo apt install ./applesmc-next-dkms_*.deb
```

This pulls in `dkms`, builds and signs both modules against your running
kernel, and installs them into `/lib/modules/$(uname -r)/updates/dkms/`.
After install:

```sh
sudo modprobe -r applesmc sbs           # drop the stock modules
sudo modprobe sbs                       # load the patched sbs first
sudo modprobe applesmc                  # then applesmc (it hooks into sbs)
ls /sys/class/power_supply/BAT0/        # should now contain charge_control_*_threshold
```

### From source (any distro)

```sh
sudo apt install dkms linux-headers-$(uname -r)
sudo git clone https://github.com/netlinux-ai/applesmc-next /usr/src/applesmc-next-0.1.6
sudo dkms add     -m applesmc-next -v 0.1.6
sudo dkms install -m applesmc-next -v 0.1.6
```

### From the AUR (Arch)

[`applesmc-next-dkms`](https://aur.archlinux.org/packages/applesmc-next-dkms)
on the AUR points at the original `c---/applesmc-next` upstream.

## Use

Direct sysfs:

```sh
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
cat /sys/class/power_supply/BAT0/charge_control_end_threshold        # -> 80
```

To remove the limit, write `100`.

### With TLP

Copy `45-apple` into TLP's plugin directory:

```sh
sudo install -Dm755 45-apple /usr/share/tlp/bat.d/45-apple
```

Then add the standard TLP keys to `/etc/tlp.conf`:

```ini
START_CHARGE_THRESH_BAT0=0      # not used; SMC BCLM has no start threshold
STOP_CHARGE_THRESH_BAT0=80
```

### GUI

If you want a tray app to drive the same sysfs file, see
[`netlinux-ai/battery-tray`](https://github.com/netlinux-ai/battery-tray).

## Kernel compatibility

| Kernel | Status |
|---|---|
| 6.1 – 6.17 | builds against `acpi_driver`-based sbshc |
| 6.18+ / Ubuntu 7.0 (Resolute) | builds against `platform_driver`-based sbshc — needs the fix in this fork |
| < 6.1 | legacy `device->parent` lookup path |

Tested on:

* MacBookPro9,2 (Ivy Bridge HD 4000) — Ubuntu 26.04, kernel 7.0.0-14-generic
* MacBookPro models with Smart Battery System (ACPI device `SBS0` under `SBSH`)

## Troubleshooting

* **`charge_control_end_threshold` doesn't appear** — patched modules aren't
  loaded. Check `dkms status applesmc-next` and `modinfo sbs | head -1` (it
  should resolve to `/lib/modules/.../updates/dkms/sbs.ko.zst`, not the
  in-tree `/kernel/drivers/acpi/sbs.ko.zst`). If stock modules are still
  loaded, `modprobe -r` them and `modprobe` again.

* **`modprobe sbs` returns 0 but BAT0 is missing** — sbs probably hit
  `-EPROBE_DEFER`. Check `sudo cat /sys/kernel/debug/devices_deferred` —
  if `ACPI0002:00` is parked there and you're on a recent kernel without
  this fork's fix, you need to update.

* **Kernel oops with `mutex_lock+0x1c` and `CR2: 0x0000000000000008`** — this
  fork's commit history fixes exactly that. Pre-fix builds (0.1.6 and earlier)
  are unrecoverable without a reboot once they hit this.

* **Writes to `charge_control_full_threshold` return I/O error** — the BFCL
  SMC key doesn't exist on all models (e.g. MacBookPro9,2). The end threshold
  still works; full is cosmetic-only.

## License

GPL-2.0 — see `LICENSE`.

## Credits

* Original work by [@c---](https://github.com/c---) and contributors.
* Platform-driver / Ubuntu-Resolute fix and Debian/Ubuntu packaging by
  netlinux-ai.
