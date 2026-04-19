# Fedora - Cockpit Management Console

## Preamble

Cockpit is a web-based graphical interface for servers, providing simple system administration, container management, network configuration, and live resource monitoring directly from a browser. This prevents having to ssh into the terminal just to do simple checks or restarts on a headless machine.

## Requirements

- RedHat/CentOS/Fedora system with cockpit package available

## Installation & Setup

Install the cockpit package:

```bash
sudo dnf install cockpit
```

Enable and start the socket (not the service directly, it runs on demand):

```bash
sudo systemctl enable --now cockpit.socket
```

Allow the service through the firewall in the designated zone (`home`):

```bash
sudo firewall-cmd --zone=home --add-service=cockpit
sudo firewall-cmd --zone=home --add-service=cockpit --permanent
```

## Usage

Access the console by navigating to:
`https://{server_ip}:9090`

Log in using the sudo-enabled credentials configured on the server.
