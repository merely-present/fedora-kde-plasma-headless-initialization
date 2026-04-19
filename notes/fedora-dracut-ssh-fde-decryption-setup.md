# Fedora - Remote LUKS Decryption via SSH (dracut-sshd)

## Preamble

This document outlines the steps and considerations for enabling remote LUKS full disk encryption (FDE) unlock via SSH on a Fedora system using dracut-sshd. This setup allows a headless server with an encrypted root filesystem to be unlocked remotely over SSH during early boot, without requiring physical access to enter the passphrase.

The setup uses dracut-sshd, which integrates OpenSSH (sshd) into the initramfs, and systemd-networkd for early boot networking.

## Requirements

- Fedora (tested on Fedora 43) installed with LUKS full disk encryption enabled
- SSH key pair generated on the client machine used for connecting
- Network access between the client machine and the server during boot
- A DHCP reservation configured on the router for the server's MAC address (recommended, to ensure a predictable IP during early boot before mDNS is available)

## Installation Steps

### 1. Install dracut-sshd

dracut-sshd is available in the official Fedora package repositories, maintained by the upstream author [Georg Sauthoff](https://github.com/gsauthof).

Fedora includes the other prerequisites (`dracut`, `dracut-network`, `systemd-networkd`) by default, so only dracut-sshd itself needs to be installed:

```bash
sudo dnf install dracut-sshd
```

### 2. Enable OpenSSH (sshd) service

```bash
sudo systemctl enable --now sshd
```

### 3. Configure early boot networking

dracut-sshd requires network connectivity during early boot. The recommended approach is systemd-networkd, which is included in Fedora.

Create the networkd configuration file:

```bash
sudo mkdir -p /etc/systemd/network
sudo nano /etc/systemd/network/20-wired.network
```

Add the following content (adjust `Name=` if the interface name does not start with `e`):

```ini
[Match]
Name=e*

[Network]
DHCP=ipv4
```

Create the dracut configuration file to include networkd and the network config in the initramfs:

```bash
sudo mkdir -p /etc/dracut.conf.d
sudo nano /etc/dracut.conf.d/90-networkd.conf
```

Add the following content:

```plaintext
install_items+=" /etc/systemd/network/20-wired.network "
add_dracutmodules+=" systemd-networkd "
```

### 4. Add the authorized SSH public key

Place the public key that will be used to connect during early boot at `/root/.ssh/dracut_authorized_keys`:

```bash
sudo mkdir -p /root/.ssh
sudo nano /root/.ssh/dracut_authorized_keys
```

Paste the public key of the client machine. This file will be included in the initramfs on the unencrypted `/boot` partition. This is standard for this type of setup and not a concern in practice, as public keys are not secret by design (see notes for detail).

Set correct permissions:

```bash
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/dracut_authorized_keys
```

### 5. Fix root account shadow token

By default on Fedora, the root account is locked with `!` in `/etc/shadow`. The sshd config active during early boot is the one provided by dracut-sshd, which sets `UsePAM no`. This is the only sshd running at this stage. When PAM is disabled, OpenSSH treats `!` as a hard account lock and refuses all authentication including public key auth. Change the token to `*`, which resolves this while changing nothing else of consequence. Root still has no valid password, and password authentication remains impossible:

```bash
sudo usermod -p '*' root
```

Verify:

```bash
sudo grep root /etc/shadow
```

The password field should now start with `*` rather than `!`.

### 6. Rebuild the initramfs

```bash
sudo dracut -f -v --regenerate-all
```

### 7. Configure the SSH client

Because the initramfs has no normal user accounts, connections during early boot must be made as `root`. mDNS is not available during early boot, so connections must use the IP address directly. A DHCP reservation on the router ensures the IP is predictable.

Add the following entries to `~/.ssh/config` on the client machine:

```plaintext
# Initramfs unlock -- connects as root before disk is decrypted
Host {hostname}-init
    HostName {ip_address}
    User root
    IdentityFile ~/.ssh/{keyname}

# Normal SSH after boot -- connects as regular user via hostname
Host {hostname}-{username}
    HostName {hostname}
    User {username}
    IdentityFile ~/.ssh/{keyname}

# Normal SSH after boot -- connects as regular user via IP (fallback)
Host {hostname}-{username}-ip
    HostName {ip_address}
    User {username}
    IdentityFile ~/.ssh/{keyname}
```

Replace `{hostname}`, `{username}`, `{ip_address}`, and `{keyname}` with the appropriate values.

## Usage

### Unlocking the disk remotely after reboot

1. Reboot the server
2. Wait for the server to appear on the network (check router DHCP leases if unsure)
3. Connect using the initramfs SSH config entry:

    ```bash
    ssh {hostname}-init
    ```

4. Once connected, run the password agent to trigger the LUKS passphrase prompt:

    ```bash
    systemd-tty-ask-password-agent
    ```

5. Enter the LUKS passphrase when prompted. The connection will close automatically once the disk is unlocked and boot continues.

6. After boot completes, connect normally:

    ```bash
    ssh {hostname}-{username}
    ```

## Verification

Verify the initramfs contains the expected files after rebuilding:

```bash
sudo lsinitrd | grep 'authorized\|bin/sshd\|network/20'
```

Expected output should show three lines: the network config, the authorized keys file, and the sshd binary:

```bash
-rw-r--r--   1 root     root    [size] [date] etc/systemd/network/20-wired.network
-rw-------   1 root     root    [size] [date] root/.ssh/dracut_authorized_keys
-rwxr-xr-x   1 root     root    [size] [date] usr/bin/sshd
```

To verify the initramfs sshd config:

```bash
sudo lsinitrd -f etc/ssh/sshd_config
```

To verify the authorized key was included correctly:

```bash
sudo lsinitrd -f root/.ssh/dracut_authorized_keys
```

To verify the initramfs was rebuilt recently:

```bash
ls -lah /boot/initramfs-$(uname -r).img
```

## Notes

### Issues encountered during setup

**Root account locked with `!` in `/etc/shadow`**

This is documented in the dracut-sshd FAQ ([issue #30](https://github.com/gsauthof/dracut-sshd/issues/30)) under the question "Why do I get `Permission denied (publickey)` although the same authorized key works after the system is booted?"

By default on Fedora, the root account is locked using `!` in the shadow password field. The initramfs sshd config sets `UsePAM no`, and when PAM is disabled OpenSSH treats `!` as a hard account lock and refuses all authentication including public key auth. Changing to `*` resolves this. Both `!` and `*` are documented in `shadow(5)` as invalid password tokens (i.e. no valid password exists), but OpenSSH distinguishes between them when PAM is not involved. PAM itself delegates password validation to the underlying crypt library which treats both as invalid hashes and denies password auth regardless. The distinction is only meaningful to OpenSSH in the no-PAM context.

**SSH connection must be made as `root`, not as the normal user**

The initramfs environment only has a root user. Connecting as the normal username will result in `Permission denied` regardless of key configuration. This is expected behavior and is handled by the SSH client config entries above.

### Root account change implications

Changing the root shadow field from `!` to `*` means password authentication for root remains impossible (neither token is a valid password hash). The effect on public key authentication for root on the fully booted system with `PermitRootLogin prohibit-password` (the Fedora default) is not fully verified. In practice, key-based root login on the booted system did not appear to work as expected after this change, which is consistent with the expectation that `*` only affects behavior in the no-PAM initramfs context. Confirm behavior on the specific setup.

### Authorized keys file on unencrypted /boot

The `/root/.ssh/dracut_authorized_keys` file is included in the initramfs, which lives on the unencrypted `/boot` partition. This means the public key is readable by anyone with physical access to the machine. Public keys are not secret by design and an attacker who reads the public key gains nothing useful. This is standard territory for this type of setup.

### Security limitations

- **Initramfs tampering / evil maid**: Secure Boot validates the kernel but not the initramfs by default (the initramfs is generated locally and not signed). An attacker with physical access could replace the initramfs with a malicious one. Building a Unified Kernel Image (UKI) raises the bar by covering the initramfs with a Secure Boot signature, but does not fully close the attack surface. The closest available mitigation is TPM remote attestation, where a remote server holds secrets and will only release them if TPM measurements match a known-good boot state. This relies on the TPM's manufacturer-burned Endorsement Key as the root of trust, meaning the guarantee ultimately traces back to the chip manufacturer rather than the owner. There is no fully owner-controlled solution to evil maid attacks available on commodity hardware today.

## References

- [dracut-sshd GitHub repository](https://github.com/gsauthof/dracut-sshd)
- [dracut-sshd Fedora package](https://src.fedoraproject.org/rpms/dracut-sshd)
- [dracut-sshd issue #30 -- Permission denied with valid key](https://github.com/gsauthof/dracut-sshd/issues/30)
- [shadow(5) man page](https://man7.org/linux/man-pages/man5/shadow.5.html)
- [systemd-networkd documentation](https://www.freedesktop.org/software/systemd/man/systemd.network.html)
