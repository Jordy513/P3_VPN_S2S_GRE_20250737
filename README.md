# VPN Point-to-Point — Túnel GRE (Generic Routing Encapsulation)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es GRE?](#21-qué-es-gre)
   - [Estructura del Paquete GRE](#22-estructura-del-paquete-gre)
   - [GRE vs. IPSec vs. GRE+IPSec](#23-gre-vs-ipsec-vs-greipsec)
   - [Parámetros del Túnel Configurado](#24-parámetros-del-túnel-configurado)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router R1 — Site A](#41-router-r1--site-a)
   - [Router R2 — Site B](#42-router-r2--site-b)
   - [Configuración de PCs (Hosts)](#43-configuración-de-pcs-hosts)
5. [Verificación del Túnel](#5-verificación-del-túnel)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Site-to-Site Point-to-Point utilizando un túnel GRE (Generic Routing Encapsulation)** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración de una interfaz de túnel lógica (`Tunnel0`) utilizando el protocolo de encapsulación GRE para interconectar dos sitios a través de un segmento de red pública simulado (`192.168.1.0/24`).
* La capacidad de GRE para **transportar tráfico multicast y broadcast** sobre una red IP unicast, habilitando la ejecución de protocolos de enrutamiento dinámico (OSPF) directamente sobre el túnel.
* La propagación automática de rutas entre las subredes LAN privadas (`20.25.37.128/25` y `20.25.37.0/25`) mediante OSPF corriendo sobre el enlace virtual GRE.
* La diferencia fundamental entre GRE puro y las soluciones IPSec: GRE **encapsula** pero no **cifra** — estableciendo el punto de partida conceptual para entender por qué en producción se combina con IPSec.

---

## 2. Marco Teórico

### 2.1 ¿Qué es GRE?

GRE (Generic Routing Encapsulation — RFC 2784) es un protocolo de tunelización de **Capa 3** desarrollado originalmente por Cisco. Su función es encapsular paquetes de cualquier protocolo de red dentro de paquetes IP, creando un enlace virtual punto a punto entre dos routers sobre una red IP existente.

GRE es identificado como **IP Protocol 47** — al igual que TCP es protocolo 6 y UDP es protocolo 17, GRE tiene su propio número de protocolo en el campo Protocol del header IP externo.

Las características fundamentales de GRE son:

* **Agnóstico al protocolo de pasajero:** Puede encapsular no solo IPv4, sino también IPv6, MPLS, AppleTalk, IPX y cualquier otro protocolo de capa de red.
* **Soporte de multicast y broadcast:** A diferencia de IPSec puro, GRE puede transportar tráfico multicast y broadcast, lo que permite ejecutar protocolos de enrutamiento dinámico (OSPF, EIGRP, RIP) sobre el túnel.
* **Sin cifrado:** GRE únicamente encapsula — el contenido del paquete viaja en texto claro. No provee confidencialidad, integridad ni autenticación.
* **Overhead fijo:** GRE agrega 24 bytes de overhead al paquete original (20 bytes de header IP externo + 4 bytes de header GRE), reduciendo el MTU efectivo del payload a 1476 bytes sobre una red con MTU de 1500.

### 2.2 Estructura del Paquete GRE

Cuando un paquete IP original es encapsulado por GRE, la estructura resultante es la siguiente:

```
┌─────────────────────────────────────────────────────────────────┐
│              PAQUETE GRE ENCAPSULADO (viaja por Internet)       │
├──────────────────┬──────────────────┬──────────────────────────┤
│  IP Header ext.  │   GRE Header     │    Payload original       │
│  (src: R1 WAN)   │  (Protocol: 47)  │  (IP Header + Datos)      │
│  (dst: R2 WAN)   │  4 bytes         │  del host origen          │
│  20 bytes        │                  │                           │
└──────────────────┴──────────────────┴──────────────────────────┘
         │                                        │
    IP visible                            IP ORIGINAL cifrada
    en Internet                           (pero NO cifrada en GRE puro)
```

El router de destino recibe el paquete, identifica el Protocol 47 (GRE), desencapsula el header externo y el header GRE, y entrega el paquete original a la interfaz interna como si hubiera llegado directamente.

### 2.3 GRE vs. IPSec vs. GRE+IPSec

| Característica | GRE puro (este lab) | IPSec puro (policy-based) | GRE + IPSec |
|---|---|---|---|
| **Encapsulación** | ✅ Sí | ✅ Sí (modo túnel) | ✅ Sí |
| **Cifrado** | ❌ No | ✅ Sí (AES, 3DES) | ✅ Sí |
| **Multicast / Broadcast** | ✅ Sí | ❌ No | ✅ Sí (GRE lo provee) |
| **Routing dinámico sobre el túnel** | ✅ Sí (OSPF, EIGRP) | ❌ No directo | ✅ Sí |
| **Overhead** | 24 bytes | ~50–90 bytes | ~74–114 bytes |
| **Protocolo de transporte** | IP Protocol 47 | IP Protocol 50 (ESP) | IP 47 encapsulado en IP 50 |
| **Autenticación de origen** | ❌ No | ✅ Sí | ✅ Sí |
| **Caso de uso** | Labs, redes internas de confianza | VPN cifrada sin routing dinámico | VPN cifrada con routing dinámico |

GRE puro es el punto de partida conceptual. En producción, siempre se combina con IPSec para agregar cifrado, dando origen al modelo **GRE over IPSec** que es la base de soluciones como DMVPN.

### 2.4 Parámetros del Túnel Configurado

| Parámetro | Valor | Descripción |
|---|---|---|
| **Protocolo de túnel** | GRE (IP Protocol 47) | Modo de encapsulación de la interfaz Tunnel0 |
| **Tunnel source R1** | `Ethernet0/0` (192.168.1.10) | Interfaz física origen del túnel en Site A |
| **Tunnel source R2** | `Ethernet0/0` (192.168.1.20) | Interfaz física origen del túnel en Site B |
| **Tunnel destination R1** | 192.168.1.20 | IP WAN del peer remoto (R2) |
| **Tunnel destination R2** | 192.168.1.10 | IP WAN del peer remoto (R1) |
| **Subred del túnel** | 10.25.37.0/30 | Enlace lógico punto a punto entre R1 y R2 |
| **IP Tunnel0 R1** | 10.25.37.1/30 | Dirección lógica del extremo local (R1) |
| **IP Tunnel0 R2** | 10.25.37.2/30 | Dirección lógica del extremo remoto (R2) |
| **MTU del túnel** | 1476 bytes | MTU estándar (1500 - 24 bytes overhead GRE) |
| **Routing dinámico** | OSPF Área 0 | Propagación de rutas LAN entre sitios |
| **Cifrado** | Ninguno | GRE puro — sin IPSec |

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio simula la interconexión de dos sitios corporativos a través de Internet mediante un túnel GRE. El segmento `192.168.1.0/24` representa la nube pública/ISP. Las subredes LAN privadas están derivadas de la matrícula `20250737`. La interfaz `Tunnel0` de cada router establece el enlace virtual GRE punto a punto usando la subred `10.25.37.0/30`.

```
                              [ INTERNET / ISP ]
                              192.168.1.0/24
                              (Router ISP: 192.168.1.2)
                                    │
                   ┌────────────────┴────────────────┐
                   │ e0/0: 192.168.1.10              │ e0/0: 192.168.1.20
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Router R1    │◄══ TÚNEL GRE ══►│  Router R2    │
           │  (Site A)     │  IP Protocol 47 │  (Site B)     │
           │  Tunnel0:     │  Sin cifrado    │  Tunnel0:     │
           │  10.25.37.1   │                 │  10.25.37.2   │
           └───────┬───────┘                 └───────┬───────┘
                   │ e0/1: 20.25.37.129/25           │ e0/1: 20.25.37.1/25
                   │                                 │
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Switch SW1   │                 │  Switch SW2   │
           └───┬───────┬───┘                 └───┬───────┬───┘
               │       │                         │       │
           ┌───┴──┐ ┌──┴───┐               ┌────┴─┐ ┌───┴──┐
           │ PC1  │ │ PC2  │               │ PC3  │ │ PC4  │
           │.130  │ │.131  │               │ .2   │ │ .3   │
           └──────┘ └──────┘               └──────┘ └──────┘
            20.25.37.128/25                 20.25.37.0/25
               SITE A                          SITE B

  ════════════════════════════════════════════════════════════════
  Flujo del túnel GRE:
    1. PC1 (20.25.37.130) hace ping a PC3 (20.25.37.2).
    2. R1 consulta su tabla de enrutamiento: la ruta hacia
       20.25.37.0/25 fue aprendida por OSPF via Tunnel0.
    3. R1 encapsula el paquete original dentro de un nuevo
       header IP (src: 192.168.1.10 dst: 192.168.1.20)
       con Protocol 47 (GRE). El contenido NO se cifra.
    4. El paquete GRE viaja por Internet hasta R2.
    5. R2 identifica Protocol 47, desencapsula el header
       GRE y entrega el paquete original a PC3.
  ════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **ISP** | Router ISP | e0/0 | 192.168.1.2 | /24 | — | Enlace hacia R1 |
| | | e0/1 | 192.168.1.2 | /24 | — | Enlace hacia R2 |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.10 | /24 | 192.168.1.2 | Gateway WAN — Site A |
| | | e0/1 | 20.25.37.129 | /25 | — | Gateway LAN — Site A |
| | | Tunnel0 | 10.25.37.1 | /30 | — | Endpoint local del túnel GRE |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.20 | /24 | 192.168.1.2 | Gateway WAN — Site B |
| | | e0/1 | 20.25.37.1 | /25 | — | Gateway LAN — Site B |
| | | Tunnel0 | 10.25.37.2 | /30 | — | Endpoint remoto del túnel GRE |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site A |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site B |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.130 | /25 | 20.25.37.129 | Host Site A |
| **PC2** | Host Linux / VPC | eth0 | 20.25.37.131 | /25 | 20.25.37.129 | Host Site A |
| **PC3** | Host Linux / VPC | eth0 | 20.25.37.2 | /25 | 20.25.37.1 | Host Site B |
| **PC4** | Host Linux / VPC | eth0 | 20.25.37.3 | /25 | 20.25.37.1 | Host Site B |

---

## 4. Scripts de Configuración

### 4.1 Router R1 — Site A

```cisco
! ══════════════════════════════════════════════════════
! R1 — Site A | VPN Túnel GRE Point-to-Point
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.10 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 20.25.37.129 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.2

! ─── Interfaz de Túnel GRE ─────────────────────────────
! tunnel mode gre ip es el modo por defecto en Cisco IOS.
! No requiere ninguna configuración de crypto.
interface Tunnel0
 description GRE-P2P-hacia-R2
 ip address 10.25.37.1 255.255.255.252
 ip mtu 1476
 ip tcp adjust-mss 1436
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.20
 tunnel mode gre ip
 no shutdown

! ─── OSPF — Enrutamiento dinámico sobre el túnel GRE ───
! GRE soporta multicast nativo, por lo que OSPF funciona
! directamente sobre Tunnel0 sin configuración adicional.
router ospf 1
 network 20.25.37.128 0.0.0.127 area 0    ! LAN local Site A
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel GRE
```

### 4.2 Router R2 — Site B

```cisco
! ══════════════════════════════════════════════════════
! R2 — Site B | VPN Túnel GRE Point-to-Point
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.20 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 20.25.37.1 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.2

! ─── Interfaz de Túnel GRE ─────────────────────────────
interface Tunnel0
 description GRE-P2P-hacia-R1
 ip address 10.25.37.2 255.255.255.252
 ip mtu 1476
 ip tcp adjust-mss 1436
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.10
 tunnel mode gre ip
 no shutdown

! ─── OSPF — Enrutamiento dinámico sobre el túnel GRE ───
router ospf 1
 network 20.25.37.0 0.0.0.127 area 0      ! LAN local Site B
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel GRE
```

### 4.3 Configuración de PCs (Hosts)

**PC1 y PC2 — Site A (`20.25.37.128/25`)**

```bash
# PC1
ip 20.25.37.130 255.255.255.128 20.25.37.129

# PC2
ip 20.25.37.131 255.255.255.128 20.25.37.129
```

**PC3 y PC4 — Site B (`20.25.37.0/25`)**

```bash
# PC3
ip 20.25.37.2 255.255.255.128 20.25.37.1

# PC4
ip 20.25.37.3 255.255.255.128 20.25.37.1
```

> **Nota:** Si los hosts son VPCs de PNETLab se usa el comando `ip` como se muestra. Si son máquinas Linux se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación del Túnel

### 5.1 Verificar el estado de la interfaz Tunnel0

```cisco
R1# show interface Tunnel0
```

*Salida esperada:*

```
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Description: GRE-P2P-hacia-R2
  Internet address is 10.25.37.1/30
  MTU 1476 bytes, BW 100 Kbit/sec, DLY 50000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Tunnel source 192.168.1.10 (Ethernet0/0), destination 192.168.1.20
  Tunnel protocol/transport GRE/IP
  Key disabled, sequencing disabled
  Checksumming of packets disabled
```

> La línea `Tunnel protocol/transport GRE/IP` confirma que el modo es GRE puro. Nótese la ausencia de `Tunnel protection via IPSec` — eso diferencia este túnel del modelo *route-based* con IPSec del laboratorio anterior.

> Si el estado es `up/down`, generalmente indica que la ruta hacia el `tunnel destination` no existe en la tabla de enrutamiento. Verificar `ping 192.168.1.20` desde R1 antes de levantar el túnel.

---

### 5.2 Verificar la tabla de enrutamiento OSPF

```cisco
R1# show ip route ospf
```

*Salida esperada:*

```
      20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O        20.25.37.0/25 [110/1001] via 10.25.37.2, 00:01:15, Tunnel0
```

> La ruta hacia la LAN del Site B (`20.25.37.0/25`) fue aprendida dinámicamente por OSPF a través de `Tunnel0`. Esto confirma que GRE está transportando correctamente los paquetes multicast de OSPF (224.0.0.5).

---

### 5.3 Verificar la adyacencia OSPF sobre el túnel

```cisco
R1# show ip ospf neighbor
```

*Salida esperada:*

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.25.37.2        1   FULL/DR         00:00:35    10.25.37.2      Tunnel0
```

> El estado `FULL` indica que la adyacencia OSPF está completamente establecida sobre el túnel GRE. Sin GRE, esto sería imposible ya que OSPF requiere multicast (`224.0.0.5`) que IPSec puro no puede transportar.

---

### 5.4 Verificar conectividad extremo a extremo

Ejecutar desde **PC1** hacia **PC3**:

```bash
PC1> ping 20.25.37.2
```

*Resultado esperado:*

```
84 bytes from 20.25.37.2 icmp_seq=1 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=2 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=3 ttl=62 time=X ms
```

---

### 5.5 Comprobar que el tráfico viaja sin cifrar (Wireshark)

Para demostrar que GRE puro no cifra el tráfico, se puede capturar en la interfaz WAN de cualquiera de los routers:

```cisco
R1# debug ip packet detail
```

Al hacer ping desde PC1 a PC3, se verá en el debug que el paquete GRE viaja con el contenido ICMP completamente visible — a diferencia de IPSec donde el payload aparece como datos cifrados opacos.

Para detener el debug:

```cisco
R1# undebug all
```

---

### 5.6 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show interface Tunnel0` | Estado del túnel GRE: `up/up` indica túnel operativo. |
| `show ip route ospf` | Rutas OSPF aprendidas a través del túnel. |
| `show ip ospf neighbor` | Adyacencias OSPF — confirma que GRE transporta multicast. |
| `show ip route` | Tabla de enrutamiento completa — verifica rutas LAN remotas. |
| `show interfaces Tunnel0 counters` | Contadores de paquetes enviados y recibidos por el túnel. |
| `ping 192.168.1.20 source 192.168.1.10` | Verifica conectividad WAN entre los endpoints del túnel. |
| `traceroute 20.25.37.2 source 20.25.37.130` | Traza la ruta extremo a extremo a través del túnel. |

---

## 6. Capturas de Pantalla

A continuación se detalla el índice de evidencias correspondientes a las fases de configuración, verificación y funcionamiento del túnel GRE, alojadas en la carpeta [`screenshots/`](screenshots/README.md):

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_wan_previo.png`](screenshots/02_ping_wan_previo.png) | Ping desde R1 (`192.168.1.10`) hacia R2 (`192.168.1.20`) confirmando conectividad WAN base antes de levantar el túnel. |
| 3 | [`03_config_r1_tunnel.png`](screenshots/03_config_r1_tunnel.png) | Consola de R1 mostrando la configuración de la interfaz `Tunnel0` con `tunnel mode gre ip`, `tunnel source` y `tunnel destination`. |
| 4 | [`04_config_r1_ospf.png`](screenshots/04_config_r1_ospf.png) | Consola de R1 mostrando la configuración de OSPF con las redes LAN y la subred del túnel incluidas en el Área 0. |
| 5 | [`05_config_r2_completa.png`](screenshots/05_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la VTI GRE con `tunnel destination 192.168.1.10` y OSPF configurado. |
| 6 | [`06_tunnel0_up.png`](screenshots/06_tunnel0_up.png) | Salida de `show interface Tunnel0` en R1 mostrando estado `up/up` y `Tunnel protocol/transport GRE/IP`. |
| 7 | [`07_ospf_neighbor_full.png`](screenshots/07_ospf_neighbor_full.png) | Salida de `show ip ospf neighbor` mostrando la adyacencia en estado `FULL` sobre la interfaz `Tunnel0`. |
| 8 | [`08_ospf_route_aprendida.png`](screenshots/08_ospf_route_aprendida.png) | Salida de `show ip route ospf` en R1 mostrando la ruta `20.25.37.0/25` aprendida vía OSPF a través de `Tunnel0`. |
| 9 | [`09_ping_extremo_a_extremo.png`](screenshots/09_ping_extremo_a_extremo.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel GRE activo, TTL=62. |

---

## 7. Consideraciones de Seguridad

### 7.1 Limitación crítica: GRE no cifra

La limitación más importante de GRE puro es que **el tráfico viaja en texto claro** sobre la red pública. Cualquier dispositivo intermedio con capacidad de captura de paquetes (Wireshark, tcpdump) puede inspeccionar el contenido completo de las comunicaciones entre los dos sitios.

Esto significa que GRE puro **nunca debe utilizarse en producción sobre Internet** para transportar datos sensibles. Su uso legítimo en producción se limita a:

* Redes de backplane internas donde la capa física ya es segura.
* Como base para soluciones que agregan cifrado encima (GRE over IPSec, DMVPN).
* Interconexión de equipos dentro del mismo datacenter.

### 7.2 GRE over IPSec — el paso natural hacia producción

La solución para mantener las ventajas de GRE (multicast, routing dinámico) y agregar cifrado es **GRE over IPSec**: el túnel GRE se mantiene igual, y se agrega una Crypto Map o un IPSec Profile que cifra el tráfico GRE antes de salir por la interfaz WAN.

```cisco
! Agregar IPSec sobre el túnel GRE existente en R1:
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key JordyITLA2026! address 192.168.1.20

crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac
 mode transport                   ! Transport mode para GRE over IPSec

crypto ipsec profile GRE_IPSEC_PROFILE
 set transform-set TS_GRE

! Aplicar el perfil IPSec al túnel GRE existente:
interface Tunnel0
 tunnel protection ipsec profile GRE_IPSEC_PROFILE
```

> Con este agregado, el túnel GRE existente queda protegido por IPSec sin cambiar ninguna otra parte de la configuración.

### 7.3 Consideración sobre el MTU y la fragmentación

GRE agrega 24 bytes de overhead (20 IP + 4 GRE). Con un MTU de red estándar de 1500 bytes, el MTU efectivo del payload es **1476 bytes**. Si además se agrega IPSec, el overhead crece a ~73–89 bytes adicionales.

Por eso se configuran en la interfaz Tunnel0:

* `ip mtu 1476` — informa al router del MTU reducido del túnel.
* `ip tcp adjust-mss 1436` — ajusta el MSS de las sesiones TCP para evitar fragmentación en tránsito.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Demostración de los scripts de configuración aplicados en R1 y R2.
* ✅ Verificación de `show interface Tunnel0` mostrando `up/up` y `GRE/IP`.
* ✅ Verificación de `show ip ospf neighbor` mostrando adyacencia `FULL` sobre `Tunnel0`.
* ✅ Verificación de `show ip route ospf` mostrando la ruta remota aprendida dinámicamente.
* ✅ Ping exitoso extremo a extremo entre PC1 (`20.25.37.130`) y PC3/PC4 (`20.25.37.2/3`).

---

## 9. Referencias

* Hanks, S. et al. (1994). *RFC 1701 — Generic Routing Encapsulation (GRE)*. IETF.
* Farinacci, D. et al. (2000). *RFC 2784 — Generic Routing Encapsulation (GRE)*. IETF.
* Cisco Systems. (2024). *Cisco IOS IP Routing: Protocol-Independent Configuration Guide — Configuring GRE Tunnels*.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide: Secure Connectivity — GRE over IPSec*.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
