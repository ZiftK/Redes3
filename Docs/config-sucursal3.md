# Configuración Sucursal 3 - Cisco Packet Tracer

## Topología de Red - Sucursal 3

```
                    Internet
                        │
                   [Router WAN]
                   IP Pública
                        │
                 [Router Firewall/VPN]
              GW: 192.168.30.x (por VLAN)
                        │
                 [Core Switch L3]
              SVIs: 192.168.30.x/yy
                        │
         ┌──────────────┼──────────────┐
         │              │              │
    [Switch 1]     [Switch 2]      [NVR Local]
     PoE+           PoE+          VLAN 35
      │              │
   PCs/VoIP      Cámaras IP
   
   7x Access Points (WiFi) ── VLAN 37 (Guest)
```

---

## 1. INVENTARIO DE EQUIPOS - SUCURSAL 3

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

## 2. TABLA DE VLANs - SUCURSAL 3

| VLAN | Nombre | Subred | Gateway | Hosts | Dispositivos |
|------|--------|--------|---------|-------|--------------|
| 31 | GERENCIA_S3 | 192.168.30.0/27 | 192.168.30.1 | 30 | 12 PCs + 12 VoIP |
| 32 | CAJAS_S3 | 192.168.30.32/29 | 192.168.30.33 | 6 | 3 PCs + 3 VoIP |
| 33 | RECEPCION_S3 | 192.168.30.40/29 | 192.168.30.41 | 6 | 2 PCs + 2 VoIP |
| 34 | SEGURIDAD_S3 | 192.168.30.48/29 | 192.168.30.49 | 6 | 1 PC + 1 VoIP |
| 35 | CAMARAS_S3 | 192.168.30.64/26 | 192.168.30.65 | 62 | 29 cámaras + NVR |
| 36 | CAJEROS_S3 | 192.168.30.128/29 | 192.168.30.129 | 6 | 2 cajeros ATM |
| 37 | CLIENTES_S3 | 192.168.30.192/26 | 192.168.30.193 | 62 | WiFi Guest (7 APs) |
| 38 | MGMT_S3 | 192.168.30.136/29 | 192.168.30.137 | 6 | Gestión switches |
| 999 | NATIVE | - | - | - | VLAN nativa (trunk) |

---

## 3. CONFIGURACIÓN ROUTER VPN - SUCURSAL 3

### Router: SUCURSAL3-RTR-01
**Modelo:** Cisco 2911 o ISR 1100

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL3-RTR-01
no ip domain-lookup
service password-encryption
security passwords min-length 10

banner motd #
*********************************************************
*  ACCESO RESTRINGIDO - SUCURSAL 3                     *
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
 ip address 192.168.30.254 255.255.255.0
 ip nat inside
 no shutdown

! ==========================================
! NAT - SALIDA A INTERNET
! ==========================================
access-list 1 permit 192.168.30.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! ==========================================
! RUTAS ESTÁTICAS
! ==========================================
! Ruta a Central vía túnel VPN
ip route 10.1.0.0 255.255.0.0 172.16.0.9

! Rutas a otras sucursales (vía Central)
ip route 192.168.10.0 255.255.255.0 172.16.0.9
ip route 192.168.20.0 255.255.255.0 172.16.0.9

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

! ACL para tráfico interesante (Sucursal 3 <-> Central)
access-list 100 permit ip 192.168.30.0 0.0.0.255 10.1.0.0 0.0.255.255

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
 permit tcp 192.168.30.128 0.0.0.7 10.1.100.0 0.0.0.15 eq 443
 permit icmp 192.168.30.128 0.0.0.7 10.1.100.0 0.0.0.15 echo
 deny   ip 192.168.30.128 0.0.0.7 any log
 permit ip any any

