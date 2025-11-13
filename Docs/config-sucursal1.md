# Configuración Sucursal 1 - Cisco Packet Tracer

## Topología de Red - Sucursal 1

```
                    Internet
                        │
                   [Router WAN]
                   IP Pública
                        │
                 [Router Firewall/VPN]
              GW: 192.168.10.x (por VLAN)
                        │
                 [Core Switch L3]
              SVIs: 192.168.10.x/yy
                        │
         ┌──────────────┼──────────────┐
         │              │              │
    [Switch 1]     [Switch 2]      [NVR Local]
     PoE+           PoE+          VLAN 15
      │              │
   PCs/VoIP      Cámaras IP
   
   7x Access Points (WiFi) ── VLAN 17 (Guest)
```

---

## 1. INVENTARIO DE EQUIPOS - SUCURSAL 1

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

## 2. TABLA DE VLANs - SUCURSAL 1

| VLAN | Nombre | Subred | Gateway | Hosts | Dispositivos |
|------|--------|--------|---------|-------|--------------|
| 11 | GERENCIA_S1 | 192.168.10.0/27 | 192.168.10.1 | 30 | 12 PCs + 12 VoIP |
| 12 | CAJAS_S1 | 192.168.10.32/29 | 192.168.10.33 | 6 | 3 PCs + 3 VoIP |
| 13 | RECEPCION_S1 | 192.168.10.40/29 | 192.168.10.41 | 6 | 2 PCs + 2 VoIP |
| 14 | SEGURIDAD_S1 | 192.168.10.48/29 | 192.168.10.49 | 6 | 1 PC + 1 VoIP |
| 15 | CAMARAS_S1 | 192.168.10.64/26 | 192.168.10.65 | 62 | 29 cámaras + NVR |
| 16 | CAJEROS_S1 | 192.168.10.128/29 | 192.168.10.129 | 6 | 2 cajeros ATM |
| 17 | CLIENTES_S1 | 192.168.10.192/26 | 192.168.10.193 | 62 | WiFi Guest (7 APs) |
| 18 | MGMT_S1 | 192.168.10.136/29 | 192.168.10.137 | 6 | Gestión switches |
| 999 | NATIVE | - | - | - | VLAN nativa (trunk) |

---

## 3. CONFIGURACIÓN ROUTER VPN - SUCURSAL 1

### Router: SUCURSAL1-RTR-01
**Modelo:** Cisco 2911 o ISR 1100

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL1-RTR-01
no ip domain-lookup
service password-encryption
security passwords min-length 10

! Banner
banner motd #
*********************************************************
*  ACCESO RESTRINGIDO - SUCURSAL 1                     *
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
 ip address 192.168.10.254 255.255.255.0
 ip nat inside
 no shutdown

! ==========================================
! NAT - SALIDA A INTERNET
! ==========================================
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! ==========================================
! RUTAS ESTÁTICAS
! ==========================================
! Ruta a Central vía túnel VPN
ip route 10.1.0.0 255.255.0.0 172.16.0.1

! Rutas a otras sucursales (vía Central)
ip route 192.168.20.0 255.255.255.0 172.16.0.1
ip route 192.168.30.0 255.255.255.0 172.16.0.1

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

! ACL para tráfico interesante (Sucursal 1 <-> Central)
access-list 100 permit ip 192.168.10.0 0.0.0.255 10.1.0.0 0.0.255.255

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
 permit tcp 192.168.10.128 0.0.0.7 10.1.100.0 0.0.0.15 eq 443
 permit icmp 192.168.10.128 0.0.0.7 10.1.100.0 0.0.0.15 echo
 deny   ip 192.168.10.128 0.0.0.7 any log
 permit ip any any

! Proteger VLAN Management - Solo desde TI Central
ip access-list extended ACL-MGMT
 permit tcp 10.1.20.0 0.0.0.31 192.168.10.136 0.0.0.7 eq 22
 permit tcp 10.1.20.0 0.0.0.31 192.168.10.136 0.0.0.7 eq 443
 deny   ip any 192.168.10.136 0.0.0.7 log
 permit ip any any

! WiFi Guest - Solo Internet, sin acceso interno
ip access-list extended ACL-WIFI-GUEST
 deny   ip 192.168.10.192 0.0.0.63 192.168.10.0 0.0.0.255 log
 deny   ip 192.168.10.192 0.0.0.63 10.1.0.0 0.0.255.255 log
 permit ip 192.168.10.192 0.0.0.63 any

! ==========================================
! SEGURIDAD BÁSICA
! ==========================================
no ip http server
no ip http secure-server
no cdp run

enable secret Cisco123Suc1!
username admin privilege 15 secret AdminSuc1Pass!

line console 0
 password ConsoleSuc1!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0

