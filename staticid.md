## Here's a complete guide to setting a static IP on Ubuntu Server. ##

---

## Find Your Network Interface Name

```bash
ip link show
# or
ip a
```

You'll see interface names like `eth0`, `ens33`, `enp0s3`, `enpXsY` — note yours down.

---

## Ubuntu 18.04+ — Netplan (Default Method)

Ubuntu Server uses **Netplan** for network configuration.

### 1. Find the Config File

```bash
ls /etc/netplan/
# Usually named: 00-installer-config.yaml or 01-netcfg.yaml
```

### 2. Edit the Config

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

### 3. Static IP Configuration

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:                          # ← Replace with your interface name
      dhcp4: false                  # Disable DHCP
      addresses:
        - 192.168.1.100/24          # ← Your desired static IP + subnet
      routes:
        - to: default
          via: 192.168.1.1          # ← Your gateway/router IP
      nameservers:
        addresses:
          - 8.8.8.8                 # Google DNS
          - 8.8.4.4                 # Google DNS secondary
          - 1.1.1.1                 # Cloudflare DNS
      dhcp6: false
```

### 4. Apply the Configuration

```bash
# Test first (reverts after 120s if something goes wrong)
sudo netplan try

# Apply permanently
sudo netplan apply
```

---

## Verify the Static IP

```bash
# Check IP address
ip a show ens33

# Check routing table
ip route show

# Check DNS
resolvectl status

# Test connectivity
ping -c 4 8.8.8.8
ping -c 4 google.com
```

---

## Multiple IPs on One Interface (Optional)

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.1.100/24          # Primary IP
        - 192.168.1.101/24          # Secondary IP
        - 10.0.0.50/24              # Additional network
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

---

## Multiple Network Interfaces (Optional)

Useful if your server has separate NICs for LAN and WAN:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:                          # Public / WAN interface
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
    ens36:                          # Private / LAN interface
      dhcp4: false
      addresses:
        - 10.0.0.10/24
```

---

## Common Issues & Fixes

### Wrong Permissions Warning
```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

### Revert if Something Breaks
```bash
# Netplan try auto-reverts after 120 seconds
# Or manually revert:
sudo netplan apply --debug
```

### Check for Errors
```bash
sudo netplan generate --debug
```

### Restart Network Service
```bash
sudo systemctl restart systemd-networkd
sudo systemctl status systemd-networkd
```

---

## Quick Reference

| Setting | Example Value | Description |
|---|---|---|
| Interface | `ens33` | Your NIC name from `ip a` |
| Static IP | `192.168.1.100/24` | IP + subnet mask |
| Gateway | `192.168.1.1` | Router IP |
| DNS Primary | `8.8.8.8` | Google DNS |
| DNS Secondary | `1.1.1.1` | Cloudflare DNS |
| CIDR /24 | `255.255.255.0` | 254 usable hosts |
| CIDR /16 | `255.255.0.0` | 65534 usable hosts |

---

**Tips:**
- Always use `netplan try` before `netplan apply` — it auto-reverts if you lose connectivity
- Keep a note of the old DHCP IP in case you need to revert
- Make sure your static IP is **outside your router's DHCP range** to avoid conflicts
- After changing SSH port + static IP, update your `~/.ssh/config` on your local machine