! Proteger VLAN Management - Solo desde TI Central
ip access-list extended ACL-MGMT
 permit tcp 10.1.20.0 0.0.0.31 192.168.30.136 0.0.0.7 eq 22
 permit tcp 10.1.20.0 0.0.0.31 192.168.30.136 0.0.0.7 eq 443
 deny   ip any 192.168.30.136 0.0.0.7 log
 permit ip any any

! WiFi Guest - Solo Internet, sin acceso interno
ip access-list extended ACL-WIFI-GUEST
 deny   ip 192.168.30.192 0.0.0.63 192.168.30.0 0.0.0.255 log
 deny   ip 192.168.30.192 0.0.0.63 10.1.0.0 0.0.255.255 log
 permit ip 192.168.30.192 0.0.0.63 any

! ==========================================
! SEGURIDAD BÁSICA
! ==========================================
no ip http server
no ip http secure-server
no cdp run

enable secret Cisco123Suc3!
username admin privilege 15 secret AdminSuc3Pass!

line console 0
 password ConsoleSuc3!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0

! SSH
ip domain-name sucursal3.banco.local
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
ip dhcp excluded-address 192.168.30.1 192.168.30.9
ip dhcp excluded-address 192.168.30.33 192.168.30.33
ip dhcp excluded-address 192.168.30.41 192.168.30.41
ip dhcp excluded-address 192.168.30.49 192.168.30.49

ip dhcp pool GERENCIA-LOCAL
 network 192.168.30.0 255.255.255.224
 default-router 192.168.30.1
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool CAJAS-LOCAL
 network 192.168.30.32 255.255.255.248
 default-router 192.168.30.33
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool RECEPCION-LOCAL
 network 192.168.30.40 255.255.255.248
 default-router 192.168.30.41
 dns-server 8.8.8.8 1.1.1.1
 lease 0 4

ip dhcp pool WIFI-GUEST
 network 192.168.30.192 255.255.255.192
 default-router 192.168.30.193
 dns-server 8.8.8.8 1.1.1.1
 lease 0 2

end
write memory
```

---

## 4. CONFIGURACIÓN CORE SWITCH L3 - SUCURSAL 3

### Switch: SUCURSAL3-CORE-SW01
**Modelo:** Cisco 3560-24PS

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname SUCURSAL3-CORE-SW01
no ip domain-lookup
service password-encryption

banner motd #
*********************************************************
*  CORE SWITCH - SUCURSAL 3                            *
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
vlan 31
 name GERENCIA_S3
vlan 32
 name CAJAS_S3
vlan 33
 name RECEPCION_S3
vlan 34
 name SEGURIDAD_S3
vlan 35
 name CAMARAS_S3
vlan 36
 name CAJEROS_S3
vlan 37
 name CLIENTES_S3
vlan 38
 name MGMT_S3
vlan 999
 name NATIVE

! ==========================================
! CONFIGURAR INTERFACES VLAN (SVIs)
! ==========================================

! VLAN 31 - Gerencia
interface Vlan31
 description Gateway VLAN Gerencia
 ip address 192.168.30.1 255.255.255.224
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 32 - Cajas
interface Vlan32
 description Gateway VLAN Cajas
 ip address 192.168.30.33 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 33 - Recepción
interface Vlan33
 description Gateway VLAN Recepcion
 ip address 192.168.30.41 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 34 - Seguridad
interface Vlan34
 description Gateway VLAN Seguridad
 ip address 192.168.30.49 255.255.255.248
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 35 - Cámaras
interface Vlan35
 description Gateway VLAN Camaras
 ip address 192.168.30.65 255.255.255.192
 no ip helper-address
 no shutdown

! VLAN 36 - Cajeros ATM
interface Vlan36
 description Gateway VLAN Cajeros
 ip address 192.168.30.129 255.255.255.248
 no ip helper-address
 no shutdown

! VLAN 37 - WiFi Guest
interface Vlan37
 description Gateway VLAN WiFi Guest
 ip address 192.168.30.193 255.255.255.192
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 38 - Management
interface Vlan38
 description Gateway VLAN Management
 ip address 192.168.30.137 255.255.255.248
 no ip helper-address
 no shutdown

! ==========================================
! RUTA PREDETERMINADA
! ==========================================
ip route 0.0.0.0 0.0.0.0 192.168.30.254

! ==========================================
! UPLINK AL ROUTER (TRUNK)
! ==========================================
interface GigabitEthernet0/1
 description Uplink to SUCURSAL3-RTR-01
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 31-38,999
 no shutdown

! ==========================================
! ENLACES A SWITCHES DE ACCESO (TRUNK)
! ==========================================
interface range GigabitEthernet0/2-3
 description Trunk to Access Switches
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 31-38,999
 spanning-tree portfast trunk
 no shutdown

! ==========================================
! PUERTO NVR (VLAN 35 - Cámaras)
! ==========================================
interface GigabitEthernet0/5
 description NVR Local
 switchport mode access
 switchport access vlan 35
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS CAJEROS ATM (VLAN 36)
! ==========================================
interface range GigabitEthernet0/6-7
 description Cajeros ATM
 switchport mode access
 switchport access vlan 36
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
spanning-tree vlan 31-38 priority 4096
spanning-tree portfast bpduguard default

! ==========================================
! DHCP SNOOPING
! ==========================================
ip dhcp snooping
ip dhcp snooping vlan 31,32,33,34,37
no ip dhcp snooping information option

interface GigabitEthernet0/1
 ip dhcp snooping trust

! ==========================================
! DYNAMIC ARP INSPECTION
! ==========================================
ip arp inspection vlan 31,32,36
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
enable secret Cisco123Suc3!
username admin privilege 15 secret AdminSuc3Pass!

line console 0
 password ConsoleSuc3!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name sucursal3.banco.local
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
snmp-server location "Sucursal 3 - Sala Equipos"
snmp-server contact "TI - ti@banco.local"

end
write memory
```

