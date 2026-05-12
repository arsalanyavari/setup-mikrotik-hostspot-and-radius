# how to setup hotspot and radius on mikrotik CHR

This guide shows how to set up MikroTik CHR on UTM, configure a basic Hotspot, and then integrate the Hotspot with MikroTik User Manager as an internal RADIUS server.

Final topology:

```text
Internet / UTM Shared Network
        |
      ether1
   MikroTik CHR
      ether2
        |
   Ubuntu VM / Hotspot Client
```

`ether1` is used for WAN / internet / management.

`ether2` is used for LAN / Hotspot users.

---

## 1. Download and install UTM

Download UTM for macOS:

```text
https://mac.getutm.app/
```

Install UTM normally on macOS.

---

## 2. Download MikroTik CHR image for UTM

For this setup, I used MikroTik CHR from:

```text
https://github.com/tikoci/mikropkl/releases/tag/chr-7.22.1
```

Download the UTM image, for example:

```text
chr.aarch64.qemu.7.22.1.utm
```

Open/import it in UTM.

---

## 3. Prepare the virtual bridge on macOS

Create a local bridge on macOS. This bridge works as the virtual switch between MikroTik `ether2` and the Ubuntu VM.

Run this on macOS Terminal if `bridge0` doesnt exist:

```bash
sudo ifconfig bridge0 create
sudo ifconfig bridge0 up
```

Check it:

```bash
ifconfig bridge0
```

> [!note]
> If `bridge0` already exists, do not create it again. Just make sure it is up:
>
> ```bash
> sudo ifconfig bridge0 up
> ifconfig bridge0
> ```

---

## 4. Configure MikroTik CHR network adapters in UTM

MikroTik CHR must have two network adapters.

### Network 1

```text
Network Mode = Shared Network
```

This becomes `ether1` inside MikroTik.

Purpose:

```text
WAN / internet / management
```

### Network 2

```text
Network Mode = Bridged
Bridge Interface = bridge0
```

This becomes `ether2` inside MikroTik.

Purpose:

```text
LAN / Hotspot users
```

> [!note]
> Do not use `Emulated VLAN` for this setup. In my test, UTM could run its own DHCP on that network, so Ubuntu received an IP from UTM instead of MikroTik.

---

## 5. Configure Ubuntu VM network in UTM

Use only one network adapter for the Ubuntu VM.

```text
Network Mode = Bridged
Bridge Interface = bridge0
```

This puts Ubuntu in the same Layer 2 network as MikroTik `ether2`.

---

## 6. Boot MikroTik CHR and set admin password

Start the MikroTik CHR VM.

Login with:

```text
username: admin
password: empty by default
```

Set a new password when prompted.

---

## 7. Set MikroTik hostname

Command:

```
/system identity set name=CHR-Hotspot
```

Why:

This changes the router prompt/name to something easier to identify.

WebFig / WinBox equivalent:

```text
System → Identity
Name: CHR-Hotspot
Apply / OK
```

---

## 8. Check interfaces

Command:

```
/interface print
```

Expected interfaces:

```text
ether1
ether2
lo
```

In this setup:

```text
ether1 = WAN / internet / management
ether2 = LAN / Hotspot users
```

WebFig / WinBox equivalent:

```text
Interfaces
```

---

## 9. Check internet connectivity from MikroTik

Commands:

```
/ping 1.1.1.1 count=2
/ping google.com count=2
```

Why:

`1.1.1.1` checks raw internet connectivity.

`google.com` checks both internet and DNS.

If `1.1.1.1` works but `google.com` fails, DNS is not working.

---

## 10. Configure MikroTik DNS

Command:

```
/ip dns set allow-remote-requests=yes servers=1.1.1.1,8.8.8.8
```

Why:

MikroTik uses `1.1.1.1` and `8.8.8.8` as upstream DNS servers.

`allow-remote-requests=yes` allows Hotspot clients to use MikroTik as their DNS server.

WebFig / WinBox equivalent:

