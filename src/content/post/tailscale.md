---
layout: ../../layouts/post.astro
title: Setting up Tailscale VPN In My Home Network
description: In-depth guide to my experience setting up Tailscale within my home network
dateFormatted: Jun 11, 2026
topic: ["VPN", "Homelab"]
technologies: ["Tailscale", "RustDesk", "Proxmox"]
tags: ["Homelab", "VPN", "Tailscale", "RustDesk"]
hidden: false
image: https://tailscale.gallerycdn.vsassets.io/extensions/tailscale/vscode-tailscale/1.1.0/1759776117386/Microsoft.VisualStudio.Services.Icons.Default
---

![TAILSCALE](/assets/images/posts/tailscale.webp)

## About Tailscale

Tailscale utilizes a mesh VPN creating a private, encrypted network, what they call a *tailnet*. Tailscale spans all devices, regardless of the physical location. Tailscale employs Wireguard, a modern VPN protocol, which handles the encrypted transport between devices. In contrast to a traditional VPN where every packet is routed through a central concentrator, Tailscale devices are connected to eachother, forming a peer-to-peer mesh.

![Tailscale Mesh Topology](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/tailscalep2p.svg)

A coordination server distributes public keys and access policies allowing machines to find eachother and traverse NAT. 

**NOTE:** *The encrypted traffic itself flows between devices, and never touches Tailscale's infrastructure.*

As machines join the tailnet, authentication is handled through an identity provider (e.g., Google, Microsoft, Okta, GitHub, etc.) rather than sharing a static key or password, so access is tied to a user account that can be granted or revoked centrally. ACL policy is configurable to define which users & machines are allowed to reach which devices & ports.

As a result, the network behaves like a single LAN with capability to connect servers, laptops, VMs, cloud instances all within one accessible network, from anywhere in the world. 

## Initial Setup

Within Proxmox Virtual Environment, I created a linux container (LXC) as my inital tailscale node— *tailscaleprd1*. For the LXC specifications, I provisioned the LXC with 4 cores and 8GB RAM, and 8GB of disk space.

![LXC Specifications](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/tailscale_resources)

### TUN Access & Tailscale Interface Creation

By default, the unprivileged LXC doesnt expose the host's `/dev/net/tun` device, which is required for Tailscale to create the virtual interface for the VPN tunnel. By adding the following lines, this will grant permission to the TUN device, and mount the interface allowing Tailscale to reach it.

To do this, through Proxmoxs host node shell, use the following command to edit the .conf of the LXC:

```bash
root@pve1:~# nano /etc/pve/lxc/[PCT ID].conf
```

Add & save the following lines:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

![TUN](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/devnettun%20initial.png)
### DHCP Reservation 
Though not required, I configured a DHCP reservation within my router to reserve the IP address assigned to *tailscaleprd1*. 

Grabbing the machines IP: 
```bash
rhunt@tailscaleprd1:~$ ip a
2: eth0@if81: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether bc:24:11:54:92:5b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.196/24 brd 192.168.1.255 scope global dynamic eth0
        [REDACTED]
```

Configuring DHCP reservation within ExperienceOS:

![DHCP Reservation](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/reservingIP_tailscale.png)

### Install Tailscale

Now that TUN access is granted, and the tailscale interface is created, we can spin up tailscale in the *tailscaleprd1* node with one command:
**NOTE:** *This command is for Linux machines only.*

```bash
rhunt@tailscaleprd1:~$ curl -fsSL https://tailscale.com/install.sh | sh
```

![Tailscale Run](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/initial-tailscalesetup_node.png)

Once installation is complete, you can run the following command to invoke the tailscale process `tailscale up -ssh`

```bash
rhunt@tailscaleprd1:~$ tailscale up --ssh

To authenticate, visit
        https://login.tailscale.com/a/[REDACTED]
```

Visit the designated URL and authenticate through your desired Identity Provider (IdP). 

![Auth](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/auth)

Once authenticated, you will see a similar screen to connect a second device. 

![Tailscale-add](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/devices-tailscale)

In my case, I wanted VPN access to my homelab, specifically my Proxmox web GUI, so I deployed tailscale on my Proxmox host node— *pve1*. We can do this by repeating the same install command as before.

![tailscape-pveinstall](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/adding_tailscale_to_pve.png)

After authenticating the second device to the tailnet, you now have an encrypted WireGuard tunnel.

## Subnet Router vs. Adding Individual Devices

Initially, I kept repeating the process by adding more devices to the tailscale, however I realized this wasn't my goal— my goal is to have an encrypted VPN tunnel to my entire network, where all devices can commnunicate with eachother regardless of physical location.

