# PavelKireev![Схема (Киреев С-42)](https://user-images.githubusercontent.com/90326215/154075081-0210959a-7170-4e39-aa0a-962230ce465a.png)

Назначение имен и IP Адресов

ISP
```ISP
hostnamectl set-hostname ISP
apt-cdrom add
apt install -y network-manager bind9 chrony 
nmcli connection modify Wired\ connection\ 1 conn.autoconnect yes conn.interface-name ens192 ipv4.method manual ipv4.addresses '3.3.3.1/24'
nmcli connection modify Wired\ connection\ 2 conn.autoconnect yes conn.interface-name ens224 ipv4.method manual ipv4.addresses '4.4.4.1/24'
nmcli connection modify Wired\ connection\ 3 conn.autoconnect yes conn.interface-name ens256 ipv4.method manual ipv4.addresses '5.5.5.1/24'

nano /etc/sysctl.conf
net.ipv4.ip_forward=1
sysctl -0

apt-cdrom add
apt install -y bind9
mkdir /opt/dns
cp /etc/bind/db.local /opt/dns/demo.db
chown -R bind:bind /opt/dns
nano /etc/apparmor.d/usr.sbin.named
    /opt/dns/** rw,
systemctl restart apparmor.service
   nano /etc/bind/named.conf.options
nano /etc/bind/named.conf.default-zones
zone "demo.wsr" {
   type master;
   allow-transfer { any; };
   file "/opt/dns/demo.db";
};

nano /opt/dns/demo.db

@ IN SOA demo.wsr. root.demo.wsr.(

@ IN NS isp.demo.wsr.
isp IN A 3.3.3.1
www IN A 4.4.4.100
www IN A 5.5.5.100
internet CNAME isp.demo.wsr.
int IN NS rtr-l.demo.wsr.
rtr-l IN  A 4.4.4.100
systemctl restart bind9

apt install -y chrony
nano /etc/chrony/chrony.conf
local stratum 4
allow 4.4.4.0/24
allow 3.3.3.0/24
systemctl restart chronyd 
```

RTR-L
```RTR-L
hostname RTR-L
int gi 1
ip address 4.4.4.100 255.255.255.0
no sh
int gi 2
ip address 192.168.100.254 255.255.255.0
no sh

ip route 0.0.0.0 0.0.0.0 4.4.4.1

int gi 1
ip nat outside

int gi 2
ip nat inside

access-list 1 permit 192.168.100.0 0.0.0.255
ip nat inside source list 1 interface Gi1 overload

interface Tunne 1
ip address 172.16.1.1 255.255.255.0
tunnel mode gre ip
tunnel source 4.4.4.100
tunnel destination 5.5.5.100

router eigrp 6500
network 192.168.100.0 0.0.0.255
network 172.16.1.0 0.0.0.255

crypto isakmp policy 1
encr aes
authentication pre-share
hash sha256
group 14

crypto isakmp key TheSecretMustBeAtLeast13bytes address 5.5.5.100
crypto isakmp nat keepalive 5

crypto ipsec transform-set TSET  esp-aes 256 esp-sha256-hmac
mode tunnel

crypto ipsec profile VTI
set transform-set TSET

interface Tunnel1
tunnel mode ipsec ipv4
tunnel protection ipsec profile VTI

ip access-list extended Lnew
    permit tcp any any established
    permit udp host 4.4.4.100 eq 53 any
    permit udp host 5.5.5.1 eq 123 any
    permit tcp any host 4.4.4.100 eq 80 
    permit tcp any host 4.4.4.100 eq 443 
    permit tcp any host 4.4.4.100 eq 2222 
    permit udp host 5.5.5.100 host 4.4.4.100 eq 500
    permit esp any any
    permit icmp any any

int gi 1 
ip access-group Lnew in

ip nat inside source static tcp 192.168.100.100 22 4.4.4.100 2222

ip nat inside source static tcp 192.168.100.200 53 4.4.4.100 53
ip nat inside source static udp 192.168.100.200 53 4.4.4.100 53

ip domain name int.demo.wsr
ip name-server 192.168.100.200
ntp server ntp.int.demo.wsr
```

RTR-R
```RTR-R
hostname RTR-R
int gi 1
ip address 5.5.5.100 255.255.255.0
no sh
int gi 2
ip address 172.16.100.254 255.255.255.0
no sh

ip route 0.0.0.0 0.0.0.0 5.5.5.1

int gi 1
ip nat outside
!
int gi 2
ip nat inside
!
access-list 1 permit 172.16.100.0 0.0.0.255
ip nat inside source list 1 interface Gi1 overload

interface Tunne 1
ip address 172.16.1.2 255.255.255.0
tunnel mode gre ip
tunnel source 5.5.5.100
tunnel destination 4.4.4.100

router eigrp 6500
network 172.16.100.0 0.0.0.255
network 172.16.1.0 0.0.0.255

conf t

crypto isakmp policy 1
encr aes
authentication pre-share
hash sha256
group 14

crypto isakmp key TheSecretMustBeAtLeast13bytes address 4.4.4.100
crypto isakmp nat keepalive 5

crypto ipsec transform-set TSET  esp-aes 256 esp-sha256-hmac
mode tunnel

crypto ipsec profile VTI
set transform-set TSET

interface Tunnel1
tunnel mode ipsec ipv4
tunnel protection ipsec profile VTI

ip access-list extended Rnew
    permit tcp any any established
    permit tcp any host 5.5.5.100 eq 80 
    permit tcp any host 5.5.5.100 eq 443 
    permit tcp any host 5.5.5.100 eq 2244 
    permit udp host 4.4.4.100 host 5.5.5.100 eq 500
    permit esp any any
    permit icmp any any

int gi 1 
ip access-group Rnew in

ip nat inside source static tcp 172.16.100.100 22 5.5.5.100 2244

ip domain name int.demo.wsr
ip name-server 192.168.100.200
ntp server ntp.int.demo.wsr
```