```text
IP → DNS
Servers: 1.1.1.1, 8.8.8.8
Allow Remote Requests: checked
Apply / OK
```

Check:

```
/ip dns print
```

---

## 11. Set IP address on ether2

Command:

```
/ip address add address=192.168.88.1/24 interface=ether2 comment="Hotspot LAN Gateway"
```

Why:

This makes MikroTik the gateway for Hotspot clients.

The Hotspot LAN will be:

```text
192.168.88.0/24
```

MikroTik will be:

```text
192.168.88.1
```

WebFig / WinBox equivalent:

```text
IP → Addresses
Add New
Address: 192.168.88.1/24
Interface: ether2
Comment: Hotspot LAN Gateway
Apply / OK
```

Check:

```
/ip address print
```

---

## 12. Create IP pool for Hotspot clients

Command:

```
/ip pool add name=hs-pool ranges=192.168.88.10-192.168.88.249
```

Why:

This pool contains the IP addresses that can be assigned to Hotspot clients.

WebFig / WinBox equivalent:

```text
IP → Pool
Add New
Name: hs-pool
Addresses: 192.168.88.10-192.168.88.249
Apply / OK
```

Check:

```
/ip pool print
```

---

## 13. Create DHCP server on ether2

Command:

```
/ip dhcp-server add name=dhcp-hotspot interface=ether2 address-pool=hs-pool lease-time=1h disabled=no
```

Why:

This DHCP server gives IP addresses to clients connected to the Hotspot LAN.

WebFig / WinBox equivalent:

```text
IP → DHCP Server
DHCP tab → Add New
Name: dhcp-hotspot
Interface: ether2
Address Pool: hs-pool
Lease Time: 01:00:00
Disabled: unchecked
Apply / OK
```

Then create the DHCP network:

```
/ip dhcp-server network add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1
```

Why:

This tells DHCP clients:

```text
Gateway = 192.168.88.1
DNS = 192.168.88.1
```

WebFig / WinBox equivalent:

```text
IP → DHCP Server → Networks tab
Add New
Address: 192.168.88.0/24
Gateway: 192.168.88.1
DNS Servers: 192.168.88.1
Apply / OK
```

Check:

```
/ip dhcp-server print
/ip dhcp-server network print
/ip dhcp-server lease print
```

---

## 14. Create NAT for Hotspot clients

Command:

```
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade comment="NAT Hotspot LAN to WAN"
```

Why:

Hotspot clients have private IP addresses such as `192.168.88.x`.

NAT allows those clients to access the internet through MikroTik `ether1`.

WebFig / WinBox equivalent:

```text
IP → Firewall → NAT
Add New
Chain: srcnat
Out Interface: ether1
Action: masquerade
Comment: NAT Hotspot LAN to WAN
Apply / OK
```

Check:

```
/ip firewall nat print
```

---

## 15. Create DNS record for the Hotspot login page

Command:

```
/ip dns static add name=hotspot.lan address=192.168.88.1 ttl=1h
```

Why:

This makes this name resolve locally:

```text
hotspot.lan → 192.168.88.1
```

So the Hotspot login page can be opened as:

```text
http://hotspot.lan/login
```

WebFig / WinBox equivalent:

```text
IP → DNS → Static
Add New
Name: hotspot.lan
Address: 192.168.88.1
Apply / OK
```

Check:

```
/ip dns static print
```

> [!note]
> I used `hotspot.lan` instead of `hotspot.local`. The `.local` suffix can conflict with mDNS/Avahi on Linux clients.

---

## 16. Create Hotspot server profile

Command:

```
/ip hotspot profile add name=hsprof1 hotspot-address=192.168.88.1 dns-name=hotspot.lan html-directory=hotspot login-by=http-chap,http-pap,cookie
```

Why:

This profile defines the general behavior of the Hotspot service.

It controls:

```text
Hotspot address
DNS name
Login methods
HTML directory for the login page
```

WebFig / WinBox equivalent:

