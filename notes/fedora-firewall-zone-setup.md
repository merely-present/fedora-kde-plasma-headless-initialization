# Fedora - Firewall Zone Setup

## Preamble

This document outlines the approach for ensuring the active network connection on the headless server is assigned to the `home` firewall zone. By default, Fedora may assign interfaces to the `public` or `FedoraWorkstation` zones. Moving the primary connection to the `home` zone allows for more appropriate internal network access constraints for a self-hosted rack environment.

Notably, one of the primary conveniences and reasons to switch is that the `home` zone permits `mdns` (Multicast DNS) traffic by default.

## Requirements

- Fedora installed and connected to the network
- `firewalld` installed and running
- `NetworkManager` managing the active network connection

## Implementation

Configure the firewall zone through NetworkManager (`nmcli`) rather than `firewall-cmd` directly. NetworkManager owns the zone assignment for each connection profile and instructs firewalld to apply it whenever a connection comes up. This overwrites the runtime zone on the interface regardless of what firewalld's own permanent config says, making `firewall-cmd` the wrong tool for persistent zone configuration when NetworkManager is running.

To update the active connection to the `home` zone via the command line:

```bash
# Get the active interface name
ACTIVE_INTERFACE=$(ip -json route show default | jq -r '.[0].dev')

# Get the active connection name
NETWORK_CONNECTION_NAME=$(nmcli --get-values GENERAL.CONNECTION device show "$ACTIVE_INTERFACE")

# Set the connection's firewall zone
sudo nmcli connection modify "$NETWORK_CONNECTION_NAME" connection.zone home

# Re-establish the connection to apply the zone change and update the DHCP hostname
sudo nmcli connection up "$NETWORK_CONNECTION_NAME"
```

## Verification

You can verify the connection profile's designated zone using NetworkManager:

```bash
nmcli --get-values connection.zone connection show "$NETWORK_CONNECTION_NAME"
```

The expected output is:

```plaintext
home
```

Additionally, check the active zones via `firewalld` to confirm the interface is correctly operating under the newly assigned zone across the active network layer:

```bash
sudo firewall-cmd --get-active-zones
```

The output includes all active zones and the interfaces bound to them. You should verify that `home` is listed with the correct interface(s) resembling the following:

```plaintext
home
  interfaces: e*
```
