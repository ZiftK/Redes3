# Equipos
## Sucursal (x3)
- 29 camaras
- 18 PCs
    - 12 gerenciales
    - 3 cajas
    - 2 recepcion
    - 1 seguridad
- 16 VoIP
    - 12 gerenciales
    - 3 cajas
    - 2 recepcion
    - 1 seguridad
- 2 Cajeros
- 7 Access Points

## Central

- 43 camaras
- 19 PCs
    - 2 secretarias
    - 1 trabajo social
    - 1 contador
    - 1 director
    - 1 subdirector
    - 11 TI
    - 1 coordinador TI
    - 1 administracion de equipos.
- 7 VoIP
    - 2 secretarias
    - 1 trabajo social
    - 1 contador
    - 1 subdirector
    - 1 coordinador TI

# Separacion por VLANs
## Sucursal (x3)
- Camaras
- Gerencia
    - Computadoras y VoIP
- Cajas
    - Computadoras y VoIP
- Recepcion
    - Computadoras y VoIP
- Seguridad
    - Computadoras y VoIP
- Cajeros
- Clientes (APs)

# Segmentación de Red

## Topología General

```
Internet
    │
    ├─── [Firewall + Router VPN] ──── CENTRAL (Red Local Privada)
    │
    ├─── [ISP] ──VPN IPSec── [Router VPN] ──── SUCURSAL 1 (Red Local Privada)
    │
    ├─── [ISP] ──VPN IPSec── [Router VPN] ──── SUCURSAL 2 (Red Local Privada)
    │
    └─── [ISP] ──VPN IPSec── [Router VPN] ──── SUCURSAL 3 (Red Local Privada)
```

**NOTA IMPORTANTE**: Cada ubicación (Central y 3 sucursales) tiene su propia red LAN privada completamente independiente. La comunicación entre ubicaciones se realiza exclusivamente a través de túneles VPN IPSec sobre Internet público.

---

## Esquema de Direccionamiento

### Redes Privadas por Ubicación
- **Oficina Central**: 10.1.0.0/16 (Red LAN privada)
- **Sucursal 1**: 192.168.10.0/24 (Red LAN privada, completamente separada)
- **Sucursal 2**: 192.168.20.0/24 (Red LAN privada, completamente separada)
- **Sucursal 3**: 192.168.30.0/24 (Red LAN privada, completamente separada)
- **Túneles VPN**: 172.16.0.0/24 (Direcciones punto a punto para túneles)

### Conectividad WAN
- **Central**: IP Pública estática (ISP)
- **Sucursal 1**: IP Pública estática o dinámica + DynDNS (ISP)
- **Sucursal 2**: IP Pública estática o dinámica + DynDNS (ISP)
- **Sucursal 3**: IP Pública estática o dinámica + DynDNS (ISP)

---

## Central - VLANs y Subredes (Red LAN: 10.1.0.0/16)

| VLAN ID | Nombre | Subred | Gateway | Hosts Útiles | Dispositivos Reales | Descripción |
|---------|--------|--------|---------|--------------|---------------------|-------------|
| 10 | VLAN_ADMINISTRACION | 10.1.10.0/27 | 10.1.10.1 | 30 | 5 PCs + 4 VoIP = 9 | Director, Subdirector, Secretarias, Admin Equipos |
| 20 | VLAN_TI | 10.1.20.0/27 | 10.1.20.1 | 30 | 12 PCs + 1 VoIP = 13 | Equipo TI y Coordinador TI |
| 30 | VLAN_FINANZAS | 10.1.30.0/28 | 10.1.30.1 | 14 | 2 PCs + 2 VoIP = 4 | Contador, Trabajo Social |
| 40 | VLAN_CAMARAS_CENTRAL | 10.1.40.0/26 | 10.1.40.1 | 62 | 43 cámaras + 1 NVR = 44 | Cámaras de seguridad |
| 50 | VLAN_VOIP_CENTRAL | 10.1.50.0/27 | 10.1.50.1 | 30 | 7 VoIP + 1 PBX = 8 | VoIP administrativo + PBX |
| 100 | VLAN_SERVIDORES | 10.1.100.0/28 | 10.1.100.1 | 14 | ~10 servidores | Servidores core, BD, aplicaciones |
| 200 | VLAN_MANAGEMENT | 10.1.200.0/27 | 10.1.200.1 | 30 | ~15 equipos de red | Gestión de switches, routers, firewalls |

### Distribución de IPs - Central

#### VLAN 10 - Administración (10.1.10.0/27) - Rango: .1 a .30
- 10.1.10.1: Gateway (Router/Switch L3)
- 10.1.10.2-3: Secretarias (2 PCs)
- 10.1.10.4: Director (1 PC)
- 10.1.10.5: Subdirector (1 PC)
- 10.1.10.6: Administración de equipos (1 PC)
- 10.1.10.10-13: VoIP (2 secretarias, 1 subdirector, 1 reserva)
- 10.1.10.20-30: Reservado para crecimiento limitado
- **Máscara /27 limita a 30 hosts** - Protege contra conexiones no autorizadas

