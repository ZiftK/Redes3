# Configuración Sucursal 2 - Cisco Packet Tracer

## Topología de Red - Sucursal 2

```
                    Internet
                        │
                   [Router WAN]
                   IP Pública
                        │
                 [Router Firewall/VPN]
              GW: 192.168.20.x (por VLAN)
                        │
                 [Core Switch L3]
              SVIs: 192.168.20.x/yy
                        │
         ┌──────────────┼──────────────┐
         │              │              │
    [Switch 1]     [Switch 2]      [NVR Local]
     PoE+           PoE+          VLAN 25
      │              │
   PCs/VoIP      Cámaras IP
   
   7x Access Points (WiFi) ── VLAN 27 (Guest)
```

---

## 1. INVENTARIO DE EQUIPOS - SUCURSAL 2

### Equipos de Red Requeridos en Packet Tracer

| Cantidad | Tipo | Modelo Packet Tracer | Descripción |
|----------|------|---------------------|-------------|
| 1 | Router | 2911 o ISR 1100 | Router VPN Cliente |
| 1 | Switch L3 | 3560-24PS | Core Switch (Enrutamiento VLANs) |
| 2 | Switch L2 | 2960-24TT | Access Switches con PoE |
| 7 | Access Point | AccessPoint-PT | APs WiFi para clientes |
| 1 | Servidor | Server-PT | NVR (cámaras) |
| 18 | PC | PC-PT | Computadoras de usuario |
| 16 | IP Phone | IP Phone 7960 | Teléfonos VoIP |
| 29 | Cámara IP | Generic (PC) | Cámaras de seguridad |
| 2 | Cajero ATM | Server-PT | Cajeros automáticos |

---

## 2. TABLA DE VLANs - SUCURSAL 2

| VLAN | Nombre | Subred | Gateway | Hosts | Dispositivos |
|------|--------|--------|---------|-------|--------------|
| 21 | GERENCIA_S2 | 192.168.20.0/27 | 192.168.20.1 | 30 | 12 PCs + 12 VoIP |
| 22 | CAJAS_S2 | 192.168.20.32/29 | 192.168.20.33 | 6 | 3 PCs + 3 VoIP |
| 23 | RECEPCION_S2 | 192.168.20.40/29 | 192.168.20.41 | 6 | 2 PCs + 2 VoIP |
| 24 | SEGURIDAD_S2 | 192.168.20.48/29 | 192.168.20.49 | 6 | 1 PC + 1 VoIP |
| 25 | CAMARAS_S2 | 192.168.20.64/26 | 192.168.20.65 | 62 | 29 cámaras + NVR |
| 26 | CAJEROS_S2 | 192.168.20.128/29 | 192.168.20.129 | 6 | 2 cajeros ATM |
| 27 | CLIENTES_S2 | 192.168.20.192/26 | 192.168.20.193 | 62 | WiFi Guest (7 APs) |
| 28 | MGMT_S2 | 192.168.20.136/29 | 192.168.20.137 | 6 | Gestión switches |
| 999 | NATIVE | - | - | - | VLAN nativa (trunk) |

---

## 3. CONFIGURACIÓN ROUTER VPN - SUCURSAL 2

### Router: SUCURSAL2-RTR-01
**Modelo:** Cisco 2911 o ISR 1100

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL2-RTR-01
no ip domain-lookup
service password-encryption
security passwords min-length 10

banner motd #
*********************************************************
*  ACCESO RESTRINGIDO - SUCURSAL 2                     *
*  Acceso no autorizado está prohibido                  *
*********************************************************
#

! ==========================================
! INTERFACES
! ==========================================

! Interfaz WAN (hacia Internet)
interface GigabitEthernet0/0
 description WAN - Internet ISP
 ip address dhcp
 ip nat outside
 no shutdown

! Interfaz LAN (hacia Core Switch)
interface GigabitEthernet0/1
 description LAN - Core Switch Connection
 ip address 192.168.20.254 255.255.255.0
 ip nat inside
 no shutdown

! ==========================================
! NAT - SALIDA A INTERNET
! ==========================================
access-list 1 permit 192.168.20.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! ==========================================
! RUTAS ESTÁTICAS
! ==========================================
! Ruta a Central vía túnel VPN
ip route 10.1.0.0 255.255.0.0 172.16.0.5

! Rutas a otras sucursales (vía Central)
ip route 192.168.10.0 255.255.255.0 172.16.0.5
ip route 192.168.30.0 255.255.255.0 172.16.0.5

! ==========================================
! VPN IPSEC - HACIA CENTRAL
! ==========================================

! Fase 1 - ISAKMP Policy
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 28800

