# PavelKireev![Схема (Киреев С-42)](https://user-images.githubusercontent.com/90326215/154075081-0210959a-7170-4e39-aa0a-962230ce465a.png)

ISP
```ISP
apt-cdrom add
apt install -y network-manager bind9 chrony

nano /etc/sysctl.conf
net.ipv4.ip_forward=1
sysctl -p

mkdir /opt/dns
cp /etc/bind/db.local /opt/dns/demo.db
chown -R bind:bind /opt/dns
nano /etc/apparmor.d/usr.sbin.named
    /opt/dns/** rw,
systemctl restart apparmor.service

nano /etc/bind/named.conf.options
    dnssec-validation no;
    allow-query { any; };
    listen-on-v6 { any: }:
    
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

no ip http secure-server
wr
reload
ip nat inside source static tcp 192.168.100.100 80 4.4.4.100 80 
ip nat inside source static tcp 192.168.100.100 443 4.4.4.100 443 
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
int gi 2
ip nat inside

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

no ip http secure-server
wr
reload
ip nat inside source static tcp 172.16.100.100 80 5.5.5.100 80 
ip nat inside source static tcp 172.16.100.100 443 5.5.5.100 443 
```

SRV
```SRV
Настроить DNS

NTP
New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow
w32tm /query /status
Start-Service W32Time
w32tm /config /manualpeerlist:4.4.4.1 /syncfromflags:manual /reliable:yes /update
Restart-Service W32Time
```
```
SMB
```
![Снимок экрана (195)](https://user-images.githubusercontent.com/90326215/173954555-043c08da-42c0-4a5d-924f-75c1c139a837.png)
![Снимок экрана (196)](https://user-images.githubusercontent.com/90326215/173954562-03aeba8e-cc19-4554-b731-96e63939218c.png)
![Снимок экрана (197)](https://user-images.githubusercontent.com/90326215/173954572-9dbcedf1-754f-4ce8-9aac-6e26a56883ee.png)
![Снимок экрана (198)](https://user-images.githubusercontent.com/90326215/173954579-e966b2ec-c5c4-4bfb-ba11-353e75fe5fc5.png)
![Снимок экрана (199)](https://user-images.githubusercontent.com/90326215/173954583-37640147-5491-44e1-8c01-5bc1d666cd1b.png)
![Снимок экрана (200)](https://user-images.githubusercontent.com/90326215/173954586-a26fad4b-aa96-42a5-938d-ef11e1ec37bd.png)

```
Install-WindowsFeature -Name FS-FileServer -IncludeManagementTools
New-Item -Path R:\storage -ItemType Directory
New-SmbShare -Name "SMB" -Path "R:\storage" -FullAccess "Everyone"

Install-WindowsFeature -Name AD-Certificate, ADCS-Web-Enrollment -IncludeManagementTools
Install-AdcsCertificationAuthority -CAType StandaloneRootCa -CACommonName "Demo.wsr" -force
Install-AdcsWebEnrollment -Confirm -force
New-SelfSignedCertificate -subject "localhost" 
Get-ChildItem cert:\LocalMachine\My
Move-item Cert:\LocalMachine\My\XFX2DX02779XFD1F6F4X8435A5X26ED2X8DEFX95 -destination Cert:\LocalMachine\Webhosting\
New-IISSiteBinding -Name 'Default Web Site' -BindingInformation "*:443:" -Protocol https -CertificateThumbPrint XFX2DX02779XFD1F6F4X8435A5X26ED2X8DEFX95 
Start-WebSite -Name "Default Web Site"
Get-CACrlDistributionPoint | Remove-CACrlDistributionPoint -force
Get-CAAuthorityInformationAccess |Remove-CAAuthorityInformationAccess -force
Get-CAAuthorityInformationAccess |Remove-CAAuthorityInformationAccess -force
Restart-Service CertSrc
```
![150323908-1e37041d-450f-477e-9f9f-24a5f1313e54](https://user-images.githubusercontent.com/90326215/173956834-c1b3ca33-32f3-4bf0-ae57-4fe5fa9f7fc5.png)
![150323973-2b4fd9a8-d934-4a49-8e7d-98e02efd7826](https://user-images.githubusercontent.com/90326215/173956850-78f1c0a3-a5fa-4207-abb8-005bd42b1705.png)
![150324325-b67039dc-6d6a-4dd1-9ade-67bbcaa3a6db](https://user-images.githubusercontent.com/90326215/173956851-66cc1f15-7428-42db-8dbb-f056db92ba13.png)
![150324402-d9e1c93a-46b0-42ac-83a0-02b97da4829b](https://user-images.githubusercontent.com/90326215/173956856-87eee0c9-8329-40d4-b0ef-71a141912086.png)
![149763513-cb821774-83ce-40f9-a47d-eaa3941504a5](https://user-images.githubusercontent.com/90326215/173956862-fb55ec45-ca85-4bae-8fa6-eccd1574c8a6.png)
![149763553-5a666a1c-6069-4022-9e20-1e3041d345b7](https://user-images.githubusercontent.com/90326215/173956867-b0108007-a2f6-40a5-8b6d-04a699b65d28.png)
![149763685-b0dfb33e-ed50-4de6-937c-1aab86524df9](https://user-images.githubusercontent.com/90326215/173956869-4ac13d7b-c2fd-4afc-a9fc-23adc31cbb9f.png)
![149763717-75fa754c-4084-4923-acd8-7dc85201fdb1](https://user-images.githubusercontent.com/90326215/173956874-1de4fca0-d999-48bd-8fc8-d94f9f295f52.png)
![149763741-0e3281f3-b17b-4101-ad77-8dd21dfc7534](https://user-images.githubusercontent.com/90326215/173956878-72c89e1a-eca5-4dbd-946e-7bf5e3dbf761.png)
![149763772-8f7acb37-379f-455e-9198-1117317d73ed](https://user-images.githubusercontent.com/90326215/173956881-2cc780d0-afb3-4159-a198-c0d6d1bad827.png)
![149763807-f91c2a81-d061-49bd-9b59-b93ae0e30ef3](https://user-images.githubusercontent.com/90326215/173956884-302b7414-2ae5-4c95-8685-5d1e43c7bf10.png)


WEB-L/WEB-R
```WEB-L/WEB-R
apt-cdrom add
apt install -y network-manager openssh-server ssh chrony cifs-utils nginx

systemctl start sshd
systemctl enable ssh

nano /etc/chrony/chrony.conf
pool ntp.int.demo.wsr iburst
allow 192.168.100.0/24
systemctl restart chrony

nano /root/.smbclient
    username=Administrator
    password=Pa$$w0rd
nano /etc/fstab
    //srv.int.demo.wsr/smb /opt/share cifs user,rw,_netdev,credentials=/root/.smbclient 0 0
mkdir /opt/share
mount -a

nano /etc/apt/sourselist
apt-cdrom add
apt install -y docker-ce
systemctl start docker
systemctl enable docker
mkdir /mnt/app
mount /dev/sr1 /mnt/app
docker load < /mnt/app/app.tar
docker images
docker run --name app  -p 8080:80 -d app
docker ps

cd /opt/share
openssl pkcs12 -nodes -nocerts -in www.pfx -out www.key

openssl pkcs12 -nodes -nokeys -in www.pfx -out www.cer
cp /opt/share/www.key /etc/nginx/www.key

cp /opt/share/www.cer /etc/nginx/www.cer
nano /etc/nginx/snippets/snakeoil.conf

nano /etc/nginx/sites-available/default
upstream backend { 
 server 192.168.100.100:8080 fail_timeout=25; 
 server 172.16.100.100:8080 fail_timeout=25; 
} 
 
server { 
    listen 443 ssl default_server; 
    include snippers/snakeoil.conf;
    server_name www.demo.wsr; 
    location / { 
        proxy_pass http://backend ;
    } 
}

server { 
    listen 80  default_server; 
    server_name _; 
    return 301 https://www.demo.wsr;
}

systemctl reload nginx

nano /etc/ssh/sshd_config
    permitRootLogin yes
systemctl restart sshd
```

CLI
```CLI
New-NetFirewallRule -DisplayName "NTP" -Direction Inbound -LocalPort 123 -Protocol UDP -Action Allow
Start-Service W32Time
w32tm /config /manualpeerlist:4.4.4.1 /syncfromflags:manual /reliable:yes /update
Restart-Service W32Time
Set-Service -Name W32Time -StartupType Automatic

scp -P 2244 'root@5.5.5.100:/opt/share/ca.cer' C:\Users\user\Desktop\
```