#### VLAN 20 - TI (10.1.20.0/27) - Rango: .1 a .30
- 10.1.20.1: Gateway
- 10.1.20.2-12: Equipo TI (11 PCs)
- 10.1.20.13: Coordinador TI (1 PC)
- 10.1.20.14: VoIP Coordinador TI
- 10.1.20.20-30: Equipos de prueba y desarrollo
- **Máscara /27 limita a 30 hosts**

#### VLAN 30 - Finanzas (10.1.30.0/28) - Rango: .1 a .14
- 10.1.30.1: Gateway
- 10.1.30.2: Contador (PC)
- 10.1.30.3: Trabajo Social (PC)
- 10.1.30.10: VoIP Contador
- 10.1.30.11: VoIP Trabajo Social
- 10.1.30.12-14: Reservado
- **Máscara /28 limita a 14 hosts** - Red crítica con acceso muy restringido

#### VLAN 40 - Cámaras Central (10.1.40.0/26) - Rango: .1 a .62
- 10.1.40.1: Gateway
- 10.1.40.2-44: Cámaras de seguridad (43 dispositivos)
- 10.1.40.50: NVR/DVR Principal
- 10.1.40.51: NVR Secundario/Respaldo
- 10.1.40.60-62: Reservado para expansión mínima
- **Máscara /26 limita a 62 hosts** - Solo cámaras, sin acceso a otros dispositivos

#### VLAN 50 - VoIP Central (10.1.50.0/27) - Rango: .1 a .30
- 10.1.50.1: Gateway
- 10.1.50.2: Central telefónica/PBX
- 10.1.50.3: Servidor SIP de respaldo
- 10.1.50.10-16: Teléfonos VoIP (7 dispositivos actuales)
- 10.1.50.17-30: Reservado para expansión telefónica
- **Máscara /27 limita a 30 hosts**

#### VLAN 100 - Servidores (10.1.100.0/28) - Rango: .1 a .14
- 10.1.100.1: Gateway
- 10.1.100.2: Servidor de Base de Datos Principal
- 10.1.100.3: Servidor de Base de Datos Secundario
- 10.1.100.4: Servidor de Aplicaciones
- 10.1.100.5: Servidor de Archivos
- 10.1.100.6: Servidor Active Directory/LDAP
- 10.1.100.7: Servidor DHCP
- 10.1.100.8: Servidor DNS
- 10.1.100.9: Servidor de VPN (interfaz LAN)
- 10.1.100.10: Servidor de respaldos
- 10.1.100.11-14: Reservado para servidores críticos adicionales
- **Máscara /28 limita a 14 hosts** - Red ultra crítica

#### VLAN 200 - Management (10.1.200.0/27) - Rango: .1 a .30
- 10.1.200.1: Gateway
- 10.1.200.2-10: Switches gestionables (interfaces de gestión)
- 10.1.200.11-15: Routers (interfaces de gestión)
- 10.1.200.16-20: Firewalls (interfaces de gestión)
- 10.1.200.21-30: APs y otros equipos de red
- **Máscara /27 limita a 30 hosts** - Solo personal TI autorizado

---

## Sucursales - VLANs y Subredes (Redes LAN Privadas Independientes)

**IMPORTANTE**: Cada sucursal tiene su propia red LAN completamente aislada. No hay conexión directa entre ellas, solo a través de VPN hacia Central.

### Sucursal 1 (Red LAN: 192.168.10.0/24)

| VLAN ID | Nombre | Subred | Gateway | Hosts Útiles | Dispositivos Reales | Descripción |
|---------|--------|--------|---------|--------------|---------------------|-------------|
| 11 | VLAN_GERENCIA_S1 | 192.168.10.0/27 | 192.168.10.1 | 30 | 12 PCs + 12 VoIP = 24 | Gerencia y administración |
| 12 | VLAN_CAJAS_S1 | 192.168.10.32/29 | 192.168.10.33 | 6 | 3 PCs + 3 VoIP = 6 | Cajas y atención |
| 13 | VLAN_RECEPCION_S1 | 192.168.10.40/29 | 192.168.10.41 | 6 | 2 PCs + 2 VoIP = 4 | Recepción |
| 14 | VLAN_SEGURIDAD_S1 | 192.168.10.48/29 | 192.168.10.49 | 6 | 1 PC + 1 VoIP = 2 | Seguridad física |
| 15 | VLAN_CAMARAS_S1 | 192.168.10.64/26 | 192.168.10.65 | 62 | 29 cámaras + 1 NVR = 30 | Cámaras de vigilancia |
| 16 | VLAN_CAJEROS_S1 | 192.168.10.128/29 | 192.168.10.129 | 6 | 2 cajeros ATM | Cajeros automáticos |
| 17 | VLAN_CLIENTES_S1 | 192.168.10.192/26 | 192.168.10.193 | 62 | WiFi público | WiFi clientes (Guest) |
| 18 | VLAN_MGMT_S1 | 192.168.10.136/29 | 192.168.10.137 | 6 | Equipos de red | Gestión local |

#### Distribución de IPs - Sucursal 1