---

## 5. CONFIGURACIÓN SWITCHES DE ACCESO

### Switch: SUCURSAL3-SW-ACC-01
**Modelo:** Cisco 2960-24TT

```cisco
enable
configure terminal

hostname SUCURSAL3-SW-ACC-01
no ip domain-lookup
service password-encryption

! Crear VLANs
vlan 31
 name GERENCIA_S3
vlan 32
 name CAJAS_S3
vlan 33
 name RECEPCION_S3
vlan 34
 name SEGURIDAD_S3
vlan 37
 name CLIENTES_S3
vlan 38
 name MGMT_S3
vlan 999
 name NATIVE

! Configuración de gestión
interface Vlan38
 description Management Interface
 ip address 192.168.30.138 255.255.255.248
 no shutdown

ip default-gateway 192.168.30.137

! Uplink al Core Switch (TRUNK)
interface GigabitEthernet0/1
 description Uplink to SUCURSAL3-CORE-SW01
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 31-34,37,38,999
 no shutdown

! Puertos Gerencia (VLAN 31)
interface range FastEthernet0/2-13
 description VLAN 31 - Gerencia PCs
 switchport mode access
 switchport access vlan 31
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
 switchport access vlan 31
 switchport voice vlan 31
 mls qos trust cos
 spanning-tree portfast
 no shutdown

! DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 31,32,33,34,37
no ip dhcp snooping information option

interface GigabitEthernet0/1
 ip dhcp snooping trust

! Spanning Tree
spanning-tree mode rapid-pvst
spanning-tree portfast bpduguard default

! Seguridad
enable secret Cisco123Suc3!
username admin privilege 15 secret AdminSuc3Pass!

line console 0
 password ConsoleSuc3!
 login local
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name sucursal3.banco.local
crypto key generate rsa modulus 2048
ip ssh version 2

ntp server 10.1.100.8

end
write memory
```

### Switch: SUCURSAL3-SW-ACC-02