```text
IP → Hotspot → Server Profiles
Add New
Name: hsprof1
Hotspot Address: 192.168.88.1
DNS Name: hotspot.lan
HTML Directory: hotspot
Login By: http-chap, http-pap, cookie
Apply / OK
```

Check:

```
/ip hotspot profile print detail where name=hsprof1
```

---

## 17. Create Hotspot server on ether2

Command:

```
/ip hotspot add name=hotspot1 interface=ether2 address-pool=hs-pool profile=hsprof1 disabled=no
```

Why:

This enables the actual Hotspot service on `ether2`.

`hsprof1` is the Hotspot server profile.

`hotspot1` is the actual Hotspot server running on `ether2`.

WebFig / WinBox equivalent:

```text
IP → Hotspot → Servers
Add New
Name: hotspot1
Interface: ether2
Address Pool: hs-pool
Profile: hsprof1
Disabled: unchecked
Apply / OK
```

Check:

```
/ip hotspot print
```

---

## 18. Create local Hotspot user profile

Command:

```
/ip hotspot user profile add name=hs-user-profile shared-users=1 rate-limit=2M/2M
```

Why:

This profile defines user-level limits.

In this example:

```text
shared-users=1
```

means the same user can only be logged in from one device at a time.

```text
rate-limit=2M/2M
```

limits the user speed.

WebFig / WinBox equivalent:

```text
IP → Hotspot → User Profiles
Add New
Name: hs-user-profile
Shared Users: 1
Rate Limit: 2M/2M
Apply / OK
```

Check:

```
/ip hotspot user profile print
```

---

## 19. Create local Hotspot test user

Command:

```
/ip hotspot user add name=test password=12345 profile=hs-user-profile
```

Why:

This creates a local Hotspot user.

This user is useful to test that Hotspot works before adding RADIUS.

WebFig / WinBox equivalent:

```text
IP → Hotspot → Users
Add New
Name: test
Password: 12345
Profile: hs-user-profile
Apply / OK
```

Check:

```
/ip hotspot user print
```

---

## 20. Test Hotspot from Ubuntu

Start the Ubuntu VM.

Renew DHCP:

```bash
sudo dhclient -r enp0s1
sudo ip addr flush dev enp0s1
sudo dhclient -v enp0s1
```

Check IP:

```bash
ip a
ip route
```

Expected result:

```text
IP address: 192.168.88.x/24
Default gateway: 192.168.88.1
```

Open Firefox and go to:

```text
http://neverssl.com
```

or directly:

```text
http://hotspot.lan/login
```

Login with:

```text
username: test
password: 12345
```

Check active Hotspot user on MikroTik:

```
/ip hotspot active print
```

WebFig / WinBox equivalent:

```text
IP → Hotspot → Active
```

---

## 21. Export backup before installing User Manager

Command:

```
/export file=before-user-manager-install
/file print
```

Why:

This saves the working Hotspot configuration before installing User Manager.

---

## 22. Check RouterOS version and architecture

Commands:

```
/system resource print
/system package print
```

Check:

```text
version
architecture-name
```

For the `chr.aarch64.qemu.7.22.1.utm` image, the package architecture should be `arm64`.

Use the exact same RouterOS version for the User Manager package.

Example:

```text
RouterOS version: 7.22.1
Architecture: arm64
```

---

## 23. Download MikroTik User Manager package

User Manager is not always installed by default. It is an extra package.

Download the matching package from MikroTik:

```text
https://mikrotik.com/download
```

For this setup, download the RouterOS `7.22.1` extra packages for `arm64`.

The expected package is:

```text
user-manager-7.22.1-arm64.npk
```

> [!note]
> Do not use the `arm` package for an `arm64/aarch64` CHR.
>
> The package version and architecture must match the installed RouterOS version and architecture.

---

## 24. Upload User Manager package to MikroTik

Use WebFig:

```text
Files → Upload
```

Upload:

```text
user-manager-7.22.1-arm64.npk
```

Check from CLI:

```
/file print
```

You should see the `.npk` file.

---

