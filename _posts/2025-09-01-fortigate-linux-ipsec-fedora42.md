---
title: "FortiGate Removed SSL VPN — How I Got Linux Clients Working with IPsec on Fedora 42"
date: 2025-09-01 17:55:00 +0400
categories: [Networking, VPN]
tags: [fortigate, fortios, linux, fedora, strongswan, ikev2, ipsec, vpn]
author: osama
description: FortiOS 7.4.8 removed SSL VPN on some FortiGate models. Here is a working IKEv2 + strongSwan setup for Fedora 42 using swanctl.
image:
  path: /assets/img/posts/fortigate-linux-ipsec.png
  alt: FortiGate and Linux strongSwan
pin: false
canonical_url: https://calisamaa.medium.com/fortigate-removed-ssl-vpn-heres-how-i-got-linux-clients-working-with-ipsec-on-fedora-42-493ab39c1a0d
---

When **FortiOS 7.4.8** shipped, SSL VPN was removed on some FortiGate models. Remote access broke for users who relied on **SSL VPN tunnel mode** with FortiClient on Linux.

## The problem

- Windows clients used **IPsec** with FortiClient.
- Linux clients (e.g., Fedora) used **SSL VPN**, since FortiClient for Linux lacks IPsec.
- After upgrading to **7.4.8**, SSL VPN was removed on the firewall, so FortiClient on Linux stopped working.
- There is no official Fortinet IPsec client for Linux.

I tested, searched docs and forums, and built a working **IKEv2 dial‑up IPsec** solution using **strongSwan 5.9.14** on **Fedora 42**.

---

## Working setup: FortiGate + strongSwan (IKEv2, swanctl)

### FortiGate (Phase 1 — dial‑up IPsec)

```shell
config vpn ipsec phase1-interface
    edit "dialup-IPSEC1"
        set type dynamic
        set interface "WAN1"
        set ike-version 2
        set local-gw <FORTIGATE_PUBLIC_IP>
        set authmethod psk
        set peertype any
        set mode-cfg enable
        set proposal aes256-sha256
        set dhgrp 14
        set transport udp
        set nattraversal enable
        set fragmentation enable
        set ip-fragmentation post-encapsulation
        set assign-ip enable
        set assign-ip-from name
        set ipv4-name "IPSEC-VPN-ADDRESS"
        set ipv4-netmask 255.255.255.255
        set dns-mode auto
        set ipv4-split-include "all"
        set psksecret ENC <SECRET PSK>
    next
end
```

### Fedora client: packages and services

```bash
sudo dnf install -y strongswan-swanctl
sudo mkdir -p /etc/strongswan/swanctl
sudo nano /etc/strongswan/swanctl/swanctl.conf   # paste the config below
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo systemctl enable --now strongswan-swanctl
sudo swanctl --load-all
sudo swanctl --initiate --child dialup-cubes
```

---

## Option 1: Shared PSK (simple)

**Pros:** simple, quick, high performance.  
**Cons:** shared secret for all clients, no per‑user audit.

Create `/etc/strongswan/swanctl/swanctl.conf`:

```ini
connections {
    dialup-cubes {
        version = 2
        remote_addrs = 1.1.1.1   # FortiGate public IP
        local {
            auth = psk
            id = @fedora-client
        }
        remote {
            auth = psk
            id = 1.1.1.1          # FortiGate public IP
        }
        children {
            dialup-cubes {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                esp_proposals = aes256-sha256-modp2048
            }
        }
        pools = ipsec_pool
        proposals = aes256-sha256-modp2048
        send_certreq = no
        encap = no               # disable NAT‑T if client has public IP
        dpd_delay = 60s
        rekey_time = 86400s
        send_cert = never
        unique = never
        vips = 0.0.0.0           # request virtual IP via mode‑cfg
    }
}

pools {
    ipsec_pool {
        # Dummy pool — FortiGate assigns the VIP
        addrs = 0.0.0.0/0
    }
}

secrets {
    ike-1 {
        secret = "<PSK>"         # must match FortiGate
    }
}
```

**Why it works**

- FortiGate `mode-cfg` + `assign-ip` for dynamic clients.
- Matching IDs and proposals (AES256/SHA256/DH14).
- MTU handled by `post-encapsulation` fragmentation.
- NAT‑T disabled if the client is on a public IP.

---

## Option 2: Per‑user auth

### Method A: EAP‑MSCHAPv2 (IKEv2)

Works with swanctl. Supports username/password. Can back to RADIUS/LDAP.

```ini
connections {
    dialup-cubes {
        version = 2
        remote_addrs = 1.1.1.1
        local {
            auth = eap-mschapv2
            id = osama
            eap_id = osama
        }
        remote {
            auth = pubkey
        }
        children {
            dialup-cubes {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                esp_proposals = aes256-sha256-modp2048
            }
        }
        proposals = aes256-sha256-modp2048
        dpd_delay = 60s
        rekey_time = 86400s
        vips = 0.0.0.0
        send_certreq = yes
        encap = no
        unique = never
    }
}

secrets {
    eap-osama {
        id = osama
        secret = "secure-user-password"
    }
}
```

> FortiGate must allow EAP and present a valid certificate. If using RADIUS, create users there.

### Method B: XAuth (legacy, IKEv1)

Use only if required by old setups. Needs `ipsec.conf` instead of swanctl and is not recommended.

---

## Notes

- Tested on **Fedora 42** with **strongSwan 5.9.14**.
- Replace placeholders such as `<FORTIGATE_PUBLIC_IP>` and `<PSK>`.
- If the client is behind NAT, set `encap = yes` and ensure UDP/500 and UDP/4500 are open.
- For split‑tunnel, narrow `remote_ts` to required subnets.

---

If you want an Ubuntu version, scripts, or hardening steps, contact me.