! SSH
ip domain-name sucursal1.banco.local
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
! DHCP LOCAL (Opcional - Redundancia)
! ==========================================
! Si VPN cae, DHCP local mantiene operación
ip dhcp excluded-address 192.168.10.1 192.168.10.9
ip dhcp excluded-address 192.168.10.33 192.168.10.33
ip dhcp excluded-address 192.168.10.41 192.168.10.41
ip dhcp excluded-address 192.168.10.49 192.168.10.49

ip dhcp pool GERENCIA-LOCAL
 network 192.168.10.0 255.255.255.224
 default-router 192.168.10.1
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool CAJAS-LOCAL
 network 192.168.10.32 255.255.255.248
 default-router 192.168.10.33
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool RECEPCION-LOCAL
 network 192.168.10.40 255.255.255.248
 default-router 192.168.10.41
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool WIFI-GUEST
 network 192.168.10.192 255.255.255.192
 default-router 192.168.10.193
 dns-server 8.8.8.8 1.1.1.1
 lease 0 2

end
write memory
```

---

## 4. CONFIGURACIÓN CORE SWITCH L3 - SUCURSAL 1

### Switch: SUCURSAL1-CORE-SW01
**Modelo:** Cisco 3560-24PS

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL1-CORE-SW01
no ip domain-lookup
service password-encryption

banner motd #
*********************************************************
*  CORE SWITCH - SUCURSAL 1                            *
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
vlan 11
 name GERENCIA_S1
vlan 12
 name CAJAS_S1
vlan 13
 name RECEPCION_S1
vlan 14
 name SEGURIDAD_S1
vlan 15
 name CAMARAS_S1
vlan 16
 name CAJEROS_S1
vlan 17
 name CLIENTES_S1
vlan 18
 name MGMT_S1
vlan 999
 name NATIVE

! ==========================================
! CONFIGURAR INTERFACES VLAN (SVIs)
! ==========================================

! VLAN 11 - Gerencia
interface Vlan11
 description Gateway VLAN Gerencia
 ip address 192.168.10.1 255.255.255.224
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 12 - Cajas
interface Vlan12
 description Gateway VLAN Cajas
 ip address 192.168.10.33 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 13 - Recepción
interface Vlan13
 description Gateway VLAN Recepcion
 ip address 192.168.10.41 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 14 - Seguridad
interface Vlan14
 description Gateway VLAN Seguridad
 ip address 192.168.10.49 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 15 - Cámaras
interface Vlan15
 description Gateway VLAN Camaras
 ip address 192.168.10.65 255.255.255.192
 no ip helper-address
 no shutdown

! VLAN 16 - Cajeros ATM
interface Vlan16
 description Gateway VLAN Cajeros
 ip address 192.168.10.129 255.255.255.248
 no ip helper-address
 no shutdown

! VLAN 17 - WiFi Guest
interface Vlan17
 description Gateway VLAN WiFi Guest
 ip address 192.168.10.193 255.255.255.192
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 18 - Management
interface Vlan18
 description Gateway VLAN Management
 ip address 192.168.10.137 255.255.255.248
 no ip helper-address
 no shutdown

! ==========================================
! RUTA PREDETERMINADA
! ==========================================
ip route 0.0.0.0 0.0.0.0 192.168.10.254

! ==========================================
! UPLINK AL ROUTER (TRUNK)
! ==========================================
interface GigabitEthernet0/1
 description Uplink to SUCURSAL1-RTR-01
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 11-18,999
 no shutdown

! ==========================================
! ENLACES A SWITCHES DE ACCESO (TRUNK)
! ==========================================
interface range GigabitEthernet0/2-3
 description Trunk to Access Switches
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 11-18,999
 spanning-tree portfast trunk
 no shutdown

! ==========================================
! PUERTO NVR (VLAN 15 - Cámaras)
! ==========================================
interface GigabitEthernet0/5
 description NVR Local
 switchport mode access
 switchport access vlan 15
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS CAJEROS ATM (VLAN 16)
! ==========================================
interface range GigabitEthernet0/6-7
 description Cajeros ATM
 switchport mode access
 switchport access vlan 16
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
spanning-tree vlan 11-18 priority 4096
spanning-tree portfast bpduguard default

! ==========================================
! DHCP SNOOPING
! ==========================================
ip dhcp snooping
ip dhcp snooping vlan 11,12,13,14,17
no ip dhcp snooping information option

! Trust uplink
interface GigabitEthernet0/1
 ip dhcp snooping trust

! ==========================================
! DYNAMIC ARP INSPECTION
! ==========================================
ip arp inspection vlan 11,12,16
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
enable secret Cisco123Suc1!
username admin privilege 15 secret AdminSuc1Pass!

line console 0
 password ConsoleSuc1!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name sucursal1.banco.local
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
snmp-server location "Sucursal 1 - Sala Equipos"
snmp-server contact "TI - ti@banco.local"

end
write memory
```