## 25. Reboot MikroTik to install User Manager

Command:

```
/system reboot
```

After reboot, check that the package is installed:

```
/system package print
```

You should see:

```text
user-manager
```

Also check:

```
/user-manager print
```

If the command exists, User Manager is installed.

---

## 26. Enable User Manager

Command:

```
/user-manager set enabled=yes
```

Check:

```
/user-manager print
```

Expected:

```text
enabled: yes
authentication-port: 1812
accounting-port: 1813
```

WebFig / WinBox equivalent:

```text
User Manager
Enabled: yes
```

---

## 27. Add MikroTik itself as a User Manager router

Because the RADIUS server and the Hotspot are on the same CHR, use localhost:

```
/user-manager router add name=local-hotspot address=127.0.0.1 shared-secret=localRadiusSecret123
```

Why:

User Manager must know which router is allowed to send RADIUS requests.

In this case, the router is the same CHR, so the address is:

```text
127.0.0.1
```

Check:

```
/user-manager router print
```

WebFig / WinBox equivalent:

```text
User Manager → Routers
Add New
Name: local-hotspot
Address: 127.0.0.1
Shared Secret: localRadiusSecret123
Apply / OK
```

---

## 28. Add RADIUS client on RouterOS

Command:

```
/radius add service=hotspot address=127.0.0.1 secret=localRadiusSecret123 authentication-port=1812 accounting-port=1813 timeout=1s
```

Why:

This tells RouterOS Hotspot to send authentication requests to the local User Manager RADIUS server.

The secret must match the User Manager router shared secret.

Check:

```
/radius print
```

WebFig / WinBox equivalent:

```text
RADIUS
Add New
Service: hotspot
Address: 127.0.0.1
Secret: localRadiusSecret123
Authentication Port: 1812
Accounting Port: 1813
Apply / OK
```

---

## 29. Enable RADIUS on Hotspot profile

Command:

```
/ip hotspot profile set hsprof1 dns-name=hotspot.lan login-by=http-chap,http-pap,cookie use-radius=yes radius-accounting=yes
```

Why:

This connects the Hotspot profile to RADIUS.

```text
use-radius=yes
```

makes Hotspot authenticate users through RADIUS.

```text
radius-accounting=yes
```

sends session accounting to User Manager.

WebFig / WinBox equivalent:

```text
IP → Hotspot → Server Profiles → hsprof1
DNS Name: hotspot.lan
Login By: http-chap, http-pap, cookie
Use RADIUS: checked
RADIUS Accounting: checked
Apply / OK
```

Check:

```
/ip hotspot profile print detail where name=hsprof1
```

---

## 30. Create a RADIUS user in User Manager

Command:

```
/user-manager user add name=radius1 password=12345 shared-users=1
```

Why:

This user is stored in User Manager, not in local Hotspot users.

Check:

```
/user-manager user print
```

WebFig / WinBox equivalent:

```text
User Manager → Users
Add New
Name: radius1
Password: 12345
Shared Users: 1
Apply / OK
```

---

## 31. Make sure DNS is correct after RADIUS setup

Run these commands on MikroTik:

```
/ip dns set allow-remote-requests=yes servers=1.1.1.1,8.8.8.8
/ip dhcp-server network set [find address=192.168.88.0/24] gateway=192.168.88.1 dns-server=192.168.88.1
```

Check static DNS:

```
/ip dns static print
```

Expected:

```text
hotspot.lan → 192.168.88.1
```

If it does not exist:

```
/ip dns static add name=hotspot.lan address=192.168.88.1 ttl=1h
```

> [!note]
> Do not remove the `hotspot.lan` entry if it already exists or is dynamic.
>
> The important final state is:
>
> ```text
> hotspot.lan → 192.168.88.1
> ```

---

## 32. Clean old Hotspot sessions and cookies

Command:

```
/ip hotspot active remove [find]
/ip hotspot cookie remove [find]
```

Why:

This avoids old cookie-based sessions confusing the new RADIUS test.

---

