## About
Steps to connect Ethernet only devices, e.g. POE security cams, to home WiFi.

## Requirements
* A raspberry pi board running PiOS and connected to home WiFi
* Ethernet cable

## Steps
1. Install `dnsmasq` 
```
sudo apt update
sudo apt upgrade
sudo apt-get install dnsmasq
```
2. Setup `/etc/dhcpcd.conf`
```
sudo vi /etc/dhcpcd.conf
```
Go to Example static IP configuration section and append the following lines:

```
interface eth0
metric 333
static ip_address=192.168.10.1/24
static routers=192.168.10.0
```
NB: Metric should be higher than the default home WiFi gateway's metric (say if it was 303 then we'd use 333).

3. Restart `dhcpcd.service` and verify it started OK.
```
sudo systemctl restart dhcpcd.service
sudo systemctl status dhcpcd.service
```
Output should look something like:
```
dhcpcd.service - dhcpcd on all interfaces
   Loaded: loaded (/lib/systemd/system/dhcpcd.service; enabled; vendor preset: e
   Active: active (running) since Tue 2023-03-14 13:23:29 AEDT; 1 day 4h ago
  Process: 470 ExecStart=/usr/lib/dhcpcd5/dhcpcd -q -b (code=exited, status=0/SU
 Main PID: 525 (dhcpcd)
    Tasks: 2 (limit: 2059)
   CGroup: /system.slice/dhcpcd.service
           ├─525 /sbin/dhcpcd -q -b
           └─573 wpa_supplicant -B -c/etc/wpa_supplicant/wpa_supplicant.conf -iw

```

4. Save original `/etc/dnsmasq.conf`
```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
```

5. Create a new `/etc/dnsmasq.conf`
```
sudo vi /etc/dnsmasq.conf
```
Add the following lines:
```
interface=eth0       # Use interface eth0
listen-address=192.168.10.1   # Specify the address to listen on
bind-dynamic         # Bind to the interface
server=8.8.8.8       # Use Google DNS
domain-needed        # Don't forward short names
bogus-priv           # Drop the non-routed address spaces.
dhcp-range=192.168.10.50,192.168.10.150,12h # IP range and lease time
```

6. Enable IP forwarding (if not already done so):
```
sudo vi /etc/sysctl.conf
```
Then uncomment `#net.ipv4.ip_forward=1`
```
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

7. Setup `iptables` rules.\
  NB: Edit in correct camera IP addresses (ref `/var/lib/misc/dnsmasq.leases`). Then run the scripts to generate `iptables` rules and to save these in `/etc/iptables.ipv4.nat`.
```
chmod 755 00-wifi2eth-*
./00-wifi2eth-bridged-rules
./00-wifi2eth-save-my-iptables
```
8. Add saved `iptables` rules for load-up on reboot by editing `sudo vi /etc/rc.local` and adding before `exit 0` the following line `iptables-restore < /etc/iptables.ipv4.nat`

9. Re-start `dnsmasq`
```
sudo systemctl restart dnsmasq.service
sudo systemctl status dnsmasq.service
```
Output should look something like:
```
dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
   Loaded: loaded (/lib/systemd/system/dnsmasq.service; enabled; vendor preset: 
   Active: active (running) since Tue 2023-03-14 13:23:32 AEDT; 1 day 5h ago
  Process: 548 ExecStartPre=/usr/sbin/dnsmasq --test (code=exited, status=0/SUCC
  Process: 559 ExecStart=/etc/init.d/dnsmasq systemd-exec (code=exited, status=0
  Process: 588 ExecStartPost=/etc/init.d/dnsmasq systemd-start-resolvconf (code=
 Main PID: 587 (dnsmasq)
    Tasks: 1 (limit: 2059)
   CGroup: /system.slice/dnsmasq.service
           └─587 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -r /ru
```

10. The cameras specified in `00-wifi2eth-bridged-rules` should now be viewable from anywhere in home WiFi with their RTSP URLs e.g. `rtsp://rpi.local:2222/cam1_rtsp_string` and `rtsp://rpi.local:3333/cam2_rtsp_string` respectively.