! Pre-shared key (debe coincidir con Central)
crypto isakmp key SuperSecretVPN123 address 0.0.0.0

! Fase 2 - IPSec Transform Set
crypto ipsec transform-set VPN-SET esp-aes 256 esp-sha256-hmac
 mode tunnel

! ACL para tráfico interesante (Sucursal 2 <-> Central)
access-list 100 permit ip 192.168.20.0 0.0.0.255 10.1.0.0 0.0.255.255

! Crypto Map
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 0.0.0.0  ! IP pública de Central (configurar real)
 set transform-set VPN-SET
 match address 100

! Aplicar crypto map a interfaz WAN
interface GigabitEthernet0/0
 crypto map VPN-MAP

! ==========================================
! ACLs DE SEGURIDAD
! ==========================================

! Proteger VLAN Cajeros - Solo a servidores Central
ip access-list extended ACL-CAJEROS
 permit tcp 192.168.20.128 0.0.0.7 10.1.100.0 0.0.0.15 eq 443
 permit icmp 192.168.20.128 0.0.0.7 10.1.100.0 0.0.0.15 echo
 deny   ip 192.168.20.128 0.0.0.7 any log
 permit ip any any

! Proteger VLAN Management - Solo desde TI Central
ip access-list extended ACL-MGMT
 permit tcp 10.1.20.0 0.0.0.31 192.168.20.136 0.0.0.7 eq 22
 permit tcp 10.1.20.0 0.0.0.31 192.168.20.136 0.0.0.7 eq 443
 deny   ip any 192.168.20.136 0.0.0.7 log
 permit ip any any

! WiFi Guest - Solo Internet, sin acceso interno
ip access-list extended ACL-WIFI-GUEST
 deny   ip 192.168.20.192 0.0.0.63 192.168.20.0 0.0.0.255 log
 deny   ip 192.168.20.192 0.0.0.63 10.1.0.0 0.0.255.255 log
 permit ip 192.168.20.192 0.0.0.63 any

! ==========================================
! SEGURIDAD BÁSICA
! ==========================================
no ip http server
no ip http secure-server
no cdp run

enable secret Cisco123Suc2!
username admin privilege 15 secret AdminSuc2Pass!

line console 0
 password ConsoleSuc2!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0

! SSH
ip domain-name sucursal2.banco.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

! ==========================================
! NTP - Sincronizar con Central
! ==========================================
ntp server 10.1.100.8
clock timezone CST -6

! ==========================================
! LOGGING
! ==========================================
logging buffered 16384
logging console warnings
logging trap informational
logging host 10.1.100.10

! ==========================================
! DHCP LOCAL (Redundancia)
! ==========================================
ip dhcp excluded-address 192.168.20.1 192.168.20.9
ip dhcp excluded-address 192.168.20.33 192.168.20.33
ip dhcp excluded-address 192.168.20.41 192.168.20.41
ip dhcp excluded-address 192.168.20.49 192.168.20.49

ip dhcp pool GERENCIA-LOCAL
 network 192.168.20.0 255.255.255.224
 default-router 192.168.20.1
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool CAJAS-LOCAL
 network 192.168.20.32 255.255.255.248
 default-router 192.168.20.33
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool RECEPCION-LOCAL
 network 192.168.20.40 255.255.255.248
 default-router 192.168.20.41
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool WIFI-GUEST
 network 192.168.20.192 255.255.255.192
 default-router 192.168.20.193
 dns-server 8.8.8.8 1.1.1.1
 lease 0 2