## 33. Reboot Ubuntu and test RADIUS login

Reboot Ubuntu or renew DHCP:

```bash
sudo dhclient -r enp0s1
sudo ip addr flush dev enp0s1
sudo dhclient -v enp0s1
```

Check DNS:

```bash
getent hosts hotspot.lan
getent hosts neverssl.com
```

Expected:

```text
hotspot.lan resolves to 192.168.88.1
neverssl.com resolves to a public IP
```

Open Firefox:

```text
http://neverssl.com
```

or:

```text
http://hotspot.lan/login
```

Login with User Manager user:

```text
username: radius1
password: 12345
```

Check on MikroTik:

```
/ip hotspot active print detail
/radius monitor 0
/user-manager session print
```

Expected:

```text
radius1 appears in active Hotspot users
RADIUS accepts counter increases
User Manager session is created
```

---

## 34. Add a second RADIUS user profile and block GitHub for that profile (exercisesing firewall rules)

This section adds a second Hotspot user profile, creates a second User Manager RADIUS user, puts users from this profile into a firewall address list, and blocks access to `github.com` only for that profile.

Final behavior:

```text
radius1 can access github.com normally.
radius2 cannot access github.com and has a different rate limit also.
```

---

### 34.1. Check that the Hotspot profile uses RADIUS

Command:

```
/ip hotspot profile print detail where name=hsprof1
```

Expected values:

```text
use-radius=yes
radius-accounting=yes
login-by=http-chap,http-pap,cookie
dns-name=hotspot.lan
```

Why:

This confirms that Hotspot users can be authenticated through User Manager / RADIUS.

If RADIUS is not enabled on the profile, enable it again:

```
/ip hotspot profile set hsprof1 dns-name=hotspot.lan login-by=http-chap,http-pap,cookie use-radius=yes radius-accounting=yes
```

---

### 34.2. Create a new Hotspot user profile with address-list

Command:

```
/ip hotspot user profile add name=hs-user-profile-github-blocked shared-users=1 rate-limit=512k/512k address-list=hs-github-blocked-users
```

Why:

This creates a new Hotspot user profile for restricted users.

The important fields are:

```text
name=hs-user-profile-github-blocked
```

This is the profile name that RADIUS will assign to the user.

```text
rate-limit=512k/512k
```

Users in this profile get a different speed limit.

```text
address-list=hs-github-blocked-users
```

When a user from this profile logs in, MikroTik automatically adds the user's IP address to this firewall address list.

Check:

```
/ip hotspot user profile print detail where name=hs-user-profile-github-blocked
```

Expected:

```text
name=hs-user-profile-github-blocked
shared-users=1
rate-limit=512k/512k
address-list=hs-github-blocked-users
```

---

### 34.3. Create the second RADIUS user in User Manager

Command:

```
/user-manager user add name=radius2 password=12345 shared-users=1 attributes=Mikrotik-Group:hs-user-profile-github-blocked
```

Why:

This creates a second user in User Manager.

The important part is:

```text
attributes=Mikrotik-Group:hs-user-profile-github-blocked
```

For MikroTik Hotspot, `Mikrotik-Group` tells Hotspot which Hotspot User Profile should be used for this RADIUS user.

So when `radius2` logs in, Hotspot applies this profile:

```text
hs-user-profile-github-blocked
```

That profile applies:

```text
rate-limit=512k/512k
address-list=hs-github-blocked-users
```

Check:

```
/user-manager user print detail where name=radius2
```

Expected:

```text
name=radius2
shared-users=1
attributes=Mikrotik-Group:hs-user-profile-github-blocked
```

---

### 34.4. Remove old Hotspot sessions and cookies

Command:

```
/ip hotspot active remove [find]
/ip hotspot cookie remove [find]
```

Why:

This makes the next test clean.

It removes old active sessions and old cookie-based logins that could cause confusion during testing.

---

### 34.5. Connect Ubuntu and log in with radius2

On Ubuntu, renew DHCP if needed:

