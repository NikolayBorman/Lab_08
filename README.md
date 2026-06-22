# Реализация DHCPv4 

## Топология



## Таблица адресации

| Устройство | Интерфейс      | IP-адрес        | Маска подсети    | Шлюз по умолчанию |
|------------|----------------|-----------------|------------------|-------------------|
| R1         | G0/0/0         | 10.0.0.1        | 255.255.255.252  | —                 |
| R1         | G0/0/1.100     | 192.168.1.1     | 255.255.255.192  | —                 |
| R1         | G0/0/1.200     | 192.168.1.65    | 255.255.255.224  | —                 |
| R1         | G0/0/1.1000    | —               | —                | —                 |
| R2         | G0/0/0         | 10.0.0.2        | 255.255.255.252  | —                 |
| R2         | G0/0/1         | 192.168.1.97    | 255.255.255.240  | —                 |
| S1         | VLAN 200       | 192.168.1.66    | 255.255.255.224  | 192.168.1.65      |
| S2         | VLAN 1         | 192.168.1.98    | 255.255.255.240  | 192.168.1.97      |
| PC‑A       | NIC            | DHCP            | DHCP             | DHCP              |
| PC‑B       | NIC            | DHCP            | DHCP             | DHCP              |


## Таблица VLAN

| VLAN | Имя         | Назначенные интерфейсы                        |
|------|-------------|-----------------------------------------------|
| 1    | —           | S2: F0/18                                     |
| 100  | Client    | S1: F0/6                                      |
| 200  | Managment  | S1: VLAN 200                                  |
| 999  | Parking_Lot | S1: F0/1-4, F0/7-24, G0/1-2                   |
| 1000 | Native      | —                                             |

---

## Часть 1. Создание сети и настройка основных параметров устройств

### 1.1 Расчёт подсетей

- **Подсеть A** (58 хостов): `/26` (64 адреса) → 192.168.1.0/26.  
  Первый адрес: 192.168.1.1 (R1 G0/0/1.100).  
  Диапазон хостов: 192.168.1.1 – 192.168.1.62.

- **Подсеть B** (28 хостов): `/27` (32 адреса) → 192.168.1.64/27.  
  Первый адрес: 192.168.1.65 (R1 G0/0/1.200).  
  Второй адрес: 192.168.1.66 (S1 VLAN 200).  
  Шлюз для S1: 192.168.1.65.

- **Подсеть C** (12 хостов): `/28` (16 адреса) → 192.168.1.96/28.  
  Первый адрес: 192.168.1.97 (R2 G0/0/1).  
  Второй адрес: 192.168.1.98 (S2 VLAN 1).  
  Шлюз для S2: 192.168.1.97.

- **Сеть между R1 и R2**: 10.0.0.0/30.

