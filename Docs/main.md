# Proyecto de Red para Banquinta

## 1. Descripción del Proyecto

**Banquinta** es un banco que necesita conectar su oficina central con **3 sucursales** ubicadas en diferentes estados. 

**Situación actual:**
- La oficina central tiene los servidores principales
- Las 3 sucursales NO tienen red implementada
- No hay conexión entre las oficinas
- Se necesita acceso seguro a los servidores centrales desde las sucursales

**Problema:**  
Las sucursales están en diferentes estados y no pueden conectarse por cable. Se necesita una solución que use Internet pero que sea segura para proteger datos bancarios.

---

## 2. Solución Propuesta

### Topología: Estrella (Hub-and-Spoke)

```
    Sucursal-1
         |
    [Central]---Sucursal-2
         |
    Sucursal-3
```

**Justificación:**
- Todas las sucursales se conectan solo con la central
- Administración centralizada, lo que la hace mas facil de manejar y monitorear
- Los servidores están en la central, entonces tiene sentido

### Tecnologías Usadas

**1. VPN Site-to-Site con IPSec**
- Crea túneles cifrados entre cada sucursal y la central lo que protege los datos bancarios al viajar por Internet

**2. NAT (Network Address Translation)**
- Permite que los PCs accedan a Internet
- Usa NAT Exemption para que VPN funcione bien
- Optimiza el uso de IPs públicas

**3. Enrutamiento Estático**
- Rutas configuradas a mano en cada router
- Suficiente para 3 sucursales
- Fácil de entender y configurar

---

## 3. Direccionamiento IP

### Redes LAN (Privadas)
- **Central**: 192.168.1.0/24
- **Sucursal-1**: 192.168.2.0/24
- **Sucursal-2**: 192.168.3.0/24
- **Sucursal-3**: 192.168.4.0/24

### Enlaces WAN (Públicas)
- **Central ↔ ISP**: 200.2.2.0/30
- **Sucursal-1 ↔ ISP**: 200.1.1.0/30
- **Sucursal-2 ↔ ISP**: 200.3.3.0/30
- **Sucursal-3 ↔ ISP**: 200.4.4.0/30

---


## 4. Equipamiento por Sucursal

### Dispositivos de Red por Sucursal
Cada una de las 3 sucursales tiene:

**Equipos de cómputo:**
- 13 PCs de gerencia
- 2 PCs de recepción
- 2 PCs de seguridad
- 3 PCs de cajeros
- **Total: 20 PCs**

**Telefonía y Seguridad:**
- 20 Teléfonos VoIP (uno por cada PC)
- 30 Cámaras de seguridad IP
- **Total: 50 dispositivos**

### Infraestructura de Red

**Routers:**
- 1 Router empresarial con soporte VPN IPSec por ubicación
- 4 routers en total (Central + 3 Sucursales)

**Switches:**
- **2 Switches PoE de 48 puertos** por sucursal para:
  - Teléfonos VoIP (alimentación por PoE)
  - Cámaras IP (alimentación por PoE)
  - PCs y otros dispositivos
- **1 Switch core** en Central para interconexión

**Cableado:**
- **Cat 6** para toda la instalación
- Considerando presupuesto y necesidades actuales
- Soporta hasta 1 Gbps (suficiente para el proyecto)

**Otros:**
- 1 UPS por ubicación para respaldo eléctrico

### Servicios de Internet

**Por ubicación:**
- Conexión principal: 20 Mbps mínimo
- 1 IP pública estática por router
- **Considerar:** Proveedor de respaldo para redundancia (futuro)
- SLA recomendado: 99% de disponibilidad

---

## 5. Segmentación de Red y Seguridad

### VLANs por Sucursal

Para mejor organización y seguridad, se segmentará cada sucursal en VLANs:

| VLAN | Departamento | Dispositivos | Descripción |
|------|--------------|--------------|-------------|
| **VLAN 10** | Gerencia | 13 PCs + 13 VoIP | Personal administrativo |
| **VLAN 20** | Cajeros | 3 PCs + 3 VoIP | Transacciones bancarias |
| **VLAN 30** | Recepción | 2 PCs + 2 VoIP | Atención al cliente |
| **VLAN 40** | Seguridad | 2 PCs + 2 VoIP + 30 cámaras | Monitoreo y vigilancia |
| **VLAN 99** | Administración | Routers y Switches | Gestión de red |

### Direccionamiento por VLAN (Ejemplo Sucursal-1)

- **VLAN 10 (Gerencia)**: 192.168.10.0/24
- **VLAN 20 (Cajeros)**: 192.168.20.0/24
- **VLAN 30 (Recepción)**: 192.168.30.0/24
- **VLAN 40 (Seguridad)**: 192.168.40.0/24
- **VLAN 99 (Admin)**: 192.168.99.0/24

### Seguridad

**IPs Estáticas:**
- Asignación de IPs fijas para tener total control y rastreo de dispositivos

**Control de Acceso:**
- Bloqueo de sitios web no autorizados (redes sociales, streaming)
- WhiteList para sitios bancarios y herramientas de trabajo

**Políticas de Seguridad:**
- VLAN de Cajeros aislada (solo acceso a servidores centrales)
- VLAN de Seguridad con acceso solo a sistema de monitoreo
- Gerencia con acceso completo pero monitoreado

---

## 6. Consideraciones Técnicas Adicionales

### Cableado Estructurado

**Categoría de cable:**
- **Cat 6** para toda la infraestructura
- Velocidad: 1 Gbps (suficiente para las necesidades actuales)
- Más económico que Cat 6A o superior
- Adecuado para VoIP, cámaras IP y datos

### Switches PoE 

**Cálculo de puertos:**
- Por sucursal: 20 VoIP + 30 cámaras = 50 puertos PoE
- Solución: **2 switches de 48 puertos PoE** (96 puertos disponibles)

Aunque es poco probable que la cantidad de equipos cambie, dejamos un margen para hacer instalacion extra de camaras o telefonos VoIP.

### Redundancia y Disponibilidad

**Consideraciones:**
- **Proveedor de Internet:** Segundo ISP para respaldo 
- **SLA recomendado:** 99% de disponibilidad mensual, debido a la importancia de las transacciones.


### Control de Contenido Web

**Restricciones:**
- Bloqueo de redes sociales (Facebook, Instagram, Twitter)
- Bloqueo de streaming (YouTube, Netflix)
- Bloqueo de sitios de descarga
- Bloque de GPTs

**Lista blanca:**
- Sitios bancarios autorizados
- Herramientas de trabajo (correo corporativo, cloud)
- Sitios gubernamentales

**Implementación:** Mediante ACLs en routers o firewall básico