end
write memory
```

---

## 4. CONFIGURACIÓN CORE SWITCH L3 - SUCURSAL 2

### Switch: SUCURSAL2-CORE-SW01
**Modelo:** Cisco 3560-24PS

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL2-CORE-SW01
no ip domain-lookup
service password-encryption

banner motd #
*********************************************************
*  CORE SWITCH - SUCURSAL 2                            *
*  Acceso solo para personal TI autorizado             *
*********************************************************
#

! ==========================================
! HABILITAR ENRUTAMIENTO IP
! ==========================================
ip routing

! ==========================================
! CREAR VLANs
! ==========================================
vlan 21
 name GERENCIA_S2
vlan 22
 name CAJAS_S2
vlan 23
 name RECEPCION_S2
vlan 24
 name SEGURIDAD_S2
vlan 25
 name CAMARAS_S2
vlan 26
 name CAJEROS_S2
vlan 27
 name CLIENTES_S2
vlan 28
 name MGMT_S2
vlan 999
 name NATIVE

! ==========================================
! CONFIGURAR INTERFACES VLAN (SVIs)
! ==========================================

! VLAN 21 - Gerencia
interface Vlan21
 description Gateway VLAN Gerencia
 ip address 192.168.20.1 255.255.255.224
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 22 - Cajas
interface Vlan22
 description Gateway VLAN Cajas
 ip address 192.168.20.33 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 23 - Recepción
interface Vlan23
 description Gateway VLAN Recepcion
 ip address 192.168.20.41 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 24 - Seguridad
interface Vlan24
 description Gateway VLAN Seguridad
 ip address 192.168.20.49 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 25 - Cámaras
interface Vlan25
 description Gateway VLAN Camaras
 ip address 192.168.20.65 255.255.255.192
 no ip helper-address
 no shutdown

! VLAN 26 - Cajeros ATM
interface Vlan26
 description Gateway VLAN Cajeros
 ip address 192.168.20.129 255.255.255.248
 no ip helper-address
 no shutdown

! VLAN 27 - WiFi Guest
interface Vlan27
 description Gateway VLAN WiFi Guest
 ip address 192.168.20.193 255.255.255.192
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 28 - Management
interface Vlan28
 description Gateway VLAN Management
 ip address 192.168.20.137 255.255.255.248
 no ip helper-address
 no shutdown

! ==========================================
! RUTA PREDETERMINADA
! ==========================================
ip route 0.0.0.0 0.0.0.0 192.168.20.254

! ==========================================
! UPLINK AL ROUTER (TRUNK)
! ==========================================
interface GigabitEthernet0/1
 description Uplink to SUCURSAL2-RTR-01
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 21-28,999
 no shutdown

! ==========================================
! ENLACES A SWITCHES DE ACCESO (TRUNK)
! ==========================================
interface range GigabitEthernet0/2-3
 description Trunk to Access Switches
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 21-28,999
 spanning-tree portfast trunk
 no shutdown

! ==========================================
! PUERTO NVR (VLAN 25 - Cámaras)
! ==========================================
interface GigabitEthernet0/5
 description NVR Local
 switchport mode access
 switchport access vlan 25
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS CAJEROS ATM (VLAN 26)
! ==========================================
interface range GigabitEthernet0/6-7
 description Cajeros ATM
 switchport mode access
 switchport access vlan 26
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! SPANNING TREE
! ==========================================
spanning-tree mode rapid-pvst
spanning-tree extend system-id
spanning-tree vlan 21-28 priority 4096
spanning-tree portfast bpduguard default

! ==========================================
! DHCP SNOOPING
! ==========================================
ip dhcp snooping
ip dhcp snooping vlan 21,22,23,24,27
no ip dhcp snooping information option

interface GigabitEthernet0/1
 ip dhcp snooping trust

! ==========================================
! DYNAMIC ARP INSPECTION
! ==========================================
ip arp inspection vlan 21,22,26
ip arp inspection validate src-mac dst-mac ip

interface GigabitEthernet0/1
 ip arp inspection trust

! ==========================================
! QoS PARA VOIP
! ==========================================
mls qos
mls qos map cos-dscp 0 8 16 24 32 46 48 56

! ==========================================
! SEGURIDAD
! ==========================================
enable secret Cisco123Suc2!
username admin privilege 15 secret AdminSuc2Pass!

line console 0
 password ConsoleSuc2!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name sucursal2.banco.local
crypto key generate rsa modulus 2048
ip ssh version 2

! ==========================================
! NTP y LOGGING
! ==========================================
ntp server 10.1.100.8
clock timezone CST -6

logging buffered 16384
logging host 10.1.100.10

! ==========================================
! SNMP
! ==========================================
snmp-server community ReadOnly RO
snmp-server location "Sucursal 2 - Sala Equipos"
snmp-server contact "TI - ti@banco.local"

end
write memory
```

---

## 5. CONFIGURACIÓN SWITCHES DE ACCESO

### Switch: SUCURSAL2-SW-ACC-01
**Modelo:** Cisco 2960-24TT