```bash
sudo dhclient -r enp0s1
sudo ip addr flush dev enp0s1
sudo dhclient -v enp0s1
```

Open Firefox and go to:

```text
http://neverssl.com
```

or directly:

```text
http://hotspot.lan/login
```

Login with:

```text
username: radius2
password: 12345
```

Check active Hotspot users:

```
/ip hotspot active print detail
```

Expected:

```text
user=radius2
```

Check that the Ubuntu client IP was added to the profile address list:

```
/ip firewall address-list print where list=hs-github-blocked-users
```

Expected result:

```text
list=hs-github-blocked-users
address=192.168.88.x
dynamic=yes
```

Why:

This confirms that `radius2` used the correct Hotspot User Profile and was automatically added to the firewall address list.

---

### 34.6. Create a destination address list for GitHub

Command:

```
/ip firewall address-list add list=blocked-github address=github.com comment="Resolved github.com for blocked hotspot profile"
```

Why:

This creates a destination address list named:

```text
blocked-github
```

The address is set to:

```text
github.com
```

MikroTik resolves this domain and uses the resolved IP address as the destination match in firewall rules.

Check:

```
/ip firewall address-list print where list=blocked-github
```

Expected result:

```text
list=blocked-github
address=github.com
```

or a resolved GitHub IP address, depending on RouterOS display/output.

> [!note]
> This method is good for a lab setup, but it may not block every GitHub-related service in production because GitHub can use multiple domains and IP addresses.

---

### 34.7. Add the firewall rule to block GitHub for radius2 profile users

Command:

```
/ip firewall filter add chain=forward src-address-list=hs-github-blocked-users dst-address-list=blocked-github action=drop comment="Block github.com for hs-user-profile-github-blocked"
```

Why:

This firewall rule blocks traffic when both conditions are true:

```text
source IP is in hs-github-blocked-users
destination IP is in blocked-github
```

So only users from the restricted Hotspot profile are blocked from GitHub.

Check firewall rules:

```
/ip firewall filter print
```

You should see a rule similar to:

```text
;;; Block github.com for hs-user-profile-github-blocked
chain=forward src-address-list=hs-github-blocked-users dst-address-list=blocked-github action=drop
```

If you want to check whether the rule is actually being hit, use:

```
/ip firewall filter print stats where comment~"github"
```

---

### 34.8. Test GitHub access with radius2

Make sure Ubuntu is logged in with:

```text
username: radius2
password: 12345
```

On Ubuntu, test:

```bash
curl -I https://github.com --connect-timeout 5
```

Expected result:

```text
timeout
connection failed
or no successful HTTP response
```

You can also test in Firefox:

```text
https://github.com
```

Expected result:

```text
github.com should not open for radius2.
```

Check firewall counter on MikroTik:

```
/ip firewall filter print stats where comment~"github"
```

Expected:

```text
packets or bytes should increase when radius2 tries to access github.com.
```

---

### 34.9. Test GitHub access with radius1

Remove active sessions and cookies:

```
/ip hotspot active remove [find]
/ip hotspot cookie remove [find]
```

Login again from Ubuntu with:

```text
username: radius1
password: 12345
```

Test GitHub:

```bash
curl -I https://github.com --connect-timeout 5
```

Expected result:

```text
github.com should be accessible for radius1.
```

Why:

`radius1` does not use the restricted Hotspot profile, so its IP should not be in:

```text
hs-github-blocked-users
```

Check:

```
/ip firewall address-list print where list=hs-github-blocked-users
```

Expected:

```text
radius1 client IP should not be listed there.
```

---

### 34.10. Optional: add more GitHub-related domains

`github.com` is enough for a basic lab test.

If you want to block more GitHub-related destinations, add these too:

```
/ip firewall address-list add list=blocked-github address=www.github.com comment="GitHub www"
/ip firewall address-list add list=blocked-github address=api.github.com comment="GitHub API"
/ip firewall address-list add list=blocked-github address=raw.githubusercontent.com comment="GitHub raw content"
/ip firewall address-list add list=blocked-github address=githubusercontent.com comment="GitHub user content"
```

