# End Hosts

## PC1 (VPCS)

```
set pcname PC1
ip 192.168.10.11 192.168.10.1 24
ip dns 8.8.8.8
save
```

## AUTO-HOST (Ubuntu 22.04 automation host)

Static addressing via netplan - `/etc/netplan/01-lab.yaml` (permissions `600`):

```yaml
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: no
      addresses: [192.168.10.20/24]
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Applied with `netplan apply`.

SSH client settings for the lab devices - `~/.ssh/config` (permissions `600`).
The older Cisco images only offer SHA1 key exchange, `ssh-rsa` host keys and CBC ciphers,
which Ubuntu 22.04's OpenSSH disables by default. Legacy algorithms are re-enabled **only**
for the lab subnet (see Issue #5 in `docs/troubleshooting-log.md`):

```
Host 192.168.10.*
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    Ciphers +aes256-cbc
```

Installed tooling:

```bash
apt update
apt install -y python3-venv python3-pip git curl
```

Root disk grown from 2.2 GB to 10 GB (see Issue #4).