**VLAN 11 - Gerencia (192.168.10.0/27)** - Cable
- .1: Gateway, .2-13: PCs Gerencia (12), .16-27: VoIP Gerencia (12)

**VLAN 12 - Cajas (192.168.10.32/29)** - Cable
- .33: Gateway, .34-36: PCs Cajas (3), .37-39: VoIP Cajas (3)

**VLAN 13 - Recepción (192.168.10.40/29)** - Cable
- .41: Gateway, .42-43: PCs Recepción (2), .44-45: VoIP Recepción (2)

**VLAN 14 - Seguridad (192.168.10.48/29)** - Cable
- .49: Gateway, .50: PC Seguridad (1), .51: VoIP Seguridad (1)

**VLAN 15 - Cámaras (192.168.10.64/26)** - Cable
- .65: Gateway, .66-94: Cámaras (29), .95: NVR

**VLAN 16 - Cajeros (192.168.10.128/29)** - Cable
- .129: Gateway, .130-131: Cajeros ATM (2)

**VLAN 17 - Clientes WiFi (192.168.10.192/26)** - Inalámbrico
- .193: Gateway, .194-254: Pool DHCP para clientes (7 APs)

**VLAN 18 - Management (192.168.10.136/29)** - Cable
- .137: Gateway, .138-143: Switches, Router, APs

---

### Sucursal 2 (Red LAN: 192.168.20.0/24)

| VLAN ID | Nombre | Subred | Gateway | Hosts Útiles | Dispositivos Reales | Descripción |
|---------|--------|--------|---------|--------------|---------------------|-------------|
| 21 | VLAN_GERENCIA_S2 | 192.168.20.0/27 | 192.168.20.1 | 30 | 12 PCs + 12 VoIP = 24 | Gerencia y administración |
| 22 | VLAN_CAJAS_S2 | 192.168.20.32/29 | 192.168.20.33 | 6 | 3 PCs + 3 VoIP = 6 | Cajas y atención |
| 23 | VLAN_RECEPCION_S2 | 192.168.20.40/29 | 192.168.20.41 | 6 | 2 PCs + 2 VoIP = 4 | Recepción |
| 24 | VLAN_SEGURIDAD_S2 | 192.168.20.48/29 | 192.168.20.49 | 6 | 1 PC + 1 VoIP = 2 | Seguridad física |
| 25 | VLAN_CAMARAS_S2 | 192.168.20.64/26 | 192.168.20.65 | 62 | 29 cámaras + 1 NVR = 30 | Cámaras de vigilancia |
| 26 | VLAN_CAJEROS_S2 | 192.168.20.128/29 | 192.168.20.129 | 6 | 2 cajeros ATM | Cajeros automáticos |
| 27 | VLAN_CLIENTES_S2 | 192.168.20.192/26 | 192.168.20.193 | 62 | WiFi público | WiFi clientes (Guest) |
| 28 | VLAN_MGMT_S2 | 192.168.20.136/29 | 192.168.20.137 | 6 | Equipos de red | Gestión local |

#### Distribución de IPs - Sucursal 2
(Misma estructura que Sucursal 1, con rango 192.168.20.x)

---

### Sucursal 3 (Red LAN: 192.168.30.0/24)

| VLAN ID | Nombre | Subred | Gateway | Hosts Útiles | Dispositivos Reales | Descripción |
|---------|--------|--------|---------|--------------|---------------------|-------------|
| 31 | VLAN_GERENCIA_S3 | 192.168.30.0/27 | 192.168.30.1 | 30 | 12 PCs + 12 VoIP = 24 | Gerencia y administración |
| 32 | VLAN_CAJAS_S3 | 192.168.30.32/29 | 192.168.30.33 | 6 | 3 PCs + 3 VoIP = 6 | Cajas y atención |
| 33 | VLAN_RECEPCION_S3 | 192.168.30.40/29 | 192.168.30.41 | 6 | 2 PCs + 2 VoIP = 4 | Recepción |
| 34 | VLAN_SEGURIDAD_S3 | 192.168.30.48/29 | 192.168.30.49 | 6 | 1 PC + 1 VoIP = 2 | Seguridad física |
| 35 | VLAN_CAMARAS_S3 | 192.168.30.64/26 | 192.168.30.65 | 62 | 29 cámaras + 1 NVR = 30 | Cámaras de vigilancia |
| 36 | VLAN_CAJEROS_S3 | 192.168.30.128/29 | 192.168.30.129 | 6 | 2 cajeros ATM | Cajeros automáticos |
| 37 | VLAN_CLIENTES_S3 | 192.168.30.192/26 | 192.168.30.193 | 62 | WiFi público | WiFi clientes (Guest) |
| 38 | VLAN_MGMT_S3 | 192.168.30.136/29 | 192.168.30.137 | 6 | Equipos de red | Gestión local |

#### Distribución de IPs - Sucursal 3
(Misma estructura que Sucursales 1 y 2, con rango 192.168.30.x)

---

## Políticas de Seguridad y Control de Acceso

### Acceso entre Ubicaciones (VPN)

