# Fedora - Firewall Zone Setup

## Preamble

This document outlines the approach for ensuring the active network interface on the headless server is assigned to the `home` firewalld zone. By default, Fedora may assign interfaces to the `public` or `FedoraWorkstation` zones. Moving the primary interface to the `home` zone allows for more appropriate internal network access constraints for a self-hosted rack environment. Notably, one of the primary conveniences and reasons to switch is that the `home` zone permits `mdns` (Multicast DNS) traffic by default, seamlessly enabling easy local hostname resolution.

## Requirements

- Fedora installed and connected to the network
- `firewalld` installed and running

## Implementation

To update the default active interface to the `home` zone via command line:

```bash
# Set the interface to the home zone permanently
sudo firewall-cmd --zone=home --change-interface=$(ip route | grep default | grep -Po '(?<=dev )[^ ]+') --permanent

# Reload the firewall to apply changes
sudo firewall-cmd --reload
```

## Verification

Check the active zones and confirm the interface is listed under `home`:

```bash
sudo firewall-cmd --get-active-zones
```

The output should be:

```plaintext
home
  interfaces: e*
```