### 1.2 Базовая настройка маршрутизатора R1
```cisco
R1#enable
R1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#no ip domain-lookup
R1(config)#hostname R1
R1(config)#enable secret class
R1(config)#line console 0
R1(config-line)#password cisco
R1(config-line)#login
R1(config-line)#logging synchronous
R1(config-line)#exec-timeout 5 0
R1(config-line)#exit
R1(config)#line vty 0 4
R1(config-line)#password cisco
R1(config-line)#login
R1(config-line)#exec-timeout 5 0
R1(config-line)#exit
R1(config)#service password-encryption
R1(config)#banner motd ^CGO OUT NOW!^C
R1(config)#clock timezone MSC 3
R1(config)#end
R1#copy running-config startup-config
%SYS-5-CONFIG_I: Configured from console by console

R1#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]
```
### 1.3 Базовая настройка маршрутизатора R2
Настраиваю аналогично R1, использую пакетное конфигурирование, меняя название устройств:
```cisco

Router>enable
Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#no ip domain-lookup
Router(config)#hostname R2
R2(config)#enable secret class
R2(config)#line console 0
R2(config-line)#password cisco
R2(config-line)#login
R2(config-line)#logging synchronous
R2(config-line)#exec-timeout 5 0
R2(config-line)#exit
R2(config)#line vty 0 4
R2(config-line)#password cisco
R2(config-line)#login
R2(config-line)#exec-timeout 5 0
R2(config-line)#exit
R2(config)#service password-encryption
R2(config)#banner motd ^CGO OUT NOW!^C
R2(config)#clock timezone MSC 3
R2(config)#end
R2#copy running-config startup-config
%SYS-5-CONFIG_I: Configured from console by console

R2#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]
```
### 1.4 Настройка подинтерфейсов на R1 
Настраиваю межвлановую маршрутизацию: поднимаю подинтерфейсы, задаю стандарт инкапсуляции, даю адреса, задаю имена.
```cisco
R1>enable
R1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#interface g0/0/1
R1(config-if)#no sh
R1(config-if)#exit
R1(config)#int g0/0/1.100
R1(config-subif)#
%LINK-3-UPDOWN: Interface GigabitEthernet0/0/1.100, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/1.100, changed state to up

R1(config-subif)#encapsulation dot1q 100
R1(config-subif)#ip address 192.168.1.1 255.255.255.192
R1(config-subif)#description Client
R1(config-subif)#exit
R1(config)#int g0/0/1.200
R1(config-subif)#
%LINK-3-UPDOWN: Interface GigabitEthernet0/0/1.200, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/1.200, changed state to up

R1(config-subif)#encapsulation dot1q 200
R1(config-subif)#ip address 192.168.1.65 255.255.255.224
R1(config-subif)#description Managment
R1(config-subif)#exit
R1(config)#int g0/0/1.1000
R1(config-subif)#
%LINK-3-UPDOWN: Interface GigabitEthernet0/0/1.1000, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/1.1000, changed state to up

R1(config-subif)#encapsulation dot1q 1000
R1(config-subif)#desc Native
R1(config-subif)#exit
R1(config)#end
R1#
%SYS-5-CONFIG_I: Configured from console by console

R1#copy running-config startup-config 
Destination filename [startup-config]? 
Building configuration...
[OK]
```
### 1.5 Настройка интерфейсов R2 и статической маршрутизации
Интерфейс G0/0/1
```cisco
GO OUT NOW!^

User Access Verification

Password:
R2>enable
Password: 
R2#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R2(config)#int g0/0/1
R2(config-if)#ip add 192.168.1.97 255.255.255.240
R2(config-if)#no sh

R2(config-if)#
%LINK-5-CHANGED: Interface GigabitEthernet0/0/1, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/1, changed state to up

R2(config-if)#exit
```
Интерфейсы G0/0/0 (R1-R2)
```cisco
GO OUT NOW!^

User Access Verification

Password: 

R1>enable
Password: 
R1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#int g0/0/0
R1(config-if)#ip add 10.0.0.1 255.255.255.252
R1(config-if)#no sh

R1(config-if)#
%LINK-5-CHANGED: Interface GigabitEthernet0/0/0, changed state to up
R1(config-if)#exit
```
```cisco
GO OUT NOW!^

User Access Verification

Password: 

R2>enable
Password: 
R2#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R2(config)#int g0/0/0
R2(config-if)#ip add 10.0.0.2 255.255.255.252
R2(config-if)#no sh

R2(config-if)#
%LINK-5-CHANGED: Interface GigabitEthernet0/0/0, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0/0, changed state to up

R2(config-if)#exit
R2(config)#
```
Задаю статическую маршрутизацию
(указываем 0 в адресе и маске, т.к. "по умолчанию")
```cisco
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2
```
```cisco
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1
```
Проверяю связность:
```cisco
R1#ping 10.0.0.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.2, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 0/0/0 ms
```
### 1.6 Базовая настройка коммутатора S1
```cisco
Switch>enable
Switch#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#no ip domain-lookup
Switch(config)#hostname S1
S1(config)#enable secret class
S1(config)#line console 0
S1(config-line)#password cisco
S1(config-line)#login
S1(config-line)#logging synchronous
S1(config-line)#exec-timeout 5 0
S1(config-line)#exit
S1(config)#line vty 0 15
S1(config-line)#password cisco
S1(config-line)#login
S1(config-line)#exec-timeout 5 0
S1(config-line)#exit
S1(config)#service password-encryption
S1(config)#banner motd ^CGO OUT NOW^C
S1(config)#clock timezone MSC 3
S1(config)#end
S1#copy running-config startup-config
%SYS-5-CONFIG_I: Configured from console by console

S1#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]
```
### 1.7 Базовая настройка коммутатора S2 
```cisco
Switch>enable
Switch#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#no ip domain-lookup
Switch(config)#hostname S2
S2(config)#enable secret class
S2(config)#line console 0
S2(config-line)#password cisco
S2(config-line)#login
S2(config-line)#logging synchronous
S2(config-line)#exec-timeout 5 0
S2(config-line)#exit
S2(config)#line vty 0 15
S2(config-line)#password cisco
S2(config-line)#login
S2(config-line)#exec-timeout 5 0
S2(config-line)#exit
S2(config)#service password-encryption
S2(config)#banner motd ^CGO OUT NOW^C
S2(config)#clock timezone MSC 3
S2(config)#end
S2#copy running-config startup-config
%SYS-5-CONFIG_I: Configured from console by console

S2#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]  
```
### 1.8 Создание VLAN на S1
```cisco
S1(config)#vlan 100
S1(config-vlan)#name Client
S1(config-vlan)#exit
S1(config)#vlan 200
S1(config-vlan)#name Managment
S1(config-vlan)#exit
S1(config)#vlan 999
S1(config-vlan)#name Parking_Lot
S1(config-vlan)#exit
S1(config)#vlan 1000
S1(config-vlan)#name Native
S1(config-vlan)#exit
S1(config)#end
S1#copy running-config startup-config
%SYS-5-CONFIG_I: Configured from console by console

S1#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]
```
### 1.9 Настройка интерфейса управления S1 (VLAN 200)
```cisco
S1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
S1(config)#int vlan 200
S1(config-if)#
%LINK-5-CHANGED: Interface Vlan200, changed state to up
S1(config-if)#ip add 192.168.1.66 255.255.255.224
S1(config-if)#no sh
S1(config-if)#exit
S1(config)#ip default-gateway 192.168.1.65
```

### 1.10 Настройка интерфейса управления S2 (VLAN 1)
```cisco
GO OUT NOW^

User Access Verification

S2>enable
Password: 
S2#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
S2(config)#interface vlan 1
S2(config-if)#ip add 192.168.1.98 255.255.255.240
S2(config-if)#no sh
S2(config-if)#exit
S2(config)#ip default-gateway 192.168.1.97
%LINK-3-UPDOWN: Interface Vlan1, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan1, changed state to up
```
### 1.11 Назначение портов и транков
S1:
```cisco
S1(config)# interface range f0/1-4, f0/7-24, g0/1-2
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 999
S1(config-if-range)# sh
%LINK-5-CHANGED: Interface FastEthernet0/1, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/2, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/3, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/4, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/7, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/8, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/9, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/10, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/11, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/12, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/13, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/14, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/15, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/16, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/17, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/18, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/19, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/20, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/21, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/22, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/23, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/24, changed state to administratively down

%LINK-5-CHANGED: Interface GigabitEthernet0/1, changed state to administratively down

%LINK-5-CHANGED: Interface GigabitEthernet0/2, changed state to administratively down
S1(config-if-range)# exit

S1(config)# interface f0/6
S1(config-if)# switchport mode access
S1(config-if)# switchport access vlan 100
S1(config-if)# no shutdown
S1(config-if)# exit

S1(config)# interface f0/5
S1(config-if)# switchport mode trunk
S1(config-if)# switchport trunk native vlan 1000
S1(config-if)# switchport trunk allowed vlan 100,200,1000
S1(config-if)# no shutdown
S1(config-if)# end
%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/5, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/5, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan200, changed state to up
```
S2 ( F0/18 для PC-B)
```cisco
S2(config)# interface range f0/1-17, f0/19-24, g0/1-2
S2(config-if-range)# sh
%LINK-5-CHANGED: Interface FastEthernet0/1, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/2, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/3, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/4, changed state to administratively down


%LINK-5-CHANGED: Interface FastEthernet0/6, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/7, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/8, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/9, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/10, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/11, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/12, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/13, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/14, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/15, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/16, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/17, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/19, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/20, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/21, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/22, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/23, changed state to administratively down

%LINK-5-CHANGED: Interface FastEthernet0/24, changed state to administratively down

%LINK-5-CHANGED: Interface GigabitEthernet0/1, changed state to administratively down

%LINK-5-CHANGED: Interface GigabitEthernet0/2, changed state to administratively down
S2(config-if-range)#
%LINK-5-CHANGED: Interface FastEthernet0/5, changed state to administratively down

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/5, changed state to down
S2(config-if-range)#interface f0/18
S2(config-if)#sw m access
S2(config-if)#sw acc vlan 1
S2(config-if)#no sh
S2(config-if)#exit
S2(config)#int f0/5
S2(config-if)#sw m acc
S2(config-if)#sw acc vlan 1
S2(config-if)#no sh

S2(config-if)#
%LINK-5-CHANGED: Interface FastEthernet0/5, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/5, changed state to up

```
### 1.12 Проверка конфигурации
```cisco
S1#sh vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
100  Client                           active    Fa0/6
200  Managment                        active    
999  Parking_Lot                      active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
                                                Fa0/7, Fa0/8, Fa0/9, Fa0/10
                                                Fa0/11, Fa0/12, Fa0/13, Fa0/14
                                                Fa0/15, Fa0/16, Fa0/17, Fa0/18
                                                Fa0/19, Fa0/20, Fa0/21, Fa0/22
                                                Fa0/23, Fa0/24, Gig0/1, Gig0/2
1000 Native                           active    
1002 fddi-default                     active    
1003 token-ring-default               active    
1004 fddinet-default                  active    
1005 trnet-default                    active    
S1#sh int trunk
Port        Mode         Encapsulation  Status        Native vlan
Fa0/5       on           802.1q         trunking      1000

Port        Vlans allowed on trunk
Fa0/5       100,200,1000

Port        Vlans allowed and active in management domain
Fa0/5       100,200,1000

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/5       100,200,1000
```
```cisco
S2#sh vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
                                                Fa0/5, Fa0/6, Fa0/7, Fa0/8
                                                Fa0/9, Fa0/10, Fa0/11, Fa0/12
                                                Fa0/13, Fa0/14, Fa0/15, Fa0/16
                                                Fa0/17, Fa0/18, Fa0/19, Fa0/20
                                                Fa0/21, Fa0/22, Fa0/23, Fa0/24
                                                Gig0/1, Gig0/2
1002 fddi-default                     active    
1003 token-ring-default               active    
1004 fddinet-default                  active    
1005 trnet-default                    active  
```
## Часть 2. Настройка и проверка DHCPv4 на R1
### 2.1 Исключение первых пяти адресов в каждой подсети
```cisco

GO OUT NOW!^

User Access Verification

Password: 

R1>enable
Password: 
R1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#ip dhcp excluded-address 192.168.1.1 192.168.1.5
R1(config)#ip dhcp excluded-address 192.168.1.97 192.168.1.101
R1(config)#
```
### 2.2 Создание пула DHCP для подсети A (клиенты VLAN 100)
```cisco
R1(config)# ip dhcp pool R1_Client_LAN
R1(dhcp-config)# network 192.168.1.0 255.255.255.192
R1(dhcp-config)# domain-name CCNA-lab.com
R1(dhcp-config)# default-router 192.168.1.1
R1(dhcp-config)# lease 2 12 30
R1(dhcp-config)# exit
```
### 2.3 Создание пула DHCP для подсети C (клиенты R2)
```cisco
R1(config)# ip dhcp pool R2_Client_LAN
R1(dhcp-config)# network 192.168.1.96 255.255.255.240
R1(dhcp-config)# domain-name CCNA-lab.com
R1(dhcp-config)# default-router 192.168.1.97
R1(dhcp-config)# lease 2 12 30
R1(dhcp-config)# end
R1#copy running-config startup-config 
Destination filename [startup-config]? 
Building configuration...
[OK]
```

### 2.4 Проверка DHCP-сервера
```cico
R1#sh ip dhcp pool

Pool R1_Client_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 62
 Leased addresses               : 0
 Excluded addresses             : 2
 Pending event                  : none

 1 subnet is currently in the pool
 Current index        IP address range                    Leased/Excluded/Total
 192.168.1.1          192.168.1.1      - 192.168.1.62      0    / 2     / 62

Pool R2_Client_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 14
 Leased addresses               : 0
 Excluded addresses             : 2
 Pending event                  : none

 1 subnet is currently in the pool
 Current index        IP address range                    Leased/Excluded/Total
 192.168.1.97         192.168.1.97     - 192.168.1.110     0    / 2     / 14
R1#sh ip dhcp binding
IP address       Client-ID/              Lease expiration        Type
                 Hardware address
R1# show ip dhcp server statistics
Memory usage          : 15419
Address pools         : 2
Automatic bindings    : 1
Manual bindings       : 0
Expired bindings      : 0
Malformed messages    : 0
Secure arp entries    : 0

Message               Received
BOOTREQUEST           0
DHCPDISCOVER          1
DHCPREQUEST           1
DHCPDECLINE           0
DHCPRELEASE           0
DHCPINFORM            0

Message               Sent
BOOTREPLY             0
DHCPOFFER             1
DHCPACK               1
DHCPNAK               0
```
PC-A:
```cmd

C:\>ipconfig /release

   IP Address......................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: 0.0.0.0
   DNS Server......................: 0.0.0.0

C:\>ipconfig /renew

   IP Address......................: 192.168.1.6
   Subnet Mask.....................: 255.255.255.192
   Default Gateway.................: 192.168.1.1
   DNS Server......................: 0.0.0.0

C:\>ipconfig /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: CCNA-lab.com
   Physical Address................: 000B.BE2B.D6D5
   Link-local IPv6 Address.........: FE80::20B:BEFF:FE2B:D6D5
   IPv6 Address....................: ::
   IPv4 Address....................: 192.168.1.6
   Subnet Mask.....................: 255.255.255.192
   Default Gateway.................: ::
                                     192.168.1.1
   DHCP Servers....................: 192.168.1.1
   DHCPv6 IAID.....................: 
   DHCPv6 Client DUID..............: 00-01-00-01-0C-47-04-14-00-0B-BE-2B-D6-D5
   DNS Servers.....................: ::
                                     0.0.0.0

Bluetooth Connection:

   Connection-specific DNS Suffix..: CCNA-lab.com
   Physical Address................: 00E0.F9E6.E356
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0
   DHCP Servers....................: 0.0.0.0
   DHCPv6 IAID.....................: 
   DHCPv6 Client DUID..............: 00-01-00-01-0C-47-04-14-00-0B-BE-2B-D6-D5
   DNS Servers.....................: ::
                                     0.0.0.0

C:\>ping 192.168.1.1

Pinging 192.168.1.1 with 32 bytes of data:

Reply from 192.168.1.1: bytes=32 time<1ms TTL=255
Reply from 192.168.1.1: bytes=32 time<1ms TTL=255
Reply from 192.168.1.1: bytes=32 time<1ms TTL=255
Reply from 192.168.1.1: bytes=32 time<1ms TTL=255

Ping statistics for 192.168.1.1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```
## Часть 3. Настройка DHCP-ретрансляции на R2
### 3.1 Включение ретрансляции на интерфейсе G0/0/1 R2
```cisco
R2#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R2(config)#int g0/0/1
R2(config-if)#ip helper-address 10.0.0.1
R2(config-if)#end
%SYS-5-CONFIG_I: Configured from console by console
R2#copy running-config startup-config
Destination filename [startup-config]? 
Building configuration...
[OK]
```
### 3.2 Получение IP-адреса на PC-B
```cmd
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: FE80::209:7CFF:FEE5:B1D4
   IPv6 Address....................: ::
   Autoconfiguration IPv4 Address..: 169.254.177.212
   Subnet Mask.....................: 255.255.0.0
   Default Gateway.................: ::
                                     0.0.0.0

Bluetooth Connection:

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0

C:\>ipconfig /release

   IP Address......................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: 0.0.0.0
   DNS Server......................: 0.0.0.0


C:\>ipconfig /renew

   IP Address......................: 192.168.1.102
   Subnet Mask.....................: 255.255.255.240
   Default Gateway.................: 192.168.1.97
   DNS Server......................: 0.0.0.0
C:\>ipconfig /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: CCNA-lab.com
   Physical Address................: 0009.7CE5.B1D4
   Link-local IPv6 Address.........: FE80::209:7CFF:FEE5:B1D4
   IPv6 Address....................: ::
   IPv4 Address....................: 192.168.1.102
   Subnet Mask.....................: 255.255.255.240
   Default Gateway.................: ::
                                     192.168.1.97
   DHCP Servers....................: 10.0.0.1
   DHCPv6 IAID.....................: 
   DHCPv6 Client DUID..............: 00-01-00-01-52-CA-00-E3-00-09-7C-E5-B1-D4
   DNS Servers.....................: ::
                                     0.0.0.0

Bluetooth Connection:

   Connection-specific DNS Suffix..: CCNA-lab.com
   Physical Address................: 0060.2FA2.4187
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0
   DHCP Servers....................: 0.0.0.0
   DHCPv6 IAID.....................: 
   DHCPv6 Client DUID..............: 00-01-00-01-52-CA-00-E3-00-09-7C-E5-B1-D4
   DNS Servers.....................: ::
                                     0.0.0.0
```
Проверка связности:
```cmd
C:\>ping 192.168.1.1

Pinging 192.168.1.1 with 32 bytes of data:

Reply from 192.168.1.1: bytes=32 time<1ms TTL=254
Reply from 192.168.1.1: bytes=32 time<1ms TTL=254
Reply from 192.168.1.1: bytes=32 time<1ms TTL=254
Reply from 192.168.1.1: bytes=32 time<1ms TTL=254

Ping statistics for 192.168.1.1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```
### 3.3 Проверка привязок DHCP на R1
```cisco
GO OUT NOW!^

User Access Verification

Password: 

R1>enable
Password: 
R1#sh ip dhcp binding 
IP address       Client-ID/              Lease expiration        Type
                 Hardware address
192.168.1.6      000B.BE2B.D6D5           --                     Automatic
192.168.1.102    0009.7CE5.B1D4           --                     Automatic
R1# show ip dhcp server statistics
Memory usage          : 15419
Address pools         : 2
Automatic bindings    : 2
Manual bindings       : 0
Expired bindings      : 0
Malformed messages    : 0
Secure arp entries    : 0

Message               Received
BOOTREQUEST           0
DHCPDISCOVER          2
DHCPREQUEST           2
DHCPDECLINE           0
DHCPRELEASE           0
DHCPINFORM            0
Message               Sent
BOOTREPLY             0
DHCPOFFER             2
DHCPACK               2
DHCPNAK               0
```
## Вопросы для повторения