```
┌─────────────────────────────────────────────────────────────┐
│  Comunicación SOLO a través de túneles VPN IPSec cifrados  │
└─────────────────────────────────────────────────────────────┘

Sucursal 1 ──(VPN)──> Central ──(VPN)──> Sucursal 2
     │                                        │
     └────────── Sin conexión directa ───────┘

Reglas:
✓ Sucursales pueden acceder a servidores en Central (VLAN 100)
✓ Central TI puede gestionar equipos en sucursales (VLAN MGMT)
✓ Tráfico VoIP entre ubicaciones (si PBX centralizada)
✗ Sucursales NO pueden acceder entre sí directamente
✗ Solo tráfico autorizado pasa por VPN (ACLs estrictas)
```

### Matriz de Acceso entre VLANs - CENTRAL

#### Central
```
┌─────────────────┬─────┬────┬─────┬────────┬──────┬──────────┬──────────┐
│ Origen/Destino  │ ADM │ TI │ FIN │ CAMARAS│ VOIP │ SERVERS  │ MGMT     │
├─────────────────┼─────┼────┼─────┼────────┼──────┼──────────┼──────────┤
│ Administración  │  ✓  │ ✗  │  ✓  │   ✓    │  ✓   │    ✓     │    ✗     │
│ TI              │  ✓  │ ✓  │  ✓  │   ✓    │  ✓   │    ✓     │    ✓     │
│ Finanzas        │  ✓  │ ✗  │  ✓  │   ✗    │  ✓   │    ✓     │    ✗     │
│ Cámaras         │  ✗  │ ✓  │  ✗  │   ✓    │  ✗   │    ✗     │    ✗     │
│ VoIP            │  ✓  │ ✓  │  ✓  │   ✗    │  ✓   │    ✓     │    ✗     │
│ Servidores      │  ✓  │ ✓  │  ✓  │   ✗    │  ✓   │    ✓     │    ✗     │
│ Management      │  ✗  │ ✓  │  ✗  │   ✗    │  ✗   │    ✗     │    ✓     │
└─────────────────┴─────┴────┴─────┴────────┴──────┴──────────┴──────────┘
```

### Matriz de Acceso entre VLANs - SUCURSALES (aplica a S1, S2, S3)

```
┌─────────────────┬────┬──────┬──────┬──────┬────────┬────────┬──────────┬──────┐
│ Origen/Destino  │GER │ CAJAS│ RECEP│ SEG  │ CAMARAS│ CAJEROS│ CLIENTES │ MGMT │
├─────────────────┼────┼──────┼──────┼──────┼────────┼────────┼──────────┼──────┤
│ Gerencia        │ ✓  │  ✓   │  ✓   │  ✓   │   ✓    │   ✓    │    ✗     │  ✗   │
│ Cajas           │ ✗  │  ✓   │  ✓   │  ✗   │   ✗    │   ✓    │    ✗     │  ✗   │
│ Recepción       │ ✗  │  ✓   │  ✓   │  ✗   │   ✗    │   ✗    │    ✓     │  ✗   │
│ Seguridad       │ ✗  │  ✗   │  ✗   │  ✓   │   ✓    │   ✗    │    ✗     │  ✗   │
│ Cámaras         │ ✗  │  ✗   │  ✗   │  ✓   │   ✓    │   ✗    │    ✗     │  ✗   │
│ Cajeros         │ ✗  │  ✗   │  ✗   │  ✗   │   ✗    │   ✓    │    ✗     │  ✗   │
│ Clientes (WiFi) │ ✗  │  ✗   │  ✗   │  ✗   │   ✗    │   ✗    │    ✓     │  ✗   │
│ Management      │ ✗  │  ✗   │  ✗   │  ✗   │   ✗    │   ✗    │    ✗     │  ✓   │
└─────────────────┴────┴──────┴──────┴──────┴────────┴────────┴──────────┴──────┘

NOTA: VoIP está integrado en las VLANs de datos con QoS, no separado.
Todas las VLANs pueden comunicarse con servidores en Central vía VPN.
```

### Reglas Generales de Firewall

#### Todas las VLANs
- ✓ Permitir acceso a Internet (puerto 80, 443) - Filtrado por política
- ✓ Permitir DNS (puerto 53) hacia servidores internos o externos autorizados
- ✓ Permitir NTP (puerto 123) hacia servidor central
- ✗ Denegar acceso directo entre sucursales (debe pasar por Central y ser autorizado)

#### VLANs de Cámaras (VLAN 40 Central, VLANs 15, 25, 35 Sucursales)
- ✓ Solo tráfico RTSP/ONVIF/HTTP hacia NVR local
- ✓ Streaming remoto hacia Central TI vía VPN (solo monitoreo)
- ✗ Sin acceso a Internet
- ✗ Sin navegación web
- ✗ Aisladas de otras VLANs excepto Seguridad
- **Port Security habilitado**: Máximo 1 MAC por puerto