```cisco
hostname SUCURSAL3-SW-ACC-02

interface Vlan38
 ip address 192.168.30.139 255.255.255.248

! Puertos para Cajas (VLAN 32)
interface range FastEthernet0/2-4
 description VLAN 32 - Cajas
 switchport mode access
 switchport access vlan 32

! Puertos para Recepción (VLAN 33)
interface range FastEthernet0/5-6
 description VLAN 33 - Recepcion
 switchport mode access
 switchport access vlan 33

! Puerto para Seguridad (VLAN 34)
interface FastEthernet0/7
 description VLAN 34 - Seguridad
 switchport mode access
 switchport access vlan 34

! Puertos para Cámaras (VLAN 35)
interface range FastEthernet0/8-24
 description VLAN 35 - Camaras IP
 switchport mode access
 switchport access vlan 35
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict

! (Resto similar a SW-ACC-01)
```

---

## 6. CONFIGURACIÓN ACCESS POINTS (WiFi Guest)

### Access Point: AP-S3-01 hasta AP-S3-07

**Config > Port 1:**
- **SSID:** Banco_Guest_S3
- **Authentication:** WPA2-PSK
- **PSK Pass Phrase:** GuestWiFi2025!
- **Encryption Type:** AES

**GUI:**
- VLAN: 37
- IP Addresses: 192.168.30.200-206
- Subnet Mask: 255.255.255.192
- Gateway: 192.168.30.193

---

## 7. ASIGNACIÓN DE IPs ESTÁTICAS

### VLAN 35 - Cámaras
```
192.168.30.65 - Gateway
192.168.30.66-94 - Cámaras 1-29
192.168.30.95 - NVR Local
```

### VLAN 36 - Cajeros ATM
```
192.168.30.129 - Gateway
192.168.30.130 - Cajero ATM 1
192.168.30.131 - Cajero ATM 2
```

### VLAN 38 - Management
```
192.168.30.137 - Gateway
192.168.30.138 - SUCURSAL3-SW-ACC-01
192.168.30.139 - SUCURSAL3-SW-ACC-02
192.168.30.140 - SUCURSAL3-CORE-SW01
192.168.30.141 - SUCURSAL3-RTR-01
```

### VLAN 37 - Access Points
```
192.168.30.193 - Gateway
192.168.30.200-206 - APs (7 unidades)
192.168.30.194-250 - Pool DHCP clientes WiFi
```

---

## 8. CONFIGURACIÓN DE PCs Y TELÉFONOS

### PCs de Usuario
- **DHCP habilitado** en todos los PCs
- Verificar obtención de IP del rango correcto

### Teléfonos VoIP
- **Call Manager:** 10.1.50.2 (PBX Central vía VPN)
- **Extensions:** 3001-3016 (Sucursal 3)

### Cajeros ATM
- **IPs Estáticas:** 192.168.30.130, 192.168.30.131
- **Comunicación:** Solo con 10.1.100.0/28 (servidores Central)

---

## 9. VERIFICACIÓN Y TESTING

### Comandos de Verificación:

**En Router:**
```
show ip interface brief
show ip route
show crypto isakmp sa
show crypto ipsec sa
show ip nat translations
ping 10.1.100.1
ping 8.8.8.8
```

**En Core Switch:**
```
show vlan brief
show ip interface brief
show ip route
show spanning-tree summary
show ip dhcp snooping binding
show ip arp inspection
```

**En Switches de Acceso:**
```
show vlan brief
show interfaces status
show port-security
show cdp neighbors
```

### Pruebas de Conectividad:
1. ✅ Ping entre VLANs locales
2. ✅ Ping a Central (10.1.100.1) vía VPN
3. ✅ Ping a Internet (8.8.8.8)
4. ✅ Ping a otras sucursales (vía Central)
5. ✅ Llamadas VoIP a Central y otras sucursales
6. ✅ Acceso WiFi Guest sin acceso a redes internas
7. ✅ Cajeros solo acceden a servidores Central
8. ✅ Resolución DNS

