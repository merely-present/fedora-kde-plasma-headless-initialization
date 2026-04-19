# Fedora - Headless Monitor Idle Power-Off (multi-user.target)

## Preamble

This document outlines the steps for enabling automatic display power-off on a headless Fedora server running at multi-user.target (no graphical session). The goal is to have a physically connected monitor power off automatically after a period of inactivity, rather than sitting indefinitely at the TTY login prompt consuming power.

## Requirements

- Fedora running at multi-user.target (no display manager or graphical session)
- A monitor connected to the system
- grubby (included in Fedora by default)

## Implementation

Set the `consoleblank` kernel parameter. This blanks the virtual console framebuffer after a specified number of seconds of inactivity, which on tested hardware also triggers a DPMS powerdown signal, causing the monitor to cut power entirely (no signal).

Use grubby to add the parameter to all kernel boot entries:

```bash
sudo grubby --update-kernel=ALL --args="consoleblank=30"
```

The value is in seconds. Adjust to preference.

Verify the parameter was written:

```bash
sudo grubby --info=ALL
```

The `args` line for each entry should include `consoleblank=30`. The change takes effect on next reboot. After rebooting, confirm the parameter is active in the running kernel:

```bash
cat /proc/cmdline
```

## Notes

### grubby vs /etc/default/grub

On Fedora with Boot Loader Specification entries (the default since Fedora 30), kernel arguments for each boot entry live in individual `.conf` files under `/boot/loader/entries/`, not in `/boot/grub2/grub.cfg`. The intuitive approach of editing `GRUB_CMDLINE_LINUX` in `/etc/default/grub` and running `grub2-mkconfig` does not update these entries -- `grub2-mkconfig` regenerates `grub.cfg` but leaves the per-kernel arguments in the Boot Loader Specification entry files untouched. grubby is the correct tool for this setup, as it writes directly to those entry files. No follow-up command is required after running grubby.

## References

- [grubby(8) man page](https://man7.org/linux/man-pages/man8/grubby.8.html)
- [Fedora Magazine - Setting kernel command line arguments](https://fedoramagazine.org/setting-kernel-command-line-arguments-with-fedora-30/)