#### VLANs de Cajeros (VLAN 16, 26, 36)
- ✓ Solo conexión cifrada hacia servidores bancarios en Central (puerto 443)
- ✓ Comunicación con VLAN Servidores Central vía VPN (túnel dedicado)
- ✗ Todo otro tráfico denegado (Whitelist estricta)
- ✗ Sin acceso a Internet público
- **Port Security habilitado**: Máximo 1 MAC por puerto (MAC del cajero registrada)
- **802.1X**: Autenticación por certificado del cajero

#### VLANs de Clientes WiFi Guest (VLANs 17, 27, 37)
- ✓ Solo acceso a Internet (NAT hacia afuera)
- ✗ Sin acceso a ninguna red interna (firewall estricto)
- ✗ Aislamiento entre clientes (Client Isolation activo)
- ✗ Sin acceso a gateway interno
- **Captive Portal**: Página de bienvenida/términos de servicio
- **Limitación de ancho de banda**: 5 Mbps por cliente

#### VLANs de Finanzas, Cajeros, Gerencia
- ✓ Control de acceso por 802.1X (autenticación obligatoria)
- ✓ Registro de todo el tráfico (NetFlow/Syslog)
- **DHCP Snooping**: Activo
- **Dynamic ARP Inspection**: Activo
- **IP Source Guard**: Activo

#### VLAN de Management (VLANs 200, 18, 28, 38)
- ✓ Acceso solo desde VLAN TI Central (10.1.20.0/27)
- ✓ SSH/HTTPS solamente (puertos 22, 443)
- ✗ Denegar Telnet, HTTP, SNMP v1/v2
- **Autenticación**: RADIUS + TACACS+
- **Logging completo** de todos los accesos

---

## Configuración de QoS (Quality of Service)

### Prioridades de Tráfico

| Prioridad | CoS | DSCP | Tipo de Tráfico |
|-----------|-----|------|-----------------|
| 1 (Más alta) | 6-7 | EF (46) | VoIP (voz) - RTP |
| 2 | 5 | AF41 (34) | VoIP (señalización) - SIP |
| 3 | 4 | AF31 (26) | Cámaras/Video en vivo |
| 4 | 3 | AF21 (18) | Transacciones bancarias/Cajeros |
| 5 | 2 | AF11 (10) | Tráfico administrativo/gestión |
| 6 (Más baja) | 0-1 | BE (0) | WiFi Guest/Internet/Bulk data |

**Configuración en cada Switch/Router:**
- Voice VLAN marcado automáticamente con CoS 5
- Cámaras marcadas con DSCP AF31
- Cajeros marcados con DSCP AF21
- Ancho de banda garantizado para voz: 30%
- Ancho de banda máximo WiFi Guest: 10%

---

## Infraestructura de Red

### Equipos Requeridos

#### Central (Oficina Principal)
- **Firewall/Router VPN Principal**: FortiGate 100F, pfSense o Cisco ASA 5516-X
  - 2 WAN (failover), capacidad VPN Site-to-Site
  - IPS/IDS, filtrado de contenido, antivirus
- **Core Switch L3**: Cisco Catalyst 3650/3850 o HP Aruba 2930F (48 puertos)
  - Enrutamiento entre VLANs
  - Soporte PoE+ (802.3at) para VoIP y APs
  - Apilable/redundante
- **Distribution Switches L2**: 2-3 switches de 24-48 puertos con PoE+
  - Para conectar dispositivos finales
- **Servidor Físico o Virtual**: Para VLAN 100
  - VMware ESXi, Proxmox o Hyper-V
  - Alojar: AD, DNS, DHCP, File Server, DB, VPN, Backup
- **Servidor PBX VoIP**: Dedicado o VM
  - FreePBX, 3CX o Asterisk
- **NVR Central**: Hikvision, Dahua o similar (soporte 64+ cámaras)
- **UPS**: APC Smart-UPS 3000VA o superior
- **Conexión WAN**: 100-200 Mbps simétrico, IP estática

#### Por Sucursal (x3)
- **Firewall/Router VPN**: FortiGate 60F, pfSense o Cisco ISR 1100
  - Cliente VPN Site-to-Site hacia Central
  - Firewall local, NAT
- **Switch Core L3**: 24 puertos PoE+ (Cisco 2960-X o similar)
  - Enrutamiento entre VLANs locales
  - PoE para VoIP, cámaras, APs
- **Access Switch L2 PoE**: 1-2 switches de 24 puertos (para expansión)
- **Access Points WiFi**: 7 unidades - Ubiquiti UniFi AC Pro o similar
  - VLAN tagging para Guest WiFi
  - Gestión centralizada (Controller)
- **NVR Local**: Hikvision/Dahua (soporte 32 cámaras)
- **UPS**: APC Smart-UPS 1500VA
- **Conexión WAN**: 50-100 Mbps, IP estática preferiblemente

---

## Conectividad VPN (Site-to-Site)

### Arquitectura Hub-and-Spoke

