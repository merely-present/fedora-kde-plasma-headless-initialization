# Fedora KDE Plasma Headless Initialization

This repository contains Ansible playbooks, configurations, and notes to automate the setup of a headless Fedora KDE Plasma server. It is primarily designed for a home rack environment focused on self-hosting applications, containers, and running the non-inference parts of local AI pipelines.

The main goal of this project is to quickly reproduce the environment and avoid having to relearn or troubleshoot the specific quirks of headless Fedora setups if a fresh reinstall is ever needed.

## Repository Structure

- `site.yml`: The main Ansible playbook that orchestrates the overall setup.
- `tasks/`: Individual Ansible task files (e.g., FDE decryption via SSH, multi-user targets, headless monitor poweroff, dotfiles setup).
- `dotfiles/`: Standard configuration files, shell scripts, or dotfiles to be copied over to the server (e.g., `.bashrc`, aliases, container network configurations).
- `notes/`: Documentation on specific hurdles, structural decisions, and manual debugging steps.

## Prerequisites

Before running the playbook, ensure the following steps are completed on your local controlling machine and the target server:

1. **Install OS**: Install Fedora (with KDE Plasma) on the target server.
    - During installation
        - Set up LUKS full disk encryption
        - Create a user account with sudo privileges (this will be used for Ansible access)
        - Ensure the system is connected to the network and has SSH enabled for remote access.

            ```bash
            sudo systemctl enable sshd --now
            ```

2. **Setup Access**: Add your public SSH key to the target server's `~/.ssh/authorized_keys`.

    ```bash
    mkdir -p ~/.ssh && printf '{public_key}\n' > ~/.ssh/authorized_keys
    ```

    - (Optional) [Add the server's IP address and username to your local `~/.ssh/config` for easier access](notes/fedora-dracut-ssh-fde-decryption-setup.md#7-configure-the-ssh-client)
3. **Install Ansible**: Install Ansible on your local machine

   ```bash
   sudo dnf install ansible
   ```

## Usage

Instead of maintaining a static `inventory` file, you can run the initialization directly against your target server on the fly, right from the terminal.

Run the following command, replacing `{server_ip}` with your server's IP address, `{username}` with your target user, and `{path_to_private_key}` with the path to your private key:

```bash
ansible-playbook -i "{server_ip}," -u {username} --private-key {path_to_private_key} site.yml --ask-become-pass
```

*(The `-K` flag, or `--ask-become-pass`, will prompt for the sudo password required for privilege escalation).*
