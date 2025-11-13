# Configuración Central - Cisco Packet Tracer

## Topología de Red - Central

```
                    Internet
                        │
                   [Router WAN]
                   IP Pública
                        │
                 [Firewall/Router]
                   10.1.0.254
                        │
              ┌─────────┴─────────┐
              │                   │
        [Core Switch L3]    [Switch Management]
         10.1.x.1 (GW)       VLAN 200
              │
    ┌─────────┼─────────┬─────────┬─────────┐
    │         │         │         │         │
[Switch 1] [Switch 2] [Servers] [NVR]   [PBX]
 PoE+      PoE+      VLAN 100  VLAN 40  VLAN 50
```

---

## 1. INVENTARIO DE EQUIPOS

### Equipos de Red Requeridos en Packet Tracer

| Cantidad | Tipo | Modelo Packet Tracer | Descripción |
|----------|------|---------------------|-------------|
| 1 | Router | 2911 o 4331 | Firewall/Router VPN Principal |
| 1 | Switch L3 | 3560-24PS o Multilayer Switch | Core Switch (Enrutamiento entre VLANs) |
| 3 | Switch L2 | 2960-24TT | Distribution/Access Switches con PoE |
| 1 | Servidor | Server-PT | Servidor para VLAN 100 (servicios) |
| 1 | Servidor | Server-PT | Servidor PBX VoIP |
| 1 | Servidor | Server-PT | NVR/Sistema de cámaras |
| 19 | PC | PC-PT | Computadoras de usuario |
| 7 | IP Phone | IP Phone 7960 | Teléfonos VoIP |
| 43 | Cámara IP | Generic (PC configurado) | Cámaras de seguridad |

---

## 2. TABLA DE VLANs - CENTRAL

| VLAN | Nombre | Subred | Gateway | Dispositivos |
|------|--------|--------|---------|--------------|
| 10 | ADMIN | 10.1.10.0/27 | 10.1.10.1 | 5 PCs + 4 VoIP |
| 20 | TI | 10.1.20.0/27 | 10.1.20.1 | 12 PCs + 1 VoIP |
| 30 | FINANZAS | 10.1.30.0/28 | 10.1.30.1 | 2 PCs + 2 VoIP |
| 40 | CAMARAS | 10.1.40.0/26 | 10.1.40.1 | 43 cámaras + NVR |
| 50 | VOIP | 10.1.50.0/27 | 10.1.50.1 | 7 VoIP + PBX |
| 100 | SERVERS | 10.1.100.0/28 | 10.1.100.1 | Servidores |
| 200 | MGMT | 10.1.200.0/27 | 10.1.200.1 | Gestión switches |
| 999 | NATIVE | - | - | VLAN nativa (trunk) |

---

## 3. CONFIGURACIÓN ROUTER PRINCIPAL (FIREWALL/VPN)

### Router: CENTRAL-RTR-01
**Modelo:** Cisco 2911 o 4331

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname CENTRAL-RTR-01
no ip domain-lookup
service password-encryption
security passwords min-length 10

! Banner de seguridad
banner motd #
*********************************************************
*  ACCESO RESTRINGIDO - OFICINA CENTRAL                *
*  Acceso no autorizado está prohibido                  *
*********************************************************
#

! ==========================================
! INTERFACES
! ==========================================

! Interfaz WAN (hacia Internet - simulado)
interface GigabitEthernet0/0
 description WAN - Internet Connection
 ip address dhcp
 ip nat outside
 no shutdown

! Interfaz LAN (hacia Core Switch)
interface GigabitEthernet0/1
 description LAN - Core Switch Connection
 ip address 10.1.0.254 255.255.0.0
 ip nat inside
 no shutdown

! ==========================================
! NAT - SALIDA A INTERNET
! ==========================================
access-list 1 permit 10.1.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! ==========================================
! RUTAS ESTÁTICAS (para VPN a sucursales)
! ==========================================
! Estas se configurarán cuando se establezcan los túneles VPN
! ip route 192.168.10.0 255.255.255.0 172.16.0.2
! ip route 192.168.20.0 255.255.255.0 172.16.0.6
! ip route 192.168.30.0 255.255.255.0 172.16.0.10