Check:

```
/ip firewall address-list print where list=blocked-github
```

The same firewall rule still applies:

```
/ip firewall filter add chain=forward src-address-list=hs-github-blocked-users dst-address-list=blocked-github action=drop comment="Block GitHub destinations for blocked Hotspot profile"
```

> [!note]
> Do not add a duplicate firewall rule if you already created one. Check first:
>
> ```
> /ip firewall filter print where comment~"GitHub"
> ```

---

### 34.11. Troubleshooting GitHub blocking

Check the restricted profile:

```
/ip hotspot user profile print detail where name=hs-user-profile-github-blocked
```

Check the RADIUS user:

```
/user-manager user print detail where name=radius2
```

Check active Hotspot user:

```
/ip hotspot active print detail
```

Check whether the client was added to the restricted address list:

```
/ip firewall address-list print where list=hs-github-blocked-users
```

Check GitHub destination address list:

```
/ip firewall address-list print where list=blocked-github
```

Check firewall rule and counters:

```
/ip firewall filter print
/ip firewall filter print stats where comment~"github"
```

If `hs-github-blocked-users` is empty, `radius2` did not get the correct Hotspot User Profile.

Check:

```
/user-manager user print detail where name=radius2
/ip hotspot user profile print detail where name=hs-user-profile-github-blocked
```

If `blocked-github` is empty or does not resolve, check DNS:

```
/ip dns print
/ping github.com count=2
```

If the firewall rule counter does not increase, move the rule higher:

```
/ip firewall filter move [find comment~"github"] destination=0
```

Then test again:

```bash
curl -I https://github.com --connect-timeout 5
```

---

### 34.12. Final expected result for radius2 profile

At the end of this section, the setup should work like this:

```text
radius1 logs in through User Manager RADIUS.
radius1 uses normal Hotspot access.
radius1 can access github.com.

radius2 logs in through User Manager RADIUS.
radius2 gets Hotspot User Profile hs-user-profile-github-blocked.
radius2 IP is added to firewall address-list hs-github-blocked-users.
Firewall blocks radius2 access to github.com.
radius2 still has normal internet access except blocked GitHub destinations.
```

---

# Troubleshooting and verification commands

## Interface checks

```
/interface print
/ip address print
```

WebFig / WinBox:

```text
Interfaces
IP → Addresses
```

---

## WAN / internet checks from MikroTik

```
/ping 1.1.1.1 count=3
/ping google.com count=3
/ip route print
```

Expected:

```text
Default route via ether1 gateway
```

WebFig / WinBox:

```text
IP → Routes
```

---

## DNS checks on MikroTik

```
/ip dns print
/ip dns static print
```

Expected:

```text
allow-remote-requests=yes
servers=1.1.1.1,8.8.8.8
hotspot.lan → 192.168.88.1
```

WebFig / WinBox:

```text
IP → DNS
IP → DNS → Static
```

---

## DHCP checks

```
/ip dhcp-server print
/ip dhcp-server network print
/ip dhcp-server lease print
```

Expected:

```text
dhcp-hotspot on ether2
network 192.168.88.0/24
gateway 192.168.88.1
dns-server 192.168.88.1
Ubuntu appears in leases
```

WebFig / WinBox:

```text
IP → DHCP Server
IP → DHCP Server → Networks
IP → DHCP Server → Leases
```

---

## NAT checks

```
/ip firewall nat print
```

Expected:

```text
chain=srcnat
out-interface=ether1
action=masquerade
```

WebFig / WinBox:

```text
IP → Firewall → NAT
```

---

## Hotspot checks

```
/ip hotspot print
/ip hotspot profile print detail where name=hsprof1
/ip hotspot user print
/ip hotspot host print
/ip hotspot active print detail
```

WebFig / WinBox:

```text
IP → Hotspot → Servers
IP → Hotspot → Server Profiles
IP → Hotspot → Users
IP → Hotspot → Hosts
IP → Hotspot → Active
```