---

## 5. CONFIGURACIÓN SWITCHES DE ACCESO

### Switch: SUCURSAL1-SW-ACC-01
**Modelo:** Cisco 2960-24TT

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL1-SW-ACC-01
no ip domain-lookup
service password-encryption

! ==========================================
! CREAR VLANs
! ==========================================
vlan 11
 name GERENCIA_S1
vlan 12
 name CAJAS_S1
vlan 13
 name RECEPCION_S1
vlan 14
 name SEGURIDAD_S1
vlan 17
 name CLIENTES_S1
vlan 18
 name MGMT_S1
vlan 999
 name NATIVE

! ==========================================
! CONFIGURACIÓN DE GESTIÓN
! ==========================================
interface Vlan18
 description Management Interface
 ip address 192.168.10.138 255.255.255.248
 no shutdown

ip default-gateway 192.168.10.137

! ==========================================
! UPLINK AL CORE SWITCH (TRUNK)
! ==========================================
interface GigabitEthernet0/1
 description Uplink to SUCURSAL1-CORE-SW01
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 11-14,17,18,999
 no shutdown

! ==========================================
! PUERTOS GERENCIA (VLAN 11)
! ==========================================
interface range FastEthernet0/2-13
 description VLAN 11 - Gerencia PCs
 switchport mode access
 switchport access vlan 11
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS VOIP GERENCIA (VLAN 11 datos + VoIP)
! ==========================================
interface range FastEthernet0/14-24
 description VoIP Gerencia
 switchport mode access
 switchport access vlan 11
 switchport voice vlan 11
 mls qos trust cos
 spanning-tree portfast
 no shutdown

! ==========================================
! DHCP SNOOPING
! ==========================================
ip dhcp snooping
ip dhcp snooping vlan 11,12,13,14,17
no ip dhcp snooping information option

interface GigabitEthernet0/1
 ip dhcp snooping trust

! ==========================================
! SPANNING TREE
! ==========================================
spanning-tree mode rapid-pvst
spanning-tree portfast bpduguard default

! ==========================================
! SEGURIDAD
! ==========================================
enable secret Cisco123Suc1!
username admin privilege 15 secret AdminSuc1Pass!

line console 0
 password ConsoleSuc1!
 login local
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name sucursal1.banco.local
crypto key generate rsa modulus 2048
ip ssh version 2

ntp server 10.1.100.8

end
write memory
```

### Switch: SUCURSAL1-SW-ACC-02
**Similar a SW-ACC-01, con las siguientes diferencias:**

```cisco
hostname SUCURSAL1-SW-ACC-02

interface Vlan18
 ip address 192.168.10.139 255.255.255.248

! Puertos para Cajas (VLAN 12)
interface range FastEthernet0/2-4
 description VLAN 12 - Cajas
 switchport mode access
 switchport access vlan 12

! Puertos para Recepción (VLAN 13)
interface range FastEthernet0/5-6
 description VLAN 13 - Recepcion
 switchport mode access
 switchport access vlan 13

! Puerto para Seguridad (VLAN 14)
interface FastEthernet0/7
 description VLAN 14 - Seguridad
 switchport mode access
 switchport access vlan 14