! ==========================================
! VPN IPSEC - SUCURSAL 1
! ==========================================
! Nota: En Packet Tracer, usar VPN GUI o comandos básicos

crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 28800

crypto isakmp key SuperSecretVPN123 address 0.0.0.0

crypto ipsec transform-set VPN-SET esp-aes 256 esp-sha256-hmac
 mode tunnel

! ACL para tráfico interesante (Central <-> Sucursal 1)
access-list 100 permit ip 10.1.0.0 0.0.255.255 192.168.10.0 0.0.0.255

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 0.0.0.0  ! IP pública de Sucursal 1
 set transform-set VPN-SET
 match address 100

interface GigabitEthernet0/0
 crypto map VPN-MAP

! Repetir para Sucursales 2 y 3 con diferentes sequence numbers

! ==========================================
! SEGURIDAD BÁSICA
! ==========================================
! Deshabilitar servicios innecesarios
no ip http server
no ip http secure-server
no cdp run

! Contraseñas
enable secret Cisco123Admin!
username admin privilege 15 secret AdminPass123!

line console 0
 password ConsolePass123!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0

! Habilitar SSH
ip domain-name banco.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

! ==========================================
! NTP
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

! Guardar configuración
end
write memory
```

---

## 4. CONFIGURACIÓN CORE SWITCH L3

### Switch: CENTRAL-CORE-SW01
**Modelo:** Cisco 3560-24PS o Multilayer Switch

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname CENTRAL-CORE-SW01
no ip domain-lookup
service password-encryption

! Banner
banner motd #
*********************************************************
*  CORE SWITCH - OFICINA CENTRAL                       *
*  Acceso solo para personal autorizado TI             *
*********************************************************
#

! ==========================================
! HABILITAR ENRUTAMIENTO IP
! ==========================================
ip routing
ipv6 unicast-routing

! ==========================================
! CREAR VLANs
! ==========================================
vlan 10
 name ADMIN
vlan 20
 name TI
vlan 30
 name FINANZAS
vlan 40
 name CAMARAS
vlan 50
 name VOIP
vlan 100
 name SERVERS
vlan 200
 name MGMT
vlan 999
 name NATIVE

! ==========================================
! CONFIGURAR INTERFACES VLAN (SVIs)
! ==========================================

! VLAN 10 - Administración
interface Vlan10
 description Gateway VLAN Administracion
 ip address 10.1.10.1 255.255.255.224
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 20 - TI
interface Vlan20
 description Gateway VLAN TI
 ip address 10.1.20.1 255.255.255.224
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 30 - Finanzas
interface Vlan30
 description Gateway VLAN Finanzas
 ip address 10.1.30.1 255.255.255.240
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 40 - Cámaras
interface Vlan40
 description Gateway VLAN Camaras
 ip address 10.1.40.1 255.255.255.192
 no ip helper-address
 no shutdown

! VLAN 50 - VoIP
interface Vlan50
 description Gateway VLAN VoIP
 ip address 10.1.50.1 255.255.255.224
 ip helper-address 10.1.100.7
 no shutdown

! VLAN 100 - Servidores
interface Vlan100
 description Gateway VLAN Servidores
 ip address 10.1.100.1 255.255.255.240
 no ip helper-address
 no shutdown

! VLAN 200 - Management
interface Vlan200
 description Gateway VLAN Management
 ip address 10.1.200.1 255.255.255.224
 no ip helper-address
 no shutdown

! ==========================================
! RUTA PREDETERMINADA
! ==========================================
ip route 0.0.0.0 0.0.0.0 10.1.0.254

! ==========================================
! UPLINK AL ROUTER (TRUNK)
! ==========================================
interface GigabitEthernet0/1
 description Uplink to CENTRAL-RTR-01
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,50,100,200
 no shutdown

! ==========================================
! ENLACES A SWITCHES DE DISTRIBUCIÓN (TRUNK)
! ==========================================
interface range GigabitEthernet0/2-4
 description Trunk to Distribution Switches
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,50,100,200
 spanning-tree portfast trunk
 no shutdown

! ==========================================
! CONEXIÓN DIRECTA A SERVIDORES (VLAN 100)
! ==========================================
interface GigabitEthernet0/5
 description Server DHCP/DNS/AD
 switchport mode access
 switchport access vlan 100
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

interface GigabitEthernet0/6
 description Server DB
 switchport mode access
 switchport access vlan 100
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! SPANNING TREE
! ==========================================
spanning-tree mode rapid-pvst
spanning-tree extend system-id
spanning-tree vlan 10,20,30,40,50,100,200 priority 4096

! Root bridge para todas las VLANs
spanning-tree portfast bpduguard default

! ==========================================
! DHCP SNOOPING
! ==========================================
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,50
no ip dhcp snooping information option

! Trust port hacia servidor DHCP
interface GigabitEthernet0/5
 ip dhcp snooping trust

! ==========================================
! DYNAMIC ARP INSPECTION (DAI)
! ==========================================
ip arp inspection vlan 10,20,30,100
ip arp inspection validate src-mac dst-mac ip

interface GigabitEthernet0/5
 ip arp inspection trust

! ==========================================
! QoS BÁSICO
! ==========================================
mls qos
mls qos map cos-dscp 0 8 16 24 32 46 48 56

! Configurar trust en puertos de VoIP
interface range GigabitEthernet0/10-15
 mls qos trust cos
 mls qos trust device cisco-phone

! ==========================================
! SEGURIDAD
! ==========================================
enable secret Cisco123Admin!
username admin privilege 15 secret AdminPass123!

line console 0
 password ConsolePass123!
 login local
 logging synchronous
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name banco.local
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
! SNMP (opcional)
! ==========================================
snmp-server community ReadOnly RO
snmp-server community ReadWrite RW
snmp-server location "Oficina Central - Sala Equipos"
snmp-server contact "TI - ti@banco.local"

end
write memory
```