---

## 10. DIAGRAMA DE CABLEADO

```
SUCURSAL3-RTR-01 Gi0/1 ──── Gi0/1 SUCURSAL3-CORE-SW01
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
         (VLAN 31)    (VLAN 31)         (VLAN 32,33)  (VLAN 35)

SUCURSAL3-CORE-SW01:
  - Gi0/5: NVR (VLAN 35)
  - Gi0/6-7: Cajeros ATM (VLAN 36)

Access Points (7x) conectados a VLAN 37
```

---

## 11. CHECKLIST DE IMPLEMENTACIÓN

- [ ] Router configurado con interfaces
- [ ] NAT configurado
- [ ] VPN IPSec hacia Central configurada
- [ ] Rutas estáticas configuradas
- [ ] ACLs de seguridad aplicadas
- [ ] VLANs creadas en todos los switches
- [ ] SVIs configuradas con IPs correctas
- [ ] IP routing habilitado en Core Switch
- [ ] Trunks configurados entre switches
- [ ] Puertos de acceso asignados a VLANs
- [ ] Port Security configurado
- [ ] DHCP Snooping habilitado
- [ ] Dynamic ARP Inspection habilitado
- [ ] QoS configurado para VoIP
- [ ] Access Points configurados
- [ ] IPs estáticas asignadas (cámaras, cajeros, mgmt, APs)
- [ ] NVR configurado en VLAN 35
- [ ] Cajeros configurados en VLAN 36
- [ ] SSH habilitado y probado
- [ ] NTP configurado apuntando a Central
- [ ] Logging configurado
- [ ] Pruebas de conectividad local exitosas
- [ ] Túnel VPN con Central establecido
- [ ] Pruebas VoIP con Central exitosas
- [ ] WiFi Guest operativo y aislado
- [ ] Todas las configuraciones guardadas (write memory)

---

## 12. RESOLUCIÓN DE PROBLEMAS COMUNES

### VPN no establece túnel
1. Verificar IP pública de Central en crypto map
2. Verificar pre-shared key coincide
3. Verificar ACL de tráfico interesante (access-list 100)
4. `show crypto isakmp sa` - Ver fase 1
5. `show crypto ipsec sa` - Ver fase 2
6. `debug crypto isakmp` (cuidado en producción)

### VoIP no funciona
1. Verificar DHCP entrega opción 150 (PBX server)
2. Verificar conectividad a 10.1.50.2 vía VPN
3. Verificar QoS habilitado
4. En teléfono: Config > verificar Call Manager

### WiFi Guest puede acceder a red interna
1. Verificar ACL-WIFI-GUEST aplicada
2. Verificar Client Isolation en APs
3. Probar ping desde cliente WiFi a 192.168.30.1 (debe fallar)

### Cámaras no graban en NVR
1. Verificar IPs estáticas correctas
2. Verificar puerto NVR en VLAN 35
3. Ping desde cámara a NVR (192.168.30.95)

### No hay Internet
1. Verificar NAT: `show ip nat translations`
2. Verificar ruta default: `show ip route`
3. Verificar interfaz WAN tiene IP: `show ip interface brief`
4. Ping a 8.8.8.8

---

## NOTAS FINALES

1. **Seguridad:** Cambiar todas las contraseñas por defecto antes de producción
2. **Backup:** Guardar configuraciones regularmente (`write memory`)
3. **Documentación:** Mantener diagrama actualizado con IPs asignadas
4. **Monitoreo:** Configurar SNMP y syslog para monitoreo centralizado
5. **Testing:** Probar failover de VPN si se implementa redundancia

---

**Última actualización:** Noviembre 2025
**Sucursal:** Sucursal 3
**Estado:** Configuración lista para implementación en Packet Tracer