SRV
```SRV
Rename-Computer -NewName SRV
$GetIndex = Get-NetAdapter
New-NetIPAddress -InterfaceIndex $GetIndex.ifIndex -IPAddress 192.168.100.200 -PrefixLength 24 -DefaultGateway 192.168.100.254
Set-DnsClientServerAddress -InterfaceIndex $GetIndex.ifIndex -ServerAddresses ("192.168.100.200","4.4.4.1")
Set-NetFirewallRule -DisplayGroup "File And Printer Sharing" -Enabled True -Profile Any

Install-WindowsFeature -Name DNS -IncludeManagementTools
Add-DnsServerPrimaryZone -Name "int.demo.wsr" -ZoneFile "int.demo.wsr.dns"
Add-DnsServerPrimaryZone -NetworkId 192.168.100.0/24 -ZoneFile "int.demo.wsr.dns"
Add-DnsServerPrimaryZone -NetworkId 172.16.100.0/24 -ZoneFile "int.demo.wsr.dns"

Add-DnsServerResourceRecordA -Name "web-l" -ZoneName "int.demo.wsr" -AllowUpdateAny -IPv4Address "192.168.100.100" -CreatePtr 
Add-DnsServerResourceRecordA -Name "web-r" -ZoneName "int.demo.wsr" -AllowUpdateAny -IPv4Address "172.16.100.100" -CreatePtr 
Add-DnsServerResourceRecordA -Name "srv" -ZoneName "int.demo.wsr" -AllowUpdateAny -IPv4Address "192.168.100.200" -CreatePtr 
Add-DnsServerResourceRecordA -Name "rtr-l" -ZoneName "int.demo.wsr" -AllowUpdateAny -IPv4Address "192.168.100.254" -CreatePtr 
Add-DnsServerResourceRecordA -Name "rtr-r" -ZoneName "int.demo.wsr" -AllowUpdateAny -IPv4Address "172.16.100.254" -CreatePtr

Add-DnsServerResourceRecordCName -Name "webapp1" -HostNameAlias "web-l.int.demo.wsr" -ZoneName "int.demo.wsr"
Add-DnsServerResourceRecordCName -Name "webapp2" -HostNameAlias "web-r.int.demo.wsr" -ZoneName "int.demo.wsr"
Add-DnsServerResourceRecordCName -Name "ntp" -HostNameAlias "srv.int.demo.wsr" -ZoneName "int.demo.wsr"
Add-DnsServerResourceRecordCName -Name "dns" -HostNameAlias "srv.int.demo.wsr" -ZoneName "int.demo.wsr"

New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow
w32tm /query /status
Start-Service W32Time
w32tm /config /manualpeerlist:4.4.4.1 /syncfromflags:manual /reliable:yes /update
Restart-Service W32Time

get-disk
set-disk -Number 1 -IsOffline $false
set-disk -Number 2 -IsOffline $false
New-StoragePool -FriendlyName "POOLRAID1" -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -CanPool $true)
New-VirtualDisk -StoragePoolFriendlyName "POOLRAID1" -FriendlyName "RAID1" -ResiliencySettingName Mirror -UseMaximumSize
Initialize-Disk -FriendlyName "RAID1"
New-Partition -DiskNumber 3 -UseMaximumSize -DriveLetter R
Format-Volume -DriveLetter R

Install-WindowsFeature -Name FS-FileServer -IncludeManagementTools
New-Item -Path R:\storage -ItemType Directory
New-SmbShare -Name "SMB" -Path "R:\storage" -FullAccess "Everyone"
```

WEB-L
```WEB-L
hostnamectl set-hostname WEB-L
apt-cdrom add
apt install -y network-manager
nmcli connection show
nmcli connection modify Wired\ connection\ 1 conn.autoconnect yes conn.interface-name ens192 ipv4.method manual ipv4.addresses '192.168.100.100/24' ipv4.dns 192.168.100.200 ipv4.gateway 192.168.100.254

apt-cdrom add
apt install -y openssh-server ssh
systemctl start sshd
systemctl enable ssh

apt-cdrom add
apt install -y chrony 
nano /etc/chrony/chrony.conf
pool ntp.int.demo.wsr iburst
allow 192.168.100.0/24
systemctl restart chrony
```

WEB-R
```WEB-R
hostnamectl set-hostname WEB-R
apt-cdrom add
apt install -y network-manager
nmcli connection show
nmcli connection modify Wired\ connection\ 1 conn.autoconnect yes conn.interface-name ens192 ipv4.method manual ipv4.addresses '172.16.100.100/24' ipv4.dns 192.168.100.200 ipv4.gateway 172.16.100.254

apt-cdrom add
apt install -y chrony 
nano /etc/chrony/chrony.conf
pool ntp.int.demo.wsr iburst
allow 192.168.100.0/24
systemctl restart chrony
```

CLI
```CLI
Rename-Computer -NewName CLI
$GetIndex = Get-NetAdapter
New-NetIPAddress -InterfaceIndex $GetIndex.ifIndex -IPAddress 3.3.3.10 -PrefixLength 24 -DefaultGateway 3.3.3.1
Set-DnsClientServerAddress -InterfaceIndex $GetIndex.ifIndex -ServerAddresses ("3.3.3.1")
New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow
Start-Service W32Time
w32tm /config /manualpeerlist:4.4.4.1 /syncfromflags:manual /reliable:yes /update
Restart-Service W32Time
Set-Service -Name W32Time -StartupType Automatic
```