---

## 5. CONFIGURACIÓN SWITCHES DE DISTRIBUCIÓN/ACCESO

### Switch: CENTRAL-SW-DIST-01 (Switch de Distribución 1)
**Modelo:** Cisco 2960-24TT

```cisco
! ==========================================
! CONFIGURACIÓN BÁSICA
! ==========================================
enable
configure terminal

hostname CENTRAL-SW-DIST-01
no ip domain-lookup
service password-encryption

! ==========================================
! CREAR VLANs
! ==========================================
vlan 10
 name ADMIN
vlan 20
 name TI
vlan 30
 name FINANZAS
vlan 40
 name CAMARAS
vlan 50
 name VOIP
vlan 200
 name MGMT
vlan 999
 name NATIVE

! ==========================================
! CONFIGURACIÓN DE GESTIÓN
! ==========================================
interface Vlan200
 description Management Interface
 ip address 10.1.200.2 255.255.255.224
 no shutdown

ip default-gateway 10.1.200.1

! ==========================================
! UPLINK AL CORE SWITCH (TRUNK)
! ==========================================
interface GigabitEthernet0/1
 description Uplink to CENTRAL-CORE-SW01
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,50,200
 no shutdown

! ==========================================
! PUERTOS DE ACCESO - VLAN 10 (ADMIN)
! ==========================================
interface range FastEthernet0/2-6
 description VLAN 10 - Administracion
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS DE ACCESO - VLAN 20 (TI)
! ==========================================
interface range FastEthernet0/7-17
 description VLAN 20 - TI
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS DE ACCESO - VLAN 30 (FINANZAS)
! ==========================================
interface range FastEthernet0/18-19
 description VLAN 30 - Finanzas
 switchport mode access
 switchport access vlan 30
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ==========================================
! PUERTOS VOIP - VLAN 50 (con voz/datos)
! ==========================================
interface range FastEthernet0/20-24
 description VoIP Phones
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 50
 mls qos trust cos
 spanning-tree portfast
 no shutdown

! ==========================================
! DHCP SNOOPING
! ==========================================
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,50
no ip dhcp snooping information option

! Trust port hacia core
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
enable secret Cisco123Admin!
username admin privilege 15 secret AdminPass123!

line console 0
 password ConsolePass123!
 login local
 exec-timeout 10 0

line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

ip domain-name banco.local
crypto key generate rsa modulus 2048
ip ssh version 2

! NTP
ntp server 10.1.100.8

end
write memory
```