```
                    ┌───────────────┐
                    │   CENTRAL     │
                    │  (HUB/Server) │
                    │ IP Pública: X │
                    │ 10.1.0.0/16   │
                    └───────┬───────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    ┌────▼────┐        ┌────▼────┐       ┌────▼────┐
    │ SUCURSAL│        │ SUCURSAL│       │ SUCURSAL│
    │    1    │        │    2    │       │    3    │
    │ (Spoke) │        │ (Spoke) │       │ (Spoke) │
    │IP Púb: Y│        │IP Púb: Z│       │IP Púb: W│
    │192.168. │        │192.168. │       │192.168. │
    │10.0/24  │        │20.0/24  │       │30.0/24  │
    └─────────┘        └─────────┘       └─────────┘

- Todas las conexiones sobre Internet público
- Túneles IPSec cifrados
- Sucursales NO se comunican entre sí directamente
- Todo tráfico inter-sucursal pasa por Central (Hub)
```

### Configuración de Túneles VPN IPSec

#### Parámetros de Seguridad (Fase 1 - IKEv2)
- **Versión IKE**: IKEv2 (preferido) o IKEv1
- **Autenticación**: Pre-Shared Key (PSK) o Certificados X.509
- **Cifrado**: AES-256-CBC o AES-256-GCM
- **Integridad**: SHA-256 o SHA-384
- **Grupo Diffie-Hellman**: Grupo 14 (2048-bit) o superior (19, 20)
- **Lifetime**: 28800 segundos (8 horas)

#### Parámetros de Seguridad (Fase 2 - IPSec)
- **Protocolo**: ESP
- **Cifrado**: AES-256-CBC o AES-256-GCM
- **Integridad**: SHA-256 o SHA-384
- **PFS (Perfect Forward Secrecy)**: Grupo 14 o superior
- **Lifetime**: 3600 segundos (1 hora)

#### Túneles Configurados

| Túnel | Endpoint Central | Endpoint Remoto | Redes Locales Central | Redes Remotas | IP Virtual Túnel |
|-------|------------------|-----------------|----------------------|---------------|------------------|
| VPN-S1 | WAN_Central | WAN_Sucursal1 | 10.1.0.0/16 | 192.168.10.0/24 | 172.16.0.0/30 |
| VPN-S2 | WAN_Central | WAN_Sucursal2 | 10.1.0.0/16 | 192.168.20.0/24 | 172.16.0.4/30 |
| VPN-S3 | WAN_Central | WAN_Sucursal3 | 10.1.0.0/16 | 192.168.30.0/24 | 172.16.0.8/30 |

**Direcciones IP de Túnel (punto a punto):**
- **Túnel Central ↔ S1**: 
  - Central: 172.16.0.1/30
  - Sucursal 1: 172.16.0.2/30
- **Túnel Central ↔ S2**: 
  - Central: 172.16.0.5/30
  - Sucursal 2: 172.16.0.6/30
- **Túnel Central ↔ S3**: 
  - Central: 172.16.0.9/30
  - Sucursal 3: 172.16.0.10/30

### ACLs para Tráfico VPN (Crypto ACL)

**En Central - Tráfico interesante hacia Sucursal 1:**
```
permit ip 10.1.0.0 0.0.255.255 192.168.10.0 0.0.0.255
```

**En Sucursal 1 - Tráfico interesante hacia Central:**
```
permit ip 192.168.10.0 0.0.0.255 10.1.0.0 0.0.255.255
```

*(Replicar para S2 y S3 con sus respectivas redes)*

### Monitoreo VPN
- **Keepalive/DPD**: 10 segundos (Dead Peer Detection)
- **Logs**: Registrar establecimiento, caída y reconexión de túneles
- **Alertas**: Notificación inmediata si túnel cae
- **Backup**: Considerar túnel secundario por ISP alternativo
- **Throughput esperado**: 
  - Sucursal a Central: 20-50 Mbps
  - Latencia objetivo: < 50ms

### Rutas Estáticas (en cada ubicación)

**Central:**
```
ip route 192.168.10.0 255.255.255.0 172.16.0.2 (vía túnel S1)
ip route 192.168.20.0 255.255.255.0 172.16.0.6 (vía túnel S2)
ip route 192.168.30.0 255.255.255.0 172.16.0.10 (vía túnel S3)
```

**Sucursal 1:**
```
ip route 10.1.0.0 255.255.0.0 172.16.0.1 (vía túnel a Central)
ip route 192.168.20.0 255.255.255.0 172.16.0.1 (vía Central)
ip route 192.168.30.0 255.255.255.0 172.16.0.1 (vía Central)
```

*(Replicar en S2 y S3)*

---

## Servicios de Red

### DHCP

**Estrategia:** Servidor DHCP centralizado en Central, con DHCP local en sucursales para resiliencia.

#### Central (Servidor Principal)
- **Servidor DHCP**: 10.1.100.7 (VLAN 100)
- **Pools por VLAN**:
  - VLAN 10 (Admin): 10.1.10.10 - 10.1.10.25
  - VLAN 20 (TI): 10.1.20.10 - 10.1.20.25
  - VLAN 30 (Finanzas): 10.1.30.5 - 10.1.30.10
  - VLAN 40 (Cámaras): Estático (sin DHCP)
  - VLAN 50 (VoIP): 10.1.50.10 - 10.1.50.25
  - VLAN 200 (MGMT): Estático (sin DHCP)
- **Lease Time**: 
  - PCs/VoIP: 8 horas
  - Dispositivos móviles: 2 horas
- **DHCP Relay**: Configurado en gateway de cada VLAN (Switch L3)
- **Opciones**:
  - Opción 3: Default Gateway (por VLAN)
  - Opción 6: DNS Server (10.1.100.8)
  - Opción 42: NTP Server (10.1.100.8)
  - Opción 66: TFTP Server para VoIP (10.1.100.8)
  - Opción 150: PBX Server (10.1.50.2)

#### Sucursales (Servidores Locales - Autonomía)
**Sucursal 1 - DHCP en Router/Switch local:**
- VLAN 11: 192.168.10.10 - 192.168.10.25
- VLAN 12: 192.168.10.34 - 192.168.10.38
- VLAN 13: 192.168.10.42 - 192.168.10.45
- VLAN 14: 192.168.10.50 - 192.168.10.52
- VLAN 15: Estático (cámaras)
- VLAN 16: Estático (cajeros)
- VLAN 17: 192.168.10.194 - 192.168.10.254 (WiFi Guest)
- VLAN 18: Estático (management)
- **Lease Time**: 4 horas (menor que central por movilidad)

*(Replicar en S2 con 192.168.20.x y S3 con 192.168.30.x)*

### DNS

#### Servidor DNS Primario (Central)
- **IP**: 10.1.100.8
- **Zona Interna**: banco.local
- **Registros**:
  - central.banco.local → 10.1.100.0/28 (red servidores)
  - sucursal1.banco.local → 192.168.10.0/24
  - sucursal2.banco.local → 192.168.20.0/24
  - sucursal3.banco.local → 192.168.30.0/24
- **Forwarders** (DNS externos):
  - 8.8.8.8 (Google)
  - 1.1.1.1 (Cloudflare)
- **DNS Reverso**: Configurado para todas las subredes internas
- **DNSSEC**: Habilitado para zonas internas

#### Servidores DNS Secundarios
- **Sucursales**: Usar DNS Central como primario vía VPN
- **Fallback**: 8.8.8.8, 1.1.1.1 si VPN cae

### NTP (Sincronización de Tiempo)

**Crítico para logs, autenticación y transacciones bancarias**

- **Servidor NTP Principal**: 10.1.100.8 (Central)
- **Stratum**: 2 (sincronizado con pool.ntp.org)
- **Fuentes Externas**:
  - time.google.com
  - time.cloudflare.com
  - pool.ntp.org
- **Clientes**: Todos los dispositivos de red apuntan a 10.1.100.8
- **Sucursales**: Sincronizar con Central vía VPN (puerto UDP 123)
- **Precisión requerida**: ±100ms entre todas las ubicaciones

---

## Monitoreo y Gestión

### SNMP
- **Versión**: SNMPv3 (seguro)
- **Comunidades**: Autenticación por usuario
- **Traps**: Configurados para alertas críticas

### Syslog
- **Servidor**: 10.1.100.30
- **Nivel de logs**: Warning y superior
- **Retención**: 90 días

### Herramientas Recomendadas
- **Monitoreo**: Zabbix, PRTG o SolarWinds
- **Análisis de tráfico**: Wireshark, NetFlow
- **Gestión**: Cisco DNA Center o similar

---

## Seguridad Adicional

### Port Security (Switches)

**VLANs Críticas - Modo Restrictivo:**
- **VLAN 30 (Finanzas)**: 1 MAC por puerto, violation: shutdown
- **VLAN 40 (Cámaras Central)**: 1 MAC por puerto, violation: restrict
- **VLAN 100 (Servidores)**: 1 MAC por puerto, violation: shutdown
- **VLANs 15, 25, 35 (Cámaras Sucursales)**: 1 MAC por puerto, violation: restrict
- **VLANs 16, 26, 36 (Cajeros)**: 1 MAC registrada permanentemente, violation: shutdown

**VLANs Normales - Modo Moderado:**
- **Otras VLANs de datos**: 2-3 MACs por puerto, violation: restrict
- **VLANs WiFi Guest**: Sin port security (por naturaleza inalámbrica)

**Configuración de ejemplo (Cisco):**
```
interface GigabitEthernet1/0/10
 description CAJERO-ATM-01
 switchport mode access
 switchport access vlan 16
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### DHCP Snooping

- **Habilitado en todas las VLANs** excepto Management
- **Trust ports**: Solo uplinks a routers y servidores DHCP
- **Untrust ports**: Todos los puertos de usuario
- **Binding database**: Guardada en NVRAM o servidor externo
- **Rate limit**: 15 paquetes/segundo en puertos untrusted

**Configuración de ejemplo:**
```
ip dhcp snooping
ip dhcp snooping vlan 10,11,12,13,20,30
no ip dhcp snooping information option
interface GigabitEthernet1/0/1
 ip dhcp snooping trust  ! Uplink a servidor DHCP
```

### Dynamic ARP Inspection (DAI)

- **Habilitado en VLANs críticas**: 10, 20, 30, 100 (Central) y 11, 12, 16 (Sucursales)
- **Basado en DHCP Snooping binding table**
- **Trust ports**: Uplinks y conexiones a routers
- **Validación**: src-mac, dst-mac, ip
- **Rate limit**: 15 paquetes/segundo

**Configuración:**
```
ip arp inspection vlan 10,20,30,100
ip arp inspection validate src-mac dst-mac ip
interface GigabitEthernet1/0/1
 ip arp inspection trust
```

### IP Source Guard

- **Habilitado en puertos de usuario** de VLANs críticas
- **Filtra por IP y MAC** según DHCP snooping table
- **Previene IP spoofing**

```
interface range GigabitEthernet1/0/2-24
 ip verify source port-security
```

### 802.1X (Port-Based NAC - Network Access Control)

**VLANs con autenticación obligatoria:**
- VLAN 10 (Administración)
- VLAN 20 (TI)
- VLAN 30 (Finanzas)
- VLAN 100 (Servidores)
- VLANs 16, 26, 36 (Cajeros - certificados)

**Servidor RADIUS/TACACS+:**
- **IP**: 10.1.100.6 (AD integrado)
- **Puerto**: 1812/1813 (RADIUS)
- **Autenticación**:
  - PCs: Usuario/Contraseña (EAP-PEAP o EAP-TLS)
  - Cajeros: Certificados X.509 (EAP-TLS)
- **VLAN dinámica**: Asignación según grupo de usuario
- **VLAN Guest**: Para dispositivos no autenticados (opcional)

**Configuración de ejemplo:**
```
aaa new-model
aaa authentication dot1x default group radius
radius server RADIUS-CENTRAL
 address ipv4 10.1.100.6 auth-port 1812 acct-port 1813
 key SuperSecretKey123

interface GigabitEthernet1/0/5
 authentication port-control auto
 dot1x pae authenticator
```

### Spanning Tree Protocol (STP) - Seguridad

- **BPDU Guard**: En todos los puertos de usuario
- **Root Guard**: En puertos de distribución
- **Loop Guard**: En enlaces troncales
- **PortFast**: Solo en puertos de acceso

```
spanning-tree portfast bpduguard default
interface GigabitEthernet1/0/2-24
 spanning-tree portfast
```

### VLANs Privadas (PVLAN) - WiFi Guest

- **Configurar en VLANs 17, 27, 37** (WiFi Guest)
- **Modo**: Community VLAN o Isolated VLAN
- **Objetivo**: Clientes WiFi no pueden verse entre sí
- **Solo pueden comunicarse con**: Gateway y DNS

### Listas de Acceso (ACLs) en Switches/Routers

**Ejemplo - Proteger VLAN Management:**
```
ip access-list extended ACL-MGMT-ACCESS
 permit tcp 10.1.20.0 0.0.0.31 192.168.10.136 0.0.0.7 eq 22
 permit tcp 10.1.20.0 0.0.0.31 192.168.10.136 0.0.0.7 eq 443
 deny   ip any 192.168.10.136 0.0.0.7 log
 permit ip any any

interface Vlan18
 ip access-group ACL-MGMT-ACCESS in
```

**Ejemplo - Proteger Cajeros (solo a servidores bancarios):**
```
ip access-list extended ACL-CAJEROS
 permit tcp 192.168.10.128 0.0.0.7 10.1.100.0 0.0.0.15 eq 443
 deny   ip 192.168.10.128 0.0.0.7 any log
 
interface Vlan16
 ip access-group ACL-CAJEROS in
```

---

## Plan de Implementación

### Fase 1: Central (Semanas 1-2)
1. Configurar Core Switch con VLANs
2. Implementar servidor DHCP/DNS
3. Configurar firewall y políticas de seguridad
4. Implementar servidor PBX
5. Configurar QoS
6. Pruebas de conectividad

### Fase 2: Sucursal Piloto (Semanas 3-4)
1. Implementar equipamiento en Sucursal 1
2. Configurar VPN Site-to-Site
3. Configurar VLANs locales
4. Implementar Access Points
5. Pruebas de conectividad y VPN
6. Ajustes y optimización

### Fase 3: Sucursales 2 y 3 (Semanas 5-6)
1. Replicar configuración de Sucursal 1
2. Configurar túneles VPN
3. Pruebas integrales
4. Documentación final

### Fase 4: Optimización (Semana 7)
1. Monitoreo de rendimiento
2. Ajuste de QoS
3. Capacitación a personal de TI
4. Entrega de documentación

---

## Mantenimiento y Respaldos

### Respaldos de Configuración
- **Frecuencia**: Diaria automática
- **Retención**: 30 días
- **Almacenamiento**: Servidor de respaldos + nube

### Actualizaciones
- **Firmware de red**: Trimestral (en ventana de mantenimiento)
- **Parches de seguridad**: Mensual
- **Revisión de políticas**: Semestral

### Documentación Requerida
- Diagramas de red (físico y lógico)
- Tablas de direccionamiento IP
- Configuraciones de equipos
- Procedimientos de troubleshooting
- Contactos de soporte
