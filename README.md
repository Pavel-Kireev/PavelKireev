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
```

SRV
```SRV
Rename-Computer -NewName SRV
$GetIndex = Get-NetAdapter
New-NetIPAddress -InterfaceIndex $GetIndex.ifIndex -IPAddress 192.168.100.200 -PrefixLength 24 -DefaultGateway 192.168.100.254
Set-DnsClientServerAddress -InterfaceIndex $GetIndex.ifIndex -ServerAddresses ("192.168.100.200","4.4.4.1")
Set-NetFirewallRule -DisplayGroup "File And Printer Sharing" -Enabled True -Profile Any
```

WEB-L
```WEB-L
hostnamectl set-hostname WEB-L
apt-cdrom add
apt install -y network-manager
nmcli connection show
nmcli connection modify Wired\ connection\ 1 conn.autoconnect yes conn.interface-name ens192 ipv4.method manual ipv4.addresses '192.168.100.100/24' ipv4.dns 192.168.100.200 ipv4.gateway 192.168.100.254
```

WEB-R
```WEB-R
hostnamectl set-hostname WEB-R
apt-cdrom add
apt install -y network-manager
nmcli connection show
nmcli connection modify Wired\ connection\ 1 conn.autoconnect yes conn.interface-name ens192 ipv4.method manual ipv4.addresses '172.16.100.100/24' ipv4.dns 192.168.100.200 ipv4.gateway 172.16.100.254
```

CLI
```CLI
Rename-Computer -NewName CLI
$GetIndex = Get-NetAdapter
New-NetIPAddress -InterfaceIndex $GetIndex.ifIndex -IPAddress 3.3.3.10 -PrefixLength 24 -DefaultGateway 3.3.3.1
Set-DnsClientServerAddress -InterfaceIndex $GetIndex.ifIndex -ServerAddresses ("3.3.3.1")
```