### NOTA: Replicar configuración similar para CENTRAL-SW-DIST-02 y CENTRAL-SW-DIST-03
- Cambiar hostname
- Cambiar IP de management (10.1.200.3, 10.1.200.4)
- Ajustar VLANs según dispositivos conectados

---

## 6. CONFIGURACIÓN DE SERVIDORES

### Servidor DHCP/DNS (10.1.100.7 y 10.1.100.8)

**Configuración en Packet Tracer:**

#### Desktop > IP Configuration:
- IP Address: 10.1.100.7
- Subnet Mask: 255.255.255.240
- Default Gateway: 10.1.100.1
- DNS Server: 10.1.100.8

#### Services > DHCP:

**Pool VLAN 10 (Admin):**
- Pool Name: ADMIN-POOL
- Default Gateway: 10.1.10.1
- DNS Server: 10.1.100.8
- Start IP: 10.1.10.10
- Subnet Mask: 255.255.255.224
- Max Users: 15
- TFTP Server: 10.1.100.8

**Pool VLAN 20 (TI):**
- Pool Name: TI-POOL
- Default Gateway: 10.1.20.1
- DNS Server: 10.1.100.8
- Start IP: 10.1.20.10
- Subnet Mask: 255.255.255.224
- Max Users: 15

**Pool VLAN 30 (Finanzas):**
- Pool Name: FINANZAS-POOL
- Default Gateway: 10.1.30.1
- DNS Server: 10.1.100.8
- Start IP: 10.1.30.5
- Subnet Mask: 255.255.255.240
- Max Users: 8

**Pool VLAN 50 (VoIP):**
- Pool Name: VOIP-POOL
- Default Gateway: 10.1.50.1
- DNS Server: 10.1.100.8
- Start IP: 10.1.50.10
- Subnet Mask: 255.255.255.224
- Max Users: 15
- TFTP Server: 10.1.50.2

#### Services > DNS:
- Activar DNS Service
- Agregar registros:
  - central.banco.local → 10.1.100.1
  - servidor.banco.local → 10.1.100.7
  - pbx.banco.local → 10.1.50.2

### Servidor PBX VoIP (10.1.50.2)

**Configuración en Packet Tracer:**

#### Desktop > IP Configuration:
- IP Address: 10.1.50.2 (Estática)
- Subnet Mask: 255.255.255.224
- Default Gateway: 10.1.50.1
- DNS Server: 10.1.100.8

#### Services > Telephony Service:
- Activar servicio
- Registrar teléfonos VoIP (extensiones 1001-1007)

---

## 7. CONFIGURACIÓN DE PCs Y TELÉFONOS

### PCs de Usuario (Ejemplo: PC-Admin-01)

**Desktop > IP Configuration:**
- Seleccionar DHCP
- Verificar que obtenga IP del rango correcto según VLAN

**Verificación:**
```
ipconfig
ping 10.1.100.1
ping 8.8.8.8
```

### Teléfonos IP (Ejemplo: Phone-Admin-01)

**Config > GUI:**
- DHCP: Enabled
- TFTP Server: 10.1.100.8 (automático vía DHCP)
- Extension: 1001

**Physical:**
- Conectar cable de teléfono al switch
- Conectar PC al puerto de teléfono (pass-through)

---

## 8. ASIGNACIÓN DE IPs ESTÁTICAS

### VLAN 40 - Cámaras (Todas estáticas)
```
Cámara 1: 10.1.40.2/26
Cámara 2: 10.1.40.3/26
...
Cámara 43: 10.1.40.44/26
NVR: 10.1.40.50/26
```

