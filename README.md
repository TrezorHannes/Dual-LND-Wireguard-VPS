
# Dual-LND-Wireguard-VPS
_Connect your lightning network nodes via wireguard VPN Tunnel through your VPS to allow fast and anonymous payments. When finished, you'll be able to run one or more of your Lightning Nodes via Tor and obfuscate your Clearnet IP Adress via a paid VPS_

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/d/d5/Alter_Elbtunnel_%28St._Pauli%29.jpg/2880px-Alter_Elbtunnel_%28St._Pauli%29.jpg" alt="Alter Elbtunnel (erbaut 1911) – Wikipedia" width="320" height="240">

This is a fork of my own Guide [provided here](https://github.com/TrezorHannes/Dual-LND-Hybrid-VPS), but instead of OpenVPN, we're using the somewhat smaller / simpler WireGuard Solution.
The Problem statement remains the same, you may prefer one solution over the other. Have a read through both and see what fits better. But in either case, you're coming here for the following reasons:
- have a dynamic IP from your Internet Service Provider
- want to hide your home IP from the world, for whatever reason
- prefer to have your node's traffic completely hidden from your ISP
- desire to decrease your Lightning Node HTLC routing times, so instead of running Tor only, you want Clearnet availability, too
-   run more than one Lightning Node, and want both to leverage the same VPN tunnel
- are just curious and want to tinker around a bit, because it's good to have those skills when demand for experience continues to rise


## Table of Content

- [Pre-Amble](#pre-amble)
  - [Objective](#objective)
  - [Challenge](#challenge)
  - [Proposed Solution](#proposed-solution)
- [Pre-Reads](#pre-reads)
- [Pre-Requisites](#pre-requisites)
- [Preperations](#preperations)
  - [Make notes](#make-notes)
  - [Visualize](#visualize)
  - [Secure](#secure)
- [Let's get started (LFG!)](#lets-get-started-lfg)
  - [Lightning Node](#lightning-node)
  - [VPS: Setup](#vps-setup)
  - [VPS: Connect to your VPS and tighten it up](#vps-connect-to-your-vps-and-tighten-it-up)
  - [VPS: Install Wireguard](#vps-install-wireguard)
    - [VPS: Firewall](#vps-firewall)
    - [VPS: LND Port-Forwarding](#vps-lnd-port-forwarding)
    - [VPS: Start your WireGuard Server](#vps-start-your-wireguard-server)
- [Into the Tunnel](#into-the-tunnel)
  - [LND Node: Install and test the VPN Tunnel](#lnd-node-install-and-test-the-vpn-tunnel)
  - [LND Node: LND adjustments to listen and channel via VPS VPN Tunnel](#lnd-node-lnd-adjustments-to-listen-and-channel-via-vps-vpn-tunnel)
- [Appendix & FAQ](#appendix--faq)


## Pre-Amble


### Objective
Your own, non-custodial [Lightning-Network](https://github.com/lightningnetwork/lnd) Node(s) running in **Hybrid-Mode** (Tor and Clearnet) via a cheap, anonymous [Virtual Private Server (VPS)](https://www.webcentral.com.au/blog/what-does-vps-stand-for). This guide outlines setup for **two nodes** (adjust steps for one or more).

### Challenge
Fast, reliable, non-custodial Bitcoin payments should ideally be anonymous. Tor provides privacy but can be slow/unreliable.

### Proposed Solution
This guide offers **one approach** to improve speed and anonymity using a VPS and WireGuard. Follow carefully, it may take about 1 hour depending on your skills.


## Pre-Reads
To understand some concepts better, read these articles:
- [Hybrid-Mode for LND](https://github.com/blckbx/lnd-hybrid-mode)
- [Expose server behind NAT with WireGuard and a VPS](https://golb.hplar.ch/2019/01/expose-server-vpn.html)
- [How To Set Up WireGuard on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-22-04)



## Pre-Requisites
- Running `lnd-0.14.2-beta` or later (Umbrel, Raspiblitz, MyNode, RaspiBolt, or bare Raspi)
- Command-line familiarity
- SSH access to your node and VPS (e.g., [PuTTY](https://www.putty.org/) on Windows)
- VPS account with a public IP (DigitalOcean or alternatives)

[![DigitalOcean Referral Badge](https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%201.svg)](https://www.digitalocean.com/?refcode=5742b053ef6d&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)

_Disclaimer: ref link, $100 credit over 60 days, cheapest option ~$5/month._


## Preperations
The better we prepare, the more we can deal with blindspots and the unexpected.

### Make notes
Document your steps! Imagine future you setting up a new node - will you remember everything?

Suggested checklist:
- [ ] VPS external IP, VPS Tunnel IP, Node Tunnel IP
- [ ] Ports to forward
- [ ] ToDos
- [ ] Questions

### Visualize
Draw a diagram to understand the flow.
![High-lvl-Flowchart](Hybrid-LND-WG-VPN.drawio.png)

### Secure
Security is critical! This guide doesn't cover all security steps in detail. Start with small funds, stay updated on security news, and ideally, do this with a peer for review. Use 2FA/YubiKeys!

## Let's get started (LFG!)
Well, let's get into it, shall we?!

### Lightning Node
We will consider you have your **Lightning Node up and running**, connected via Tor and some funds on it. You also have SSH access to it and administrative privileges.

### VPS: Setup 
If you don't have a VPS, sign up via [referral link](https://m.do.co/c/5742b053ef6d) or choose another provider with static IP and reasonable cost (maybe even Lightning payable!).

For DigitalOcean, create a Droplet (takes minutes):
   - Add Droplet
   - OS: Ubuntu 20.04 (LTS) x64 (or your preference)
   - Plan: Basic, Shared CPU (upgrade later if needed)
   - CPU option: "Regular Intel with SSD" (~$6/month)
   - No extra volume needed
   - Datacenter region: your choice
   - Authentication: SSH keys (recommended, follow steps to add public keys. PuTTY-gen for Windows)
   - No backups, monitoring, IPv6 needed for this guide
   - Hostname: something memorable, e.g., `myLND-node-VPS`

After setup, you'll get a public IPv4 address. Note it down as `VPS Public IP: 207.154.241.101` (example).



### VPS: Connect to your VPS and tighten it up
Connect via SSH: `ssh root@207.154.241.101`. Harden your VPS immediately:
   - `apt update && apt upgrade`
   - [Add sudo user](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04) (e.g., `admin`), disable root password login.
   - Enable UFW (Uncomplicated Firewall):
```bash
$ apt install ufw
$ ufw default deny incoming
$ ufw default allow outgoing
$ ufw allow OpenSSH
$ ufw allow 9735 comment 'LND Main Node 1'
$ ufw enable
```
   - Follow [further hardening steps](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)
   - `sudo apt install fail2ban` (SSH protection)

### VPS: Install Wireguard
Follow [DigitalOcean guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-22-04) for detailed context. Skipping IPv6 for simplicity.

   - `sudo apt update && sudo apt install wireguard` # Install WireGuard
   - `wg genkey | sudo tee /etc/wireguard/private.key` # Generate private key
   - `sudo chmod go= /etc/wireguard/private.key` # Restrict private key permissions
   - `sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key` # Generate public key
   - Choose IP range `10.8.0.0/8`. VPS: `10.8.0.1`, Node: `10.8.0.2`
   - Edit `sudo nano /etc/wireguard/wg0.conf`:
```
[Interface]
PrivateKey = ***base64_encoded_private_key_goes_here***
Address = 10.8.0.1/24
ListenPort = 51820
SaveConfig = true
```
   - `CTRL+X`, `Y`, `Enter` to save.
   - `sudo nano /etc/sysctl.conf`, uncomment `net.ipv4.ip_forward=1`, `sudo sysctl -p` (enable forwarding).

   
#### VPS: Firewall
For packet forwarding, add rules to `wg0.conf`. This forwards packets from your node to the internet device.

   - Find internet device: `ip route list default` (e.g., `eth0`, `enps`). Note it down.
   - Edit `sudo nano /etc/wireguard/wg0.conf`, add after `SaveConfig = true`:

```bash
PostUp = ufw route allow in on wg0 out on eth0
PostUp = iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
PreDown = ufw route delete allow in on wg0 out on eth0
PreDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
   - `CTRL+X`, `Y`, `Enter` to save.

#### VPS: LND Port-Forwarding
Forward LND packets to your node.

   - Check LND port in `cat ~/.lnd/lnd.conf` (`listen=0.0.0.0:9735` assumed).
   - Add iptables forwarding and routing rules:
```bash
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -i eth0 -o wg0 -p tcp --syn --dport 9735 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o wg0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -i wg0 -o eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 9735 -j DNAT --to-destination 10.8.0.2
sudo iptables -t nat -A POSTROUTING -o wg0 -p tcp --dport 9735 -d 10.8.0.2 -j SNAT --to-source 10.8.0.1
```
   *Adjust ports for multiple nodes.*

   - **Allow SSH and WireGuard UDP port in UFW**:
     - `sudo ufw allow 51820/udp`
     - `sudo ufw allow OpenSSH`
   - **(Best Practice) Limit SSH access to your home IP**:
     - `sudo ufw allow from 185.111.222.0/24 proto tcp to any port 22 comment 'SSH from Home'` (example, replace with your IP).
     - Test SSH login in another terminal to avoid lockout!
   - Refresh UFW: `sudo ufw disable && sudo ufw enable`, check status: `sudo ufw status verbose`

   - **Make iptables rules persistent**:
```bash
# Create a script to save and restore the iptables rules
sudo nano /etc/wireguard/iptables-save.sh
```

Add these lines to the script:
```bash
#!/bin/bash
# Save current iptables rules
iptables-save > /etc/wireguard/iptables.rules
ip6tables-save > /etc/wireguard/ip6tables.rules
```

Make it executable:
```bash
sudo chmod +x /etc/wireguard/iptables-save.sh
```

Now create a service to restore rules at boot:
```bash
sudo nano /etc/systemd/system/iptables-restore.service
```

Add this content:
```
[Unit]
Description=Restore iptables rules
After=network.target
Before=wg-quick@wg0.service
[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/wireguard/iptables.rules
ExecStart=/sbin/ip6tables-restore /etc/wireguard/ip6tables.rules
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
```
Now save your current rules and enable the service:
```bash
sudo /etc/wireguard/iptables-save.sh
sudo systemctl enable iptables-restore.service
```

This approach preserves your UFW configuration while ensuring your custom iptables rules for WireGuard are applied after a reboot.


#### VPS: Start your WireGuard Server
   - `sudo systemctl enable wg-quick@wg0.service` # Enable as service
   - `sudo systemctl start wg-quick@wg0.service` # Start service
   - `sudo systemctl status wg-quick@wg0.service` # Check status

Your WireGuard server is running! Internet can connect to VPS on ports 9735 and 22 (SSH from home), tunnel on port 51820.

Check your notes for:
   - [ ] VPS Wireguard IP (`ip address`, look for `wg0`)
   - [ ] Port from `wg0.conf` (`51820` if default)
   - [ ] Wireguard Server Public Key





## Into the Tunnel
We have installed the tunnel through the mountain, but need to get our LND Node to use it.

### LND Node: Install and test the VPN Tunnel
Switch to your **Lightning Node** terminal. Replicate VPS WireGuard setup to connect to the VPS tunnel.

```bash
$ sudo apt update
$ sudo apt install wireguard -y
$ sudo apt install resolvconf -y # for DNS via tunnel
$ wg genkey | sudo tee /etc/wireguard/private.key
$ sudo chmod go= /etc/wireguard/private.key
$ sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
# Note down private key (secret!) and public key (for server config).
```

Create `wg0.conf` on your node: `sudo nano /etc/wireguard/wg0.conf`

```
[Interface]
PrivateKey = ***base64_encoded_peer_private_key_goes_here***
Address = 10.8.0.2/24

[Peer]
PublicKey = U9uE2kb/nrrzsEU58GD3pKFU3TLYDMCbetIsnV8eeFE=
AllowedIPs = 0.0.0.0/0
Endpoint = 207.154.241.101:51820
PersistentKeepalive = 25
```


**Important**: Route all node traffic through the tunnel, but keep local LAN access.

   - `ip route list table main default` (find node's internet gateway IP and device, e.g., `203.0.113.1 eth0`)
   - `ip -brief address show eth0` (find node's IP, e.g., `203.0.113.5` or `192.168.1.20`)
   - `resolvectl dns eth0` (VPS DNS servers, e.g., `67.207.67.2 67.207.67.3` from VPS terminal)
   - Edit `sudo nano /etc/wireguard/wg0.conf`, add before `[Peer]`:


```
PostUp = ip rule add table 200 from 203.0.113.5
PostUp = ip route add table 200 default via 203.0.113.1
PreDown = ip rule delete table 200 from 203.0.113.5
PreDown = ip route delete table 200 default via 203.0.113.1

DNS = 67.207.67.2 67.207.67.3
```
   - `sudo cat /etc/wireguard/public.key` # Note down node's public key.

Now, switch to **VPS terminal** to allow node connection.

   - `sudo wg set wg0 peer NodePublicKey allowed-ips 10.8.0.2` (replace `NodePublicKey`)
   - `sudo wg` (verify settings)

**Warning**: Next step reroutes node traffic through tunnel, potential downtime for LND. Be patient!

   - `sudo wg-quick up wg0` (activate tunnel temporarily)
   - `sudo wg` (check status on both Node and VPS terminals, handshake and traffic?)
   - `ip route get 1.1.1.1` (check DNS resolving on node)
   - `sudo wg-quick down wg0` (deactivate tunnel)
   - For permanent tunnel: `sudo wg-quick down wg0`, `sudo systemctl enable wg-quick@wg0.service`, `sudo systemctl start wg-quick@wg0.service`, `sudo systemctl status wg-quick@wg0.service`

Tunnel established! Troubleshoot with `sudo wg show` or `sudo systemctl journal -u wg-quick@wg0.service`.


### LND Node: LND adjustments to listen and channel via VPS VPN Tunnel
We switch Terminal windows again, going back to your LND Node. A quick disclaimer again, since we are fortunate enough to have plenty of good LND node solutions out there, we cannot cater for every configuration out there. Feel free to leave comments or log issues if you get stuck for your node, we'll be looking at the two most different setups here. But this should work very similar on _MyNode_, _Raspibolt_ or _Citadel_.

Be very cautious with your `lnd.conf`. Make a backup before with `cp ~/.lnd/lnd.conf ~/.lnd/lnd.bak` so you can revert back when things don't work out. 
The brackets below indicate the section where each line needs to be added to. Don't place anything anywhere else, as it will cause your LND constrain from starting properly.

_Adjust ports and IPs accordingly!_

<details><summary>Click here to expand  Raspibolt settings</summary>
<p>

   LND.conf adjustments, open with `sudo nano /mnt/hdd/lnd/lnd.conf`

[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735`           | # to add your VPS Public-IP |
   | `nat=false`                                 | # deactivate NAT |

[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                           | # ensure Tor is active |
   | `tor.v3=true`                               | # with the latest version. v2 is going to be deprecated this summer |
   | `tor.streamisolation=false`                 | # this needs to be false, otherwise hybrid mode doesn't work |
   | `tor.skip-proxy-for-clearnet-targets=true`  | # activate hybrid mode |

`CTRL-X` => `Yes` => `Enter` to save

LND Systemd Startup adjustment

   | Command | Description |
   | --- | --- |
   | `sudo systemctl restart lnd.service` | apply changes and restart your lnd.service. It will ask you to reload the systemd services, copy the command, and run it with sudo. This can take a while, depends how long your last restart was. Be patient. | 
   | `sudo tail -n 30 -f /mnt/hdd/lnd/logs/bitcoin/mainnet/lnd.log` | to check whether LND is restarting properly |
   | `lncli getinfo` | to validate that your node is now online with two uris, your pub-id@VPS-IP and pub-id@Tor-onion |

   ```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9736",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```        
</p>
</details>

<details><summary>Click here to expand Raspiblitz 1.7.x settings</summary>
<p>

   LND.conf adjustments, open with `sudo nano /mnt/hdd/lnd/lnd.conf`

[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735`           | # to add your VPS Public-IP |
   | `nat=false`                                 | # deactivate NAT |

[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                           | # ensure Tor is active |
   | `tor.v3=true`                               | # with the latest version. v2 is going to be deprecated this summer |
   | `tor.streamisolation=false`                 | # this needs to be false, otherwise hybrid mode doesn't work |
   | `tor.skip-proxy-for-clearnet-targets=true`  | # activate hybrid mode |

`CTRL-X` => `Yes` => `Enter` to save
  
RASPIBLITZ CONFIG FILE
`sudo nano /mnt/hdd/raspiblitz.conf` since Raspiblitz has some LND pre-check scripts which otherwise overwrite your settings.
   | Command | Description |
   | --- | --- |
   | `publicIP='207.154.241.101'`                | # add your VPS Public-IP |
   | `lndPort='9735'`                            | # define the LND port |
   | `lndAddress='207.154.241.101'`              | # define your LND public IP address |

`CTRL-X` => `Yes` => `Enter` to save

LND Systemd Startup adjustment

   | Command | Description |
   | --- | --- |
   | `sudo systemctl restart lnd.service` | apply changes and restart your lnd.service. It will ask you to reload the systemd services, copy the command, and run it with sudo. This can take a while, depends how long your last restart was. Be patient. | 
   | `sudo tail -n 30 -f /mnt/hdd/lnd/logs/bitcoin/mainnet/lnd.log` | to check whether LND is restarting properly |
   | `lncli getinfo` | to validate that your node is now online with two uris, your pub-id@VPS-IP and pub-id@Tor-onion |

   ```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9736",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```        
</p>
</details>

<details><summary>Click here to expand Raspiblitz 1.8.x settings</summary>
<p>

   LND.conf adjustments, open with `sudo nano /mnt/hdd/lnd/lnd.conf`

[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735`           | # to add your VPS Public-IP |
   | `nat=false`                                 | # deactivate NAT |

[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                           | # ensure Tor is active |
   | `tor.v3=true`                               | # with the latest version. v2 is going to be deprecated this summer |
   | `tor.streamisolation=false`                 | # this needs to be false, otherwise hybrid mode doesn't work |
   | `tor.skip-proxy-for-clearnet-targets=true`  | # activate hybrid mode |

`CTRL-X` => `Yes` => `Enter` to save
  
RASPIBLITZ LND-checkup FILE
`sudo nano /home/admin/config.scripts/lnd.check.sh` since Raspiblitz has some LND pre-check scripts which otherwise overwrite your settings. Go to line 184 or search for `enforce PublicIP if (if not running Tor)`. Uncomment those 5 lines indicated here:

```
#  if [ "${runBehindTor}" != "on" ]; then
#    setting ${lndConfFile} ${insertLine} "externalip" "${publicIP}:${lndPort}"
#  else
    # when running Tor a public ip can make startup problems - so remove
#    sed -i '/^externalip=*/d' ${lndConfFile}
#  fi
```

`CTRL-X` => `Yes` => `Enter` to save

LND Systemd Startup adjustment

   | Command | Description |
   | --- | --- |
   | `sudo systemctl restart lnd.service` | apply changes and restart your lnd.service. It will ask you to reload the systemd services, copy the command, and run it with sudo. This can take a while, depends how long your last restart was. Be patient. | 
   | `sudo tail -n 30 -f /mnt/hdd/lnd/logs/bitcoin/mainnet/lnd.log` | to check whether LND is restarting properly |
   | `lncli getinfo` | to validate that your node is now online with two uris, your pub-id@VPS-IP and pub-id@Tor-onion |

   ```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9736",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```        
</p>
</details>

<details><summary>Click here to expand Umbrel Pre-0.5 & Citadel settings</summary>
<p>
<!-- Add further comments for Umbrel and validate how to adjust starting LND docker for 0.5 with those changes, and making them persistent -->

 LND.conf adjustments, open with `sudo nano /home/umbrel/umbrel/lnd/lnd.conf`

   
[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735` | # to add your VPS Public-IP | 
   | `nat=false`                       | # deactivate NAT | 

[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                          | # ensure Tor is active | 
   | `tor.v3=true`                              | # with the latest version. v2 is going to be deprecated this summer | 
   | `tor.streamisolation=false`                | # this needs to be false, otherwise hybrid mode doesn't work | 
   | `tor.skip-proxy-for-clearnet-targets=true` | # activate hybrid mode | 

`CTRL-X` => `Yes` => `Enter` to save

LND Restart to incorporate changes to `lnd.conf`
   | Command | Description |
   | --- | --- |
   | `cd umbrel && docker-compose restart lnd` | This can take a while. Be patient. |
   | `tail -n 30 -f ~/umbrel/lnd/logs/bitcoin/mainnet/lnd.log` | check whether LND is restarting properly |  
   | `~/umbrel/bin/lncli getinfo` | validate that your node is now online with two uris, your pub-id@VPS-IP and pub-id@Tor-onion |
```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9736",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```
</p>
</details>

<details><summary>Click here to expand Umbrel Version 0.5.x  settings</summary>
<p>
<!-- Add further comments for Umbrel and validate how to adjust starting LND docker for 0.5 with those changes, and making them persistent -->


 LND.conf adjustments, open with `sudo nano /home/umbrel/umbrel/lnd/lnd.conf`

   
[**Application Options**]
   | Command | Description |
   | --- | --- |
   | `externalip=207.154.241.101:9735` | # to add your VPS Public-IP | 
   | `nat=false`                       | # deactivate NAT | 


[**tor**]
   | Command | Description |
   | --- | --- |
   | `tor.active=true`                          | # ensure Tor is active | 
   | `tor.v3=true`                              | # with the latest version. v2 is going to be deprecated this summer | 
   | `tor.streamisolation=false`                | # this needs to be false, otherwise hybrid mode doesn't work | 
   | `tor.skip-proxy-for-clearnet-targets=true` | # activate hybrid mode | 

`CTRL-X` => `Yes` => `Enter` to save

LND Restart to incorporate changes to `lnd.conf`
  | Command | Description |
   | --- | --- |
   | `~/umbrel/scripts/app stop lightning && ~/umbrel/scripts/app start lightning` |  same applies here: Be patient. |  
   | `tail -f ~/umbrel/app-data/lightning/data/lnd/logs/bitcoin/mainnet/lnd.log` | Check the logs |  
   | `~/umbrel/scripts/app compose lightning exec lnd lncli getinfo` | Check the two Uris looking like below |   

  ```
"03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@207.154.241.101:9736",
        "03502e39bb6ebfacf4457da9ef84cf727fbfa37efc7cd255b088de426aa7ccb004@vsryyejeizfx4vylexg3qvbtwlecbbtdgh6cka72gnzv5tnvshypyvqd.onion:9735"
```

</p>
</details>

### Celebrate and wrapper
Now the moment of truth: once you tested the reboot, checked the LND log, and `lncli getinfo` shows you both the Tor and the VPS Clearnet IP as uris, you're done. `curl https://api.ipify.org` responds with your VPS Clearnet-IP, too. LN gossip will soon populate your IP offering, and aggregator sites like Amboss or 1ml will pick it up. Time to celebrate 🍻 
or troubleshoot where things could have gone wrong. If the former: Congratulations - you made it!

Hope you enjoyed this article. Please do share feedback and suggestions for improvement.
If this guide was of any help, I'd appreciate if you share the article with others, give me a follow on X [![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/HandsdownI.svg?style=social&label=Follow%20%40HodlmeTight1337)](https://twitter.com/HodlmeTight1337) or [nostr](https://njump.me/npub1ch25m5lkk8kfepr63f0jnpd9te8l9f585pfpr2g2ma4pre9rmlrqlu0yjy), perhaps even donating some sats to [hakuna@hodlmetight.com](https://lnbits.hodlmetight.org/tipjar/1)

I'm also always grateful for incoming Hybrid channels to my node: [HODLmeTight](https://amboss.space/node/037f66e84e38fc2787d578599dfe1fcb7b71f9de4fb1e453c5ab85c05f5ce8c2e3)


## Appendix & FAQ

#### I'm stuck and have no idea why it's not working. Who can help?
Please add an issue on Github with your question and provide as much detail as possible. Keep it safe though, no macaroon or user-ids! Before that, use a [port-checker tool](https://portchecker.co/) or [Tunnel⚡Sats Pingbot](http://t.me/TunnelSatsBot) to check your connection

#### Why DigitalOcean - can't we pick a VPS where we can pay with Lightning, and anonymously
Consider this guide a work-in-progress. I've picked DigitalOcean since I know what I'm doing there. Heard good things about [Luna Node](https://www.lunanode.com/), it's cheaper and you can pay with sats, so will test this out next. Also happy to add further alternatives, leave comments if you think these can accomplish the same results. Fee free to provide suggestions here.

#### Can I add more nodes connecting to the tunnel? If so, how?
In fact, I have more than one node connected to the tunnel. You need to handle your port-forwarding appropriately, since every node needs their unique LND listen port. Eg Node 1 has 9735, Node 2 9736 and so on. IPtable rules and UFW needs to be adjusted. But once you got this guide internalised, the principle should be clear. Otherwise, let me know.