---

## RADIUS checks

```
/radius print detail
/radius monitor 0
```

Useful counters:

```text
accepts  → successful RADIUS login
rejects  → wrong user/password or disabled user
timeouts → RADIUS server not reachable or not enabled
bad-replies → shared secret mismatch
```

WebFig / WinBox:

```text
RADIUS
```

---

## User Manager checks

```
/user-manager print
/user-manager router print
/user-manager user print
/user-manager session print
```

WebFig / WinBox:

```text
User Manager
User Manager → Routers
User Manager → Users
User Manager → Sessions
```

---

## Ubuntu client checks

```bash
ip a
ip route
cat /etc/resolv.conf
resolvectl status
getent hosts hotspot.lan
getent hosts neverssl.com
```

Expected:

```text
IP address: 192.168.88.x/24
Default gateway: 192.168.88.1
DNS server: 192.168.88.1
hotspot.lan resolves to 192.168.88.1
```

---

## Common problems

### Ubuntu gets IP from UTM instead of MikroTik

Wrong network mode.

Fix:

```text
MikroTik ether2 = Bridged to bridge0
Ubuntu = Bridged to bridge0
Do not use Emulated VLAN
```

---

### hotspot.lan does not open

Check MikroTik DNS:

```
/ip dns print
/ip dns static print
/ip dhcp-server network print
```

Check Ubuntu DNS:

```bash
getent hosts hotspot.lan
resolvectl status
```

Firefox DNS over HTTPS may also cause problems. Disable it:

```text
Firefox → Settings → Privacy & Security → DNS over HTTPS → Off
```

---

### Login works but internet does not

Check MikroTik internet:

```
/ping 1.1.1.1 count=3
/ping google.com count=3
```

Check NAT:

```
/ip firewall nat print
```

Check Ubuntu DNS:

```bash
getent hosts neverssl.com
```

---

### RADIUS login fails

Check:

```
/radius monitor 0
/log print where topics~"radius|hotspot"
/user-manager user print
/user-manager router print
```

Meaning:

```text
rejects → wrong username/password
timeouts → User Manager not enabled or wrong address
bad-replies → shared secret mismatch
```

---

### radius2 does not get the restricted profile

Check:

```
/user-manager user print detail where name=radius2
/ip hotspot user profile print detail where name=hs-user-profile-github-blocked
/ip hotspot active print detail
```

Expected:

```text
radius2 should use Mikrotik-Group:hs-user-profile-github-blocked.
```

Also check the address list:

```
/ip firewall address-list print where list=hs-github-blocked-users
```

Expected:

```text
The Ubuntu client IP should be listed after radius2 login.
```

---

### radius2 can still access GitHub

Check the blocked GitHub address list:

```
/ip firewall address-list print where list=blocked-github
```

Check DNS:

```
/ip dns print
/ping github.com count=2
```

Check firewall rule counters:

```
/ip firewall filter print stats where comment~"github"
```

If the counter is not increasing, move the rule higher:

```
/ip firewall filter move [find comment~"github"] destination=0
```

Then test again from Ubuntu:

```bash
curl -I https://github.com --connect-timeout 5
```

---

## Final expected state

At the end, the setup should work like this:

```text
Ubuntu connects to MikroTik ether2
Ubuntu gets IP from MikroTik DHCP
Ubuntu opens neverssl.com
MikroTik redirects to Hotspot login page
User logs in with radius1 / 12345
Hotspot sends authentication to local User Manager RADIUS
User Manager accepts login
Ubuntu gets internet access through MikroTik NAT
```

After adding the second restricted profile, the additional expected state is:

```text
radius1 can log in through Hotspot RADIUS and access github.com
radius2 can log in through Hotspot RADIUS
radius2 receives the hs-user-profile-github-blocked profile
radius2 gets rate-limit 512k/512k
radius2 IP is added to hs-github-blocked-users
radius2 cannot access github.com
radius2 still has general internet access
```