```cisco
enable
configure terminal

hostname SUCURSAL2-SW-ACC-01
no ip domain-lookup
service password-encryption

! Crear VLANs
vlan 21
 name GERENCIA_S2
vlan 22
 name CAJAS_S2
vlan 23
 name RECEPCION_S2
vlan 24
 name SEGURIDAD_S2
vlan 27
 name CLIENTES_S2
vlan 28
 name MGMT_S2
vlan 999
 name NATIVE

! Configuración de gestión
interface Vlan28
 description Management Interface
 ip address 192.168.20.138 255.255.255.248
 no shutdown

ip default-gateway 192.168.20.137

! Uplink al Core Switch (TRUNK)
interface GigabitEthernet0/1
 description Uplink to SUCURSAL2-CORE-SW01
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 21-24,27,28,999
 no shutdown

! Puertos Gerencia (VLAN 21)
interface range FastEthernet0/2-13
 description VLAN 21 - Gerencia PCs
 switchport mode access
 switchport access vlan 21
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Puertos VoIP Gerencia
interface range FastEthernet0/14-24
 description VoIP Gerencia
 switchport mode access
 switchport access vlan 21
 switchport voice vlan 21
 mls qos trust cos
 spanning-tree portfast
 no shutdown

! DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 21,22,23,24,27
no ip dhcp snooping information option

interface GigabitEthernet0/1
 ip dhcp snooping trust

! Spanning Tree
spanning-tree mode rapid-pvst
spanning-tree portfast bpduguard default

! Seguridad
enable secret Cisco123Suc2!
username admin privilege 15 secret AdminSuc2Pass!

line console 0
 password ConsoleSuc2!
 login local
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name sucursal2.banco.local
crypto key generate rsa modulus 2048
ip ssh version 2

ntp server 10.1.100.8

end
write memory
```

### Switch: SUCURSAL2-SW-ACC-02

```cisco
hostname SUCURSAL2-SW-ACC-02

interface Vlan28
 ip address 192.168.20.139 255.255.255.248

! Puertos para Cajas (VLAN 22)
interface range FastEthernet0/2-4
 description VLAN 22 - Cajas
 switchport mode access
 switchport access vlan 22

! Puertos para Recepción (VLAN 23)
interface range FastEthernet0/5-6
 description VLAN 23 - Recepcion
 switchport mode access
 switchport access vlan 23

! Puerto para Seguridad (VLAN 24)
interface FastEthernet0/7
 description VLAN 24 - Seguridad
 switchport mode access
 switchport access vlan 24

! Puertos para Cámaras (VLAN 25)
interface range FastEthernet0/8-24
 description VLAN 25 - Camaras IP
 switchport mode access
 switchport access vlan 25
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict

! (Resto similar a SW-ACC-01)
```

---

## 6. CONFIGURACIÓN ACCESS POINTS (WiFi Guest)

### Access Point: AP-S2-01 hasta AP-S2-07

**Config > Port 1:**
- **SSID:** Banco_Guest_S2
- **Authentication:** WPA2-PSK
- **PSK Pass Phrase:** GuestWiFi2025!
- **Encryption Type:** AES

**GUI:**
- VLAN: 27
- IP Addresses: 192.168.20.200-206
- Subnet Mask: 255.255.255.192
- Gateway: 192.168.20.193

---

## 7. ASIGNACIÓN DE IPs ESTÁTICAS

### VLAN 25 - Cámaras
```
192.168.20.65 - Gateway
192.168.20.66-94 - Cámaras 1-29
192.168.20.95 - NVR Local
```

### VLAN 26 - Cajeros ATM
```
192.168.20.129 - Gateway
192.168.20.130 - Cajero ATM 1
192.168.20.131 - Cajero ATM 2
```

### VLAN 28 - Management
```
192.168.20.137 - Gateway
192.168.20.138 - SUCURSAL2-SW-ACC-01
192.168.20.139 - SUCURSAL2-SW-ACC-02
192.168.20.140 - SUCURSAL2-CORE-SW01
192.168.20.141 - SUCURSAL2-RTR-01
```

### VLAN 27 - Access Points
```
192.168.20.193 - Gateway
192.168.20.200-206 - APs (7 unidades)
192.168.20.194-250 - Pool DHCP clientes WiFi
```

---

## 8. VERIFICACIÓN Y TESTING

### Pruebas Esenciales:
1. `ping 10.1.100.1` - Ping a Central vía VPN
2. `ping 8.8.8.8` - Salida a Internet
3. Llamadas VoIP a Central (extensiones 1001-1007)
4. Acceso WiFi Guest sin acceso a red interna
5. Cajeros solo acceden a 10.1.100.0/28

---

## 9. CHECKLIST DE IMPLEMENTACIÓN

- [ ] Router configurado con VPN
- [ ] VLANs creadas
- [ ] SVIs configuradas
- [ ] Trunks establecidos
- [ ] Puertos de acceso asignados
- [ ] IPs estáticas configuradas
- [ ] Access Points configurados
- [ ] DHCP Relay configurado
- [ ] Port Security habilitado
- [ ] ACLs de seguridad aplicadas
- [ ] VPN con Central funcional
- [ ] VoIP operativo
- [ ] Configuraciones guardadas

---

**Última actualización:** Noviembre 2025
**Sucursal:** Sucursal 2
