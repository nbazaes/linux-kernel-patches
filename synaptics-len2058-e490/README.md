# Input: synaptics — add LEN2058 to SMBus passlist for ThinkPad E490

**Subsystem:** `drivers/input/mouse/synaptics.c`  
**Status:** Under review by Dmitry Torokhov  
**Patch:** [lore.kernel.org](https://lore.kernel.org/linux-input/20260514013552.14234-1-contacto@bazaes.cl/)

## Patch History

- **v1** — Initial submission
  [lore.kernel.org](https://lore.kernel.org/linux-input/20260513045749.185969-1-contacto@bazaes.cl/)

- **v2** — Updated commit message after review by Dmitry Torokhov.
  Clarified that `psmouse.synaptics_intertouch=1` is not ignored when
  explicitly set — the passlist is only checked when the parameter is
  unset. Also fixed `Assisted-by` tag format per kernel documentation.
  [lore.kernel.org](https://lore.kernel.org/linux-input/20260514013552.14234-1-contacto@bazaes.cl/)

---

## Problem

The ThinkPad E490 touchpad (Synaptics TM3471-020, PNP ID: `LEN2058`) felt sluggish and unresponsive under Linux. Despite the hardware supporting RMI4 over SMBus — a richer protocol than PS/2 — the kernel was falling back to PS/2 mode silently.

Symptoms:

- Touchpad felt heavy and slow regardless of libinput acceleration settings
- `libinput list-devices` showed no touchpad device at all
- `sudo dmesg` revealed the kernel knew about a better bus but wasn't using it

## Root Cause Analysis

The kernel's `synaptics.c` driver maintains a hardcoded allowlist (`smbus_pnp_ids[]`) of PNP IDs for which SMBus/RMI4 mode is permitted. This list already included similar ThinkPad models:

```c
"LEN2054", /* E480 */
"LEN2055", /* E580 */
// LEN2058 was missing here
"LEN2068", /* T14 Gen 1 */
```

The E490's PNP ID `LEN2058` was absent from this list. When `psmouse` detected the touchpad, it attempted SMBus negotiation, found no matching entry in the allowlist, and silently fell back to PS/2 mode. The parameter `psmouse.synaptics_intertouch=1` was ignored entirely because the check happens before the parameter is evaluated.

This was confirmed by reading the kernel source:

```c
if (!psmouse_matches_pnp_id(psmouse, smbus_pnp_ids))
    return -ENXIO;  // bail out silently, fall back to PS/2
```

## The Fix

A one-line addition to `smbus_pnp_ids[]` in `drivers/input/mouse/synaptics.c`:

```diff
  "LEN2055", /* E580 */
+ "LEN2058", /* E490 */
  "LEN2068", /* T14 Gen 1 */
```

## Testing

Tested on ThinkPad E490 running Arch Linux with kernel `7.0.5-zen1`.

After patching and recompiling the `psmouse` module, `dmesg` confirmed RMI4 over SMBus:

```
psmouse serio1: synaptics: Trying to set up SMBus access
rmi4_smbus 6-002c: registering SMbus-connected sensor
rmi4_f01 rmi4-00.fn01: found RMI device, manufacturer: Synaptics, product: TM3471-020
input: Synaptics TM3471-020 as /devices/pci0000:00/...
input: TPPS/2 Elan TrackPoint as /devices/pci0000:00/...
```

Notably, the fix works **without** the `psmouse.synaptics_intertouch=1` kernel parameter — the device is detected and switched to RMI4 automatically once `LEN2058` is in the allowlist.

## How to Build the Patched Module

If your kernel does not yet include this patch, you can compile and install the module manually:

```bash
# Clone kernel sources at your exact version
git clone --depth 1 --branch <your-zen-tag> https://github.com/zen-kernel/zen-kernel.git
cd zen-kernel

# Copy your running kernel config
zcat /proc/config.gz > .config

# Apply the patch
patch -p1 < path/to/0001-Input-synaptics-add-LEN2058.patch

# Prepare build system
make prepare && make scripts

# Compile only the mouse module directory
make -C /usr/lib/modules/$(uname -r)/build M=$(pwd)/drivers/input/mouse modules -j$(nproc)

# Install
zstd drivers/input/mouse/psmouse.ko -o psmouse.ko.zst
sudo cp psmouse.ko.zst /usr/lib/modules/$(uname -r)/kernel/drivers/input/mouse/psmouse.ko.zst
```

Then configure libinput in your compositor. For Sway (`~/.config/sway/config`):

```
input type:touchpad {
    natural_scroll enabled
    tap enabled
    dwt enabled
    middle_emulation enabled
    accel_profile adaptive
    pointer_accel 0.4
}
```

## References

- [Patch on lore.kernel.org](https://lore.kernel.org/linux-input/20260513045749.185969-1-contacto@bazaes.cl/)
- [Validity90 — prior art for similar Synaptics SMBus patches](https://github.com/nmikhailov/Validity90)
- Maintainer: Dmitry Torokhov `<dmitry.torokhov@gmail.com>`
