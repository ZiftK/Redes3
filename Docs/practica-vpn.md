# Práctica de Laboratorio: VPN Site-to-Site

## Objetivo

Implementar un túnel VPN IPSec entre la **oficina Central** y **Sucursal-1** de Banquinta. Esta practica cumple solo una parte del proyecto completo descrito en [main](main.md).

**Se pretende:**
- Configurar túneles VPN IPSec
- Usar NAT Exemption (para que VPN y NAT funcionen juntos)
- Verificar túneles VPN

---

## Escenario

**Dispositivos:**
- **Router-Central**: Red LAN 192.168.1.0/24
- **Sucursal-1**: Red LAN 192.168.2.0/24
- **ISP**: Simula Internet

**Topología:**

```
[PC-Central]          [PC-Sucursal]
     |                      |
192.168.1.0/24        192.168.2.0/24
     |                      |
[Router-Central]      [Sucursal-1]
     |                      |
200.2.2.1/30          200.1.1.1/30
     |                      |
     +------[ISP]----------+
```

**Se necesita:**
1. PCs en redes internas que puedan comunicarse entre sí (VPN)
2. PCs que puedan acceder a Internet (NAT)
3. Tráfico entre PCs en redes internas cifrado (IPSec)

---

## Configuración

### 1. Router Central

**Interfaces y ruteo:**
```cisco
ena
conf t
hostname Router-Central

interface g0/0/0
 ip add 192.168.1.1 255.255.255.0
 no sh

interface g0/0/1
 ip add 200.2.2.1 255.255.255.252
 no sh

ip route 0.0.0.0 0.0.0.0 200.2.2.2
```

**NAT con Exemption**
```cisco
access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 permit ip 192.168.1.0 0.0.0.255 any

ip nat inside source list 101 interface g0/0/1 overload

interface g0/0/0
 ip nat inside
interface g0/0/1
 ip nat outside
```

**VPN:**
```cisco
access-list 100 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key BanquintaSecure2025! address 200.1.1.1

crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac

crypto map MYMAP 10 ipsec-isakmp
 set peer 200.1.1.1
 set transform-set MYSET
 match address 100

interface g0/0/1
 crypto map MYMAP

exit
write memory
```

---

### 2. Router ISP

```cisco
ena
conf t
hostname ISP

int g0/0/1
 ip add 200.2.2.2 255.255.255.252
 no sh

int g0/0/0
 ip add 200.1.1.2 255.255.255.252
 no sh

ip route 192.168.1.0 255.255.255.0 200.2.2.1
ip route 192.168.2.0 255.255.255.0 200.1.1.1

exit
write memory
```

---

### 3. Router Sucursal-1

**Interfaces y ruteo:**
```cisco
ena
conf t
hostname Sucursal-1

int g0/0/0
 ip add 192.168.2.1 255.255.255.0
 no sh

int g0/0/1
 ip add 200.1.1.1 255.255.255.252
 no sh

ip route 0.0.0.0 0.0.0.0 200.1.1.2
```

**VPN:**
```cisco
access-list 100 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key BanquintaSecure2025! address 200.2.2.1

crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac

crypto map MYMAP 10 ipsec-isakmp
 set peer 200.2.2.1
 set transform-set MYSET
 match address 100
```

**NAT con Exemption (IMPORTANTE):**
```cisco
access-list 101 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 101 permit ip 192.168.2.0 0.0.0.255 any

ip nat inside source list 101 interface g0/0/1 overload

interface g0/0/0
 ip nat inside
interface g0/0/1
 ip nat outside

interface g0/0/1
 crypto map MYMAP

exit
write memory
```

Sin no configuramos el NAT exemption en **ambos routers**, NAT traduce IP **antes** de que el VPN actue por lo que el VPN no reconoce el tráfico. Con esta configuración, los paquetes van directo al túnel cifrado sin pasar por la traducción del NAT.

---

## Verificación

```cisco
! Ver túnel VPN
show crypto isakmp sa
show crypto ipsec sa

! Probar conectividad
ping 192.168.2.1 source 192.168.1.1

! Ver NAT
show ip nat translations
```

---

## Conceptos Clave

### ¿Por qué NAT Exemption en AMBOS routers?

**Flujo de procesamiento:**
```
Paquete → ACL → NAT → VPN → Salida
```

**Sin NAT Exemption en Router Central:**
- PC Central (192.168.1.10) → PC Sucursal (192.168.2.5)
- NAT traduce 192.168.1.10 → 200.2.2.1
- VPN no reconoce el tráfico (espera ver 192.168.1.0/24) ❌

**Con NAT Exemption en Router Central:**
- PC Central (192.168.1.10) → PC Sucursal (192.168.2.5)
- ACL 101 dice "NO traducir tráfico a 192.168.2.0/24"
- VPN reconoce y cifra el tráfico ✅

**Lo mismo aplica para Router Sucursal-1**

### ACLs de NAT Exemption

```cisco
! Router Central
access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255  ! No traducir VPN
access-list 101 permit ip 192.168.1.0 0.0.0.255 any                   ! Traducir Internet

! Router Sucursal-1
access-list 101 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255  ! No traducir VPN
access-list 101 permit ip 192.168.2.0 0.0.0.255 any                   ! Traducir Internet
``` 


## Expandir a Múltiples Sucursales

Para agregar Sucursal-2 y Sucursal-3:

**En Router Central, agregar:**
```cisco
! ACLs VPN
access-list 110 permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 120 permit ip 192.168.1.0 0.0.0.255 192.168.4.0 0.0.0.255

! NAT Exemption (actualizar ACL 101)
no access-list 101
access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.4.0 0.0.0.255
access-list 101 permit ip 192.168.1.0 0.0.0.255 any

! Crypto maps
crypto map MYMAP 20 ipsec-isakmp
 set peer 200.3.3.1
 set transform-set MYSET
 match address 110

crypto map MYMAP 30 ipsec-isakmp
 set peer 200.4.4.1
 set transform-set MYSET
 match address 120

! Claves
crypto isakmp key BanquitaS2-2025! address 200.3.3.1
crypto isakmp key BanquitaS3-2025! address 200.4.4.1
```

**En cada Sucursal, actualizar NAT Exemption:**
```cisco
! Ejemplo para Sucursal-2
access-list 101 deny ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 101 deny ip 192.168.3.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 deny ip 192.168.3.0 0.0.0.255 192.168.4.0 0.0.0.255
access-list 101 permit ip 192.168.3.0 0.0.0.255 any
```

---

## Configuraciones Completas (Listas para Copiar)

### Router Central
```cisco
ena
conf t
hostname Router-Central

interface g0/0/0
 ip add 192.168.1.1 255.255.255.0
 no sh

interface g0/0/1
 ip add 200.2.2.1 255.255.255.252
 no sh

ip route 0.0.0.0 0.0.0.0 200.2.2.2

access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 permit ip 192.168.1.0 0.0.0.255 any

ip nat inside source list 101 interface g0/0/1 overload

interface g0/0/0
 ip nat inside
interface g0/0/1
 ip nat outside

access-list 100 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key BanquintaSecure2025! address 200.1.1.1

crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac

crypto map MYMAP 10 ipsec-isakmp
 set peer 200.1.1.1
 set transform-set MYSET
 match address 100

interface g0/0/1
 crypto map MYMAP

exit
write memory
```

### Router ISP
```cisco
ena
conf t
hostname ISP

int g0/0/1
 ip add 200.2.2.2 255.255.255.252
 no sh

int g0/0/0
 ip add 200.1.1.2 255.255.255.252
 no sh

ip route 192.168.1.0 255.255.255.0 200.2.2.1
ip route 192.168.2.0 255.255.255.0 200.1.1.1

exit
write memory
```

### Router Sucursal-1
```cisco
ena
conf t
hostname Sucursal-1

int g0/0/0
 ip add 192.168.2.1 255.255.255.0
 no sh

int g0/0/1
 ip add 200.1.1.1 255.255.255.252
 no sh

ip route 0.0.0.0 0.0.0.0 200.1.1.2

access-list 100 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255

access-list 101 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 101 permit ip 192.168.2.0 0.0.0.255 any

ip nat inside source list 101 interface g0/0/1 overload

interface g0/0/0
 ip nat inside
interface g0/0/1
 ip nat outside

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key BanquintaSecure2025! address 200.2.2.1

crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac

crypto map MYMAP 10 ipsec-isakmp
 set peer 200.2.2.1
 set transform-set MYSET
 match address 100

interface g0/0/1
 crypto map MYMAP

exit
write memory
```

---

## Resumen

**Logramos:**
- Túnel VPN seguro (Central <-> Sucursal-1)
- Cifrado AES-256
- Acceso a Internet (NAT)
- NAT Exemption funcionando en AMBOS routers

**Puntos importantes:**
1. **NAT Exemption es obligatorio en AMBOS routers** cuando tienen VPN y NAT
2. Políticas ISAKMP deben ser idénticas en ambos extremos
3. ACLs de VPN son "espejo" (origen ↔ destino)
4. Claves pre-compartidas deben coincidir exactamente

**Siguiente paso:** Expandir a Sucursal-2 y Sucursal-3
