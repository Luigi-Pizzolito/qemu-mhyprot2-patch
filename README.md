# qemu-mhyprot2-patch

Patched QEMU plus a small host kernel module for running games protected by
mhyprot2 (Genshin Impact) in a VFIO Windows VM.

Combines:
- [zhaodice/qemu-anti-detection](https://github.com/zhaodice/qemu-anti-detection): renames QEMU/virtual-device identifiers (`qemu-10.2.2.patch`).
- The vmcall patch (derived from [this qemu-devel post](https://lists.nongnu.org/archive/html/qemu-devel/2025-07/msg05044.html)): disables KVM's hypercall-rewrite quirk and makes `vmcall`/`vmmcall` raise #UD.

## Why

Without it, the anti-cheat's kernel-mode `vmcall` probe makes KVM rewrite the
guest's instruction bytes. That write hits a read-only kernel page and Windows
bugchecks with `0xBE ATTEMPTED_WRITE_TO_READONLY_MEMORY`.

- `qemu-<version>.patch` sets `KVM_X86_QUIRK_FIX_HYPERCALL_INSN` off through `KVM_CAP_DISABLE_QUIRKS2` in `kvm_arch_init`.
- `kvm-hypercall-patch/` hooks `kvm_emulate_hypercall` so guest hypercalls get #UD, as on real hardware.

## Usage

Requires the build dependencies (Arch: `sudo pacman -S git wget base-devel glib2 ninja python`),
`dkms`, and the headers for your running kernel.

```bash
./run
```

This downloads QEMU 11.1.3, applies the patch, builds, installs over the system QEMU in `/usr`
(use `PREFIX=/usr/local ./run` to keep the package-managed copy; libvirt then needs that emulator path), installs the DKMS module, and enables
it at boot through `/etc/modules-load.d/kvm-hypercall-patch.conf`.

The patched version must match the QEMU package you have installed, otherwise the binary and the other QEMU files will be mixed.
A distro QEMU update overwrites the patched binary, so run `./run` again after updating (and port the patch if the version changed).

Then fully shut down and restart the VM. Verify with:

```bash
lsmod | grep hypercall
sudo dmesg | grep -i "hypercall ud hook"
```

## Layout

| Path | Purpose |
|------|---------|
| `qemu-11.1.3.patch` | Combined QEMU patch for 11.1.3 (matches current Arch `qemu`) |
| `qemu-10.2.2.patch` | Same patch for 10.2.2 |
| `kvm-hypercall-patch/` | DKMS module making vmcall/vmmcall raise #UD |
| `configs/samuil1337.xml` | Example libvirt domain |
| `ssdt1.dat` | Optional ACPI table adding a fake laptop battery (load with `-acpitable file=ssdt1.dat`); not referenced by the example config |
| `run` | Build and install script |

Other QEMU versions: add a `qemu-<version>.patch` and run `QEMU_VERSION=<version> ./run`.
Other versions usually need their patch ported by hand, since some string hunks move between releases.
Build against the same version your distro ships (currently 11.1.3 on Arch), so the installed binary matches the rest of the QEMU package.