! Puertos para Cámaras (VLAN 15)
interface range FastEthernet0/8-24
 description VLAN 15 - Camaras IP
 switchport mode access
 switchport access vlan 15
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
```

---

## 6. CONFIGURACIÓN ACCESS POINTS (WiFi Guest)

### Access Point: AP-S1-01 hasta AP-S1-07
**Modelo:** AccessPoint-PT

**Configuración en Packet Tracer:**

#### Config > Port 1:
- **SSID:** Banco_Guest_S1
- **Authentication:** WPA2-PSK
- **PSK Pass Phrase:** GuestWiFi2025!
- **Encryption Type:** AES
- **Channel:** Auto (o distribuir: 1, 6, 11)

#### GUI:
- VLAN: 17
- DHCP: Disabled (el AP debe tener IP estática VLAN 17)
- IP Address: 192.168.10.200-206 (uno por AP)
- Subnet Mask: 255.255.255.192
- Gateway: 192.168.10.193

**Conexión física:**
- Conectar APs a switches en puertos configurados como access VLAN 17

---

## 7. ASIGNACIÓN DE IPs ESTÁTICAS

### VLAN 15 - Cámaras (Todas estáticas)
```
192.168.10.65 - Gateway
192.168.10.66-94 - Cámaras 1-29
192.168.10.95 - NVR Local
```

### VLAN 16 - Cajeros ATM (Estáticas)
```
192.168.10.129 - Gateway
192.168.10.130 - Cajero ATM 1
192.168.10.131 - Cajero ATM 2
```

### VLAN 18 - Management (Estáticas)
```
192.168.10.137 - Gateway
192.168.10.138 - SUCURSAL1-SW-ACC-01
192.168.10.139 - SUCURSAL1-SW-ACC-02
192.168.10.140 - SUCURSAL1-CORE-SW01
192.168.10.141 - SUCURSAL1-RTR-01 (gestión)
```

### VLAN 17 - Access Points (Estáticas)
```
192.168.10.193 - Gateway
192.168.10.200 - AP-S1-01
192.168.10.201 - AP-S1-02
192.168.10.202 - AP-S1-03
192.168.10.203 - AP-S1-04
192.168.10.204 - AP-S1-05
192.168.10.205 - AP-S1-06
192.168.10.206 - AP-S1-07
192.168.10.194-250 - Pool DHCP para clientes WiFi
```

---

## 8. CONFIGURACIÓN DE PCs Y TELÉFONOS

### PCs de Gerencia (VLAN 11)
**Desktop > IP Configuration:**
- DHCP habilitado
- Verificar rango: 192.168.10.2-27

### PCs de Cajas (VLAN 12)
- DHCP habilitado
- Rango: 192.168.10.34-38

### PCs de Recepción (VLAN 13)
- DHCP habilitado
- Rango: 192.168.10.42-45

### Teléfonos VoIP
**Config > GUI:**
- DHCP: Enabled
- Call Manager: 10.1.50.2 (PBX Central vía VPN)
- Extensions: 2001-2016 (Sucursal 1)

### Cajeros ATM (Configurar como Servidores)
**Desktop > IP Configuration:**
- IP: 192.168.10.130 (ATM-1), 192.168.10.131 (ATM-2)
- Mask: 255.255.255.248
- Gateway: 192.168.10.129
- DNS: 10.1.100.8 (vía VPN)

---

## 9. DIAGRAMA DE CABLEADO

```
SUCURSAL1-RTR-01 Gi0/1 ──── Gi0/1 SUCURSAL1-CORE-SW01
                                      │
                        ┌─────────────┼─────────────┐
                        │                           │
                       Gi0/2                      Gi0/3
                        │                           │
                SW-ACC-01                      SW-ACC-02
                Fa0/2-24                       Fa0/2-24
                    │                              │
              ┌─────┴─────┐                  ┌─────┴─────┐
         PCs Gerencia  VoIP             Cajas/Recep   Cámaras
         (VLAN 11)    (VLAN 11)         (VLAN 12,13)  (VLAN 15)

SUCURSAL1-CORE-SW01:
  - Gi0/5: NVR (VLAN 15)
  - Gi0/6-7: Cajeros ATM (VLAN 16)

Access Points (7x) conectados a VLAN 17
```

---

## 10. VERIFICACIÓN Y TESTING

### En Router:
```
show ip interface brief
show ip route
show crypto isakmp sa
show crypto ipsec sa
ping 10.1.100.1  ! Ping a Central vía VPN
```

### En Core Switch:
```
show vlan brief
show ip interface brief
show ip route
show spanning-tree summary
show ip dhcp snooping binding
```

### En Switches de Acceso:
```
show vlan brief
show interfaces status
show port-security
```

### Pruebas de Conectividad:
1. Ping entre VLANs locales
2. Ping a Central (10.1.100.1) vía VPN
3. Ping a Internet (8.8.8.8)
4. Llamadas VoIP a Central
5. Acceso WiFi Guest (sin acceso a redes internas)
6. Verificar cajeros solo acceden a servidores Central

---

## 11. CHECKLIST DE IMPLEMENTACIÓN

- [ ] Configurar Router con VPN hacia Central
- [ ] Configurar VLANs en Core Switch
- [ ] Configurar SVIs con IPs correctas
- [ ] Habilitar ip routing
- [ ] Configurar trunks
- [ ] Configurar puertos de acceso por VLAN
- [ ] Asignar IPs estáticas (cámaras, cajeros, mgmt)
- [ ] Configurar Access Points WiFi
- [ ] Configurar DHCP Relay (o local como backup)
- [ ] Configurar Port Security
- [ ] Configurar DHCP Snooping
- [ ] Configurar ACLs de seguridad
- [ ] Probar conectividad local
- [ ] Probar VPN con Central
- [ ] Probar VoIP con Central
- [ ] Guardar todas las configuraciones

---

## NOTAS IMPORTANTES

1. **VPN**: Configurar IP pública real de Central en crypto map
2. **DHCP**: Usar servidor Central como primario (vía VPN), local como backup
3. **VoIP**: Teléfonos se registran en PBX Central (10.1.50.2)
4. **Cajeros**: Solo pueden comunicarse con servidores en Central (ACL estricta)
5. **WiFi Guest**: Totalmente aislado de redes internas
6. **Testing**: Probar conectividad local antes de VPN

---

**Última actualización:** Noviembre 2025
**Sucursal:** Sucursal 1
