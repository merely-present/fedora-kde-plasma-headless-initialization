# Fedora - Multi-User Target Setup (Headless)

## Preamble

This document outlines the steps for setting the default boot target to a headless state (`multi-user.target`) on Fedora, particularly for KDE Plasma systems. This prevents the display manager from starting on boot and avoids automatic suspend issues caused by an idle desktop session.

## Setup Steps

### Set default boot target to multi-user (headless)

Setting the default target to `multi-user` prevents the display manager from starting on boot and also prevents automatic suspend from triggering due to an idle desktop session. If any GUI setup needs to be completed first, this step can be deferred.

Use `sudo systemctl isolate multi-user.target` to switch to the headless target live without rebooting, then run `set-default` once done:

```bash
sudo systemctl set-default multi-user.target
```

### Reverting to Graphical Target

To switch back to the graphical target at any time:

```bash
sudo systemctl isolate graphical.target
sudo systemctl set-default graphical.target
```