After researching further into Tailscale capabilities, I came across [Tailscale Subnet Routers](https://tailscale.com/docs/features/subnet-routers), which provided what I was looking for. 

Per Tailscale documenation, Tailscale subnet routers allow you to extend your tailnet to include devices that don't or can't run the Tailscale client, acting as a gateway between your tailnet and physical subnets, enabling secure access to legacy devices, entire networks, or services without installing Tailscale everywhere. 

### Setting up Subnet Router within Tailnet

To achieve the subnet router functionality within my tailnet, you must enable IP forwarding within the kernel. To do so, you can execute the following commands in your initial tailscale node (in my case, *tailscaleprd1*).

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

![tailscale-subnetrouter-troubleshoot](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/troubleshot-subnetrouter)

After executing the IP forwarding settings to `etc/sysctl.d/99-tailscale.conf` and running `sysctl -p` to apply them returned a command not found error and a warning that IP forwarding is disbaled. This was due to `sysctl` residing in `/usr/sbin`, which wasnt included in the shells `$PATH`. To fix this, I executed the absolute path to apply the settings: 
```bash
/sbin/sysctl -p /etc/sysctl.d/99-tailscale.conf
```

To validate the output of `/etc/sysctl.d/99-tailscale.conf` and ensure the IP forwarding settings applied correctly, you can run the following commands, which should return a value of 1.

```bash
rhunt@tailscaleprd1:~$ cat /proc/sys/net/ipv4/ip_forward
cat /proc/sys/net/ipv6/conf/all/forwarding
1
1
```

After IP forwarding settings are set, you must advertise your subnet routes via the following command: 

`sudo tailscale set --advertise-routes=192.0.2.0/24,198.51.100.0/24`

**Make sure to replace the command with active subnets in your own environment.**

Once done, you can verify the status of tailscale through `tailscale status`.

Lastly, the IP route must be approved within the Tailscale Admin Center.

![approve-subnetroute](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/approve-route)

As the route is now approved, the tailnet is now advertising the subnet `192.168.1.0/24` network to the tailnet and forwards traffic to it. Devices within the tailnet can now reach hosts on that LAN as if they were local, regardless of the location of the physical device *witthout requiring Tailscale running themselves*.

### Custom DNS — AdGuard

A side benefit of routing traffic through the tailnet is that devices connected remotely also use my home DNS server, [AdGuard](https://adguard.com/kb/), for ad and tracker blocking — the same filtering that applies on the local network follows you wherever you connect from.

This is configured in the Tailscale Admin Center under **DNS → Custom Nameservers**, pointing to the LAN IP of the AdGuard instance.

### Accessing Devices via Mobile

Now that the tailnet can access any device on the network, I tested this out by installing and connecting to my tailnet via my phone, which I was able to access my Proxmox Web GUI.

<img src="https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/pve_from_mobile-tailscale.png" alt="pve-mobile" style="max-height: 500px; width: auto; display: block; margin: 0 auto;" />

## Windows RDP Workaround 

One of the features I was looking to gain was being able to control my Windows PC remotely, which is possible through Tailscale. However, this is not possible with Windows 11 Home Edition as it does not support incoming Remote Desktop Connections.

So, I explored some other solutions and stumbled upon [RustDesk](https://rustdesk.com/docs/en/), which is an open-source remote control alternative popular for self-hosting and security.


I installed the RustDesk .msi on both my Windows computers, and started getting into the security settings. 

### RustDesk Security Settings

As this needs to work remotely, the first thing I did was set a strong permanent unattended password and set up 2FA for any incoming connections.

![rustdesk-sec](https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/2fa-perm)

Additionally, I enabled Direct IP access over a port I specified to allow for connecting directly to the machine IP or tailnet assigned IP.

### Limiting IP Ranges - Firewall

Even though 2FA is enabled for any incoming connnection, I wanted a implicit deny for any incoming RustDesk connections from outside my 192.x network, and my 100.x tailscale network. To do this, I set up a Windows Firewall Rule via PowerShell using the following command:

```powershell
PS C:\Users\ryana> New-NetFirewallRule `
>>   -DisplayName "Allow RustDesk Direct IP from Tailscale and LAN" `
>>   -Direction Inbound `
>>   -Action Allow `
>>   -Protocol TCP `
>>   -LocalPort 21118 `
>>   -RemoteAddress 100.64.0.0/10,192.168.1.0/24
```

This rule effectively blocks any connection outside of the 100.x and 192.x networks.

To test this and ensure it worked, I connected to Tailscale on my phone, and RDP'd into my main PC, which was allowed through the firewall.

<img src="https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/tailscale%20working%20mobile.png" alt="tailscale working mobile" style="max-height: 500px; width: auto; display: block; margin: 0 auto;" />

And, to ensure it is blocking unknown incoming connections appropriately, I disconnected from the Tailscale VPN on my phone, disconnected from WiFi, and attempted the RustDesk connection, which was effectively blocked.

<img src="https://6fghcxgkmfuhb9vl.public.blob.vercel-storage.com/block%20access%20rustdesk%20working.png" alt="blockaccess" style="max-height: 500px; width: auto; display: block; margin: 0 auto;" />

### Closing Thoughts

Theres alot left to explore within Tailscale, but ultimately I'm really happy with setting up this VPN to access my home network, limit incoming RDP/RustDesk connections. 