### VLAN 100 - Servidores (Todas estáticas)
```
10.1.100.1 - Gateway
10.1.100.2 - Servidor DB Principal
10.1.100.3 - Servidor DB Secundario
10.1.100.4 - Servidor Aplicaciones
10.1.100.5 - Servidor Archivos
10.1.100.6 - Servidor AD/RADIUS
10.1.100.7 - Servidor DHCP
10.1.100.8 - Servidor DNS/NTP
10.1.100.9 - Servidor VPN (interfaz LAN)
10.1.100.10 - Servidor Respaldos/Logs
```

### VLAN 200 - Management (Todas estáticas)
```
10.1.200.1 - Gateway
10.1.200.2 - CENTRAL-SW-DIST-01
10.1.200.3 - CENTRAL-SW-DIST-02
10.1.200.4 - CENTRAL-SW-DIST-03
10.1.200.11 - CENTRAL-RTR-01 (gestión)
```

---

## 9. VERIFICACIÓN Y TESTING

### En Router:
```
show ip interface brief
show ip route
show crypto isakmp sa
show crypto ipsec sa
show ip nat translations
```

### En Core Switch:
```
show vlan brief
show ip interface brief
show ip route
show spanning-tree summary
show ip dhcp snooping
show ip arp inspection
```

### En Switches de Distribución:
```
show vlan brief
show interfaces status
show port-security
show spanning-tree
```

### Pruebas de Conectividad:
1. Ping entre VLANs (verificar ACLs)
2. Ping a Internet (8.8.8.8)
3. Pruebas VoIP (llamadas entre extensiones)
4. Acceso a servidores desde PCs
5. Resolución DNS

---

## 10. DIAGRAMA DE CABLEADO

```
CENTRAL-RTR-01 Gi0/1 ──────── Gi0/1 CENTRAL-CORE-SW01
                               │
                ┌──────────────┼──────────────┐
                │              │              │
               Gi0/2          Gi0/3          Gi0/4
                │              │              │
        SW-DIST-01      SW-DIST-02      SW-DIST-03
          Fa0/2-24        Fa0/2-24        Fa0/2-24
            │               │               │
         PCs/VoIP        PCs/VoIP        Cámaras
        VLAN 10,20,30  VLAN 10,20,50    VLAN 40

CENTRAL-CORE-SW01 Gi0/5 ──── Servidor DHCP/DNS (VLAN 100)
CENTRAL-CORE-SW01 Gi0/6 ──── Servidor DB (VLAN 100)
CENTRAL-CORE-SW01 Gi0/7 ──── Servidor PBX (VLAN 50)
```

---

## 11. CHECKLIST DE IMPLEMENTACIÓN

- [ ] Configurar Router principal con NAT
- [ ] Configurar VLANs en Core Switch
- [ ] Configurar SVIs (interfaces VLAN) con IPs correctas
- [ ] Habilitar ip routing en Core Switch
- [ ] Configurar trunks entre switches
- [ ] Configurar puertos de acceso por VLAN
- [ ] Configurar DHCP Relay (ip helper-address)
- [ ] Configurar servidor DHCP con pools
- [ ] Configurar servidor DNS
- [ ] Configurar servidor PBX VoIP
- [ ] Conectar y configurar teléfonos IP
- [ ] Asignar IPs estáticas a servidores y cámaras
- [ ] Configurar Port Security
- [ ] Configurar DHCP Snooping
- [ ] Configurar Spanning Tree
- [ ] Configurar QoS para VoIP
- [ ] Probar conectividad entre VLANs
- [ ] Probar llamadas VoIP
- [ ] Configurar acceso SSH
- [ ] Configurar NTP
- [ ] Guardar todas las configuraciones

---

## NOTAS IMPORTANTES

1. **Packet Tracer Limitations**: Algunas funciones avanzadas pueden no estar disponibles
2. **VPN**: Configurar después de tener las sucursales listas
3. **Testing**: Probar cada VLAN individualmente antes de integrar
4. **Backup**: Guardar el archivo .pkt frecuentemente
5. **Documentar**: Mantener registro de IPs asignadas

---

**Última actualización:** Noviembre 2025
**Autor:** Equipo TI - Proyecto Redes
