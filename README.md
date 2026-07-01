# 🔐 DMVPN Fase 3 — IPSec IKEv2 + EIGRP

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![DMVPN](https://img.shields.io/badge/DMVPN-Fase%203-green?style=for-the-badge)
![IKEv2](https://img.shields.io/badge/IPSec-IKEv2-purple?style=for-the-badge)
![EIGRP](https://img.shields.io/badge/Routing-EIGRP-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-red?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-gray?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y validación de una **VPN DMVPN Fase 3 con IPSec IKEv2 y enrutamiento dinámico EIGRP**. La red implementa un router HUB y dos routers SPOKE conectados mediante un ISP simulado. La comunicación entre las LANs se protege con IPSec/IKEv2 y se optimiza con `ip nhrp redirect` en el HUB e `ip nhrp shortcut` en los spokes, permitiendo comunicación spoke-to-spoke directa después de generarse el tráfico inicial.

> 💡 **Claves técnicas de Fase 3:**
> - `ip nhrp redirect` en el HUB — cuando detecta que debe reenviar tráfico por la misma interfaz Tunnel0, envía un **NHRP Traffic Indication** al spoke origen indicando la NBMA real del spoke destino.
> - `ip nhrp shortcut` en los spokes — al recibir el redirect, el spoke instala una ruta **shortcut /32** en su tabla de routing hacia el spoke destino, redirigiendo el tráfico de forma directa sin pasar por el HUB.
> - El HUB también mantiene `no ip split-horizon eigrp 25` para permitir la propagación correcta de rutas EIGRP entre spokes.
> - **IKEv2** usa la estructura `proposal → policy → keyring → profile` con clave wildcard `0.0.0.0 0.0.0.0` para permitir negociación dinámica entre cualquier par de peers.

---

## 🗺️ Topología de Red

Un HUB central con dos SPOKES y un ISP que simula la red pública. Cada sitio tiene una LAN con su VPC. El HUB también cuenta con su propia LAN (VPCH).

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| HUB | Ethernet0/0 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia HUB |
| ISP | Ethernet0/2 | 10.7.25.6/30 | Enlace hacia SPOKE1 |
| ISP | Ethernet0/1 | 10.7.25.10/30 | Enlace hacia SPOKE2 |
| SPOKE1 | Ethernet0/0 | 10.7.25.5/30 | WAN hacia ISP |
| SPOKE2 | Ethernet0/0 | 10.7.25.9/30 | WAN hacia ISP |
| HUB | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN HUB |
| VPCH | eth0 | 10.7.25.66/27 | Host LAN HUB |
| SPOKE1 | Ethernet0/1 | 10.7.25.97/27 | Gateway LAN SPOKE1 |
| VPC1 | eth0 | 10.7.25.98/27 | Host LAN SPOKE1 |
| SPOKE2 | Ethernet0/1 | 10.7.25.129/27 | Gateway LAN SPOKE2 |
| VPC2 | eth0 | 10.7.25.130/27 | Host LAN SPOKE2 |
| **HUB** | **Tunnel0** | **10.7.25.161/27** | **DMVPN NHS** |
| **SPOKE1** | **Tunnel0** | **10.7.25.162/27** | **DMVPN Spoke** |
| **SPOKE2** | **Tunnel0** | **10.7.25.163/27** | **DMVPN Spoke** |

### 📡 Segmentos

| Segmento | Red | Descripción |
|:--------:|:---:|-------------|
| HUB - ISP | 10.7.25.0/30 | Enlace WAN HUB |
| SPOKE1 - ISP | 10.7.25.4/30 | Enlace WAN SPOKE1 |
| SPOKE2 - ISP | 10.7.25.8/30 | Enlace WAN SPOKE2 |
| LAN HUB | 10.7.25.64/27 | Red local del HUB |
| LAN SPOKE1 | 10.7.25.96/27 | Red local de SPOKE1 |
| LAN SPOKE2 | 10.7.25.128/27 | Red local de SPOKE2 |
| Red DMVPN | 10.7.25.160/27 | Red multipoint del túnel |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | DMVPN Fase 3 Hub-and-Spoke |
| Tecnología de túnel | mGRE / NHRP |
| Peers | 1 HUB y 2 SPOKES |
| Protección | IPSec con IKEv2 |
| Cifrado IKEv2 | AES-CBC-256 |
| Integridad IKEv2 | SHA256 |
| Grupo Diffie-Hellman | Grupo 5 |
| Autenticación | Pre-Shared Key |
| Clave compartida | VPN12345 (wildcard 0.0.0.0 0.0.0.0) |
| Transform-set IPSec | TS-DMVPN-IKEV2: esp-aes 256 esp-sha256-hmac |
| Modo IPSec | Transport |
| NHRP Network ID | 725 |
| NHRP Authentication | DMVPN123 |
| Comando clave en HUB | `ip nhrp redirect` |
| Comando clave en SPOKES | `ip nhrp shortcut` |
| Enrutamiento dinámico | EIGRP AS 25 |

---

## 🔍 Funcionamiento de Fase 3

El mecanismo spoke-to-spoke en Fase 3 sigue estos pasos:

**1. Registro inicial** — Al arrancar, SPOKE1 y SPOKE2 envían un `NHRP Registration Request` al HUB (NHS), informando su IP de túnel y su dirección NBMA (WAN real).

**2. Tráfico inicial vía HUB** — El primer paquete de VPC1 hacia VPC2 viaja `VPC1 → SPOKE1 → HUB → SPOKE2 → VPC2`, porque EIGRP aprendió la LAN remota con next-hop = HUB.

**3. NHRP Redirect** — El HUB detecta que recibe y reenvía tráfico por la misma interfaz Tunnel0. Envía un `NHRP Traffic Indication` a SPOKE1 con la NBMA real de SPOKE2.

**4. NHRP Resolution** — SPOKE1 envía `NHRP Resolution Request` al HUB, que responde con la dirección NBMA de SPOKE2. SPOKE1 instala una ruta shortcut `/32` hacia la IP de túnel de SPOKE2.

**5. Tráfico directo** — A partir del segundo flujo, el tráfico va directamente `SPOKE1 → SPOKE2`, con IPSec IKEv2 protegiendo ese camino. El trace muestra el next-hop como la IP de túnel del spoke remoto.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
ip cef
interface Ethernet0/0
 description WAN_HACIA_HUB
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description WAN_HACIA_SPOKE2
 ip address 10.7.25.10 255.255.255.252
 no shutdown
interface Ethernet0/2
 description WAN_HACIA_SPOKE1
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### HUB — Configuración Principal
```cisco
hostname HUB
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_HUB
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

crypto ikev2 proposal IKEV2-DMVPN-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5
crypto ikev2 policy IKEV2-DMVPN-POLICY
 proposal IKEV2-DMVPN-PROP
crypto ikev2 keyring IKEV2-DMVPN-KEYRING
 peer DMVPN-PEERS
  address 0.0.0.0 0.0.0.0
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345
crypto ikev2 profile IKEV2-DMVPN-PROFILE
 match identity remote address 0.0.0.0
 identity local address 10.7.25.1
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-DMVPN-KEYRING

crypto ipsec transform-set TS-DMVPN-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN-IKEV2-PROFILE
 set transform-set TS-DMVPN-IKEV2
 set ikev2-profile IKEV2-DMVPN-PROFILE
 set pfs group5

interface Tunnel0
 description DMVPN_PHASE3_HUB_IKEV2
 ip address 10.7.25.161 255.255.255.224
 no ip redirects
 ip nhrp authentication DMVPN123
 ip nhrp map multicast dynamic
 ip nhrp network-id 725
 ip nhrp redirect
 no ip split-horizon eigrp 25
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 725
 tunnel protection ipsec profile DMVPN-IKEV2-PROFILE
 no shutdown

router eigrp 25
 no auto-summary
 passive-interface default
 no passive-interface Tunnel0
 network 10.7.25.64 0.0.0.31
 network 10.7.25.160 0.0.0.31
```

### SPOKE1 — Configuración Principal
```cisco
hostname SPOKE1
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.5 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_SPOKE1
 ip address 10.7.25.97 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.6

crypto ikev2 proposal IKEV2-DMVPN-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5
crypto ikev2 policy IKEV2-DMVPN-POLICY
 proposal IKEV2-DMVPN-PROP
crypto ikev2 keyring IKEV2-DMVPN-KEYRING
 peer DMVPN-PEERS
  address 0.0.0.0 0.0.0.0
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345
crypto ikev2 profile IKEV2-DMVPN-PROFILE
 match identity remote address 0.0.0.0
 identity local address 10.7.25.5
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-DMVPN-KEYRING

crypto ipsec transform-set TS-DMVPN-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN-IKEV2-PROFILE
 set transform-set TS-DMVPN-IKEV2
 set ikev2-profile IKEV2-DMVPN-PROFILE
 set pfs group5

interface Tunnel0
 description DMVPN_PHASE3_SPOKE1_IKEV2
 ip address 10.7.25.162 255.255.255.224
 no ip redirects
 ip nhrp authentication DMVPN123
 ip nhrp map 10.7.25.161 10.7.25.1
 ip nhrp map multicast 10.7.25.1
 ip nhrp nhs 10.7.25.161
 ip nhrp network-id 725
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 725
 tunnel protection ipsec profile DMVPN-IKEV2-PROFILE
 no shutdown

router eigrp 25
 no auto-summary
 passive-interface default
 no passive-interface Tunnel0
 network 10.7.25.96 0.0.0.31
 network 10.7.25.160 0.0.0.31
```

### SPOKE2 — Configuración Principal
```cisco
hostname SPOKE2
ip cef
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.9 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_SPOKE2
 ip address 10.7.25.129 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.10

crypto ikev2 proposal IKEV2-DMVPN-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5
crypto ikev2 policy IKEV2-DMVPN-POLICY
 proposal IKEV2-DMVPN-PROP
crypto ikev2 keyring IKEV2-DMVPN-KEYRING
 peer DMVPN-PEERS
  address 0.0.0.0 0.0.0.0
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345
crypto ikev2 profile IKEV2-DMVPN-PROFILE
 match identity remote address 0.0.0.0
 identity local address 10.7.25.9
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-DMVPN-KEYRING

crypto ipsec transform-set TS-DMVPN-IKEV2 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile DMVPN-IKEV2-PROFILE
 set transform-set TS-DMVPN-IKEV2
 set ikev2-profile IKEV2-DMVPN-PROFILE
 set pfs group5

interface Tunnel0
 description DMVPN_PHASE3_SPOKE2_IKEV2
 ip address 10.7.25.163 255.255.255.224
 no ip redirects
 ip nhrp authentication DMVPN123
 ip nhrp map 10.7.25.161 10.7.25.1
 ip nhrp map multicast 10.7.25.1
 ip nhrp nhs 10.7.25.161
 ip nhrp network-id 725
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 725
 tunnel protection ipsec profile DMVPN-IKEV2-PROFILE
 no shutdown

router eigrp 25
 no auto-summary
 passive-interface default
 no passive-interface Tunnel0
 network 10.7.25.128 0.0.0.31
 network 10.7.25.160 0.0.0.31
```

### Configuración de VPCs
```bash
# VPCH (LAN HUB)
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC1 (LAN SPOKE1)
ip 10.7.25.98 255.255.255.224 10.7.25.97

# VPC2 (LAN SPOKE2)
ip 10.7.25.130 255.255.255.224 10.7.25.129
```

---

## ✅ Verificación

```cisco
show ip interface brief
show running-config interface tunnel0
show running-config | section crypto
show running-config | section router eigrp
show dmvpn
show ip nhrp
show ip eigrp neighbors
show crypto ikev2 sa
show crypto ipsec sa
show crypto session
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show dmvpn` en HUB | 2 peers NHRP registrados (SPOKE1 y SPOKE2) |
| `show ip nhrp` en HUB | Entradas dinámicas de ambos spokes |
| `show ip nhrp` en SPOKE1 | Shortcut hacia red de SPOKE2 con NBMA 10.7.25.9 |
| `show ip nhrp` en SPOKE2 | Shortcut hacia red de SPOKE1 con NBMA 10.7.25.5 |
| `show ip eigrp neighbors` | Vecinos activos por Tunnel0 en HUB, SPOKE1, SPOKE2 |
| `show crypto ikev2 sa` | READY entre HUB↔SPOKE1, HUB↔SPOKE2 y SPOKE1↔SPOKE2 |
| `show crypto ipsec sa` | encaps/decaps activos |
| `show crypto session` | UP-ACTIVE hacia todos los peers |

> 💡 **Evidencia clave de Fase 3:** El trace VPC1 → VPC2 debe mostrar `10.7.25.97 → 10.7.25.163 → 10.7.25.130`, donde `10.7.25.163` es la IP de túnel de SPOKE2 (shortcut directo). El HUB no aparece como salto intermedio una vez instalado el shortcut.

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" al final es normal en VPCS — usa UDP y el destino responde que el puerto no está disponible. El ping exitoso confirma que el host fue alcanzado.

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Interfaces WAN, LAN y Tunnel0 en todos los routers | ✅ up/up |
| NHRP: SPOKE1 y SPOKE2 registrados en HUB | ✅ 2 peers en `show dmvpn` |
| NHRP shortcut en SPOKE1 hacia red de SPOKE2 | ✅ Confirmado |
| NHRP shortcut en SPOKE2 hacia red de SPOKE1 | ✅ Confirmado |
| Vecinos EIGRP por Tunnel0 | ✅ Activos en HUB, SPOKE1 y SPOKE2 |
| IKEv2 SA (HUB↔SPOKEs y SPOKE1↔SPOKE2) | ✅ READY |
| IPSec SA | ✅ encaps/decaps activos |
| Sesiones crypto | ✅ UP-ACTIVE |
| Ping / Trace VPC1 → VPC2 (spoke-to-spoke directo) | ✅ Exitoso |
| Ping / Trace VPC2 → VPC1 (spoke-to-spoke directo) | ✅ Exitoso |
| Prueba desde spoke hacia VPCH | ✅ Exitoso |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_DMVPN_Fase3_IKEV2_EIGRP.txt`](SaelGerman_2025-0725_Script_DMVPN_Fase3_IKEV2_EIGRP.txt) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_DMVPN-Fase3-IKEV2_P2.pdf`](SaelGerman_2025-0725_DMVPN-Fase3-IKEV2_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología DMVPN Fase 3 IKEv2 + EIGRP](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/01_topologia_dmvpn_fase3_ikev2_eigrp.png)
- 📸 [Figura 2 — Interfaces del ISP](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/02_isp_show_ip_interface_brief.png)
- 📸 [Figura 3 — Interfaces del HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/03_hub_show_ip_interface_brief.png)
- 📸 [Figura 4 — Configuración Tunnel0 en HUB (nhrp redirect)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/04_hub_show_running_config_interface_tunnel0.png)
- 📸 [Figura 5 — Parámetros crypto IKEv2/IPSec en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/05_hub_show_running_config_section_crypto.png)
- 📸 [Figura 6 — Configuración EIGRP AS 25 en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/06_hub_show_running_config_section_router_eigrp.png)
- 📸 [Figura 7 — Interfaces de SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/07_spoke1_show_ip_interface_brief.png)
- 📸 [Figura 8 — Configuración Tunnel0 en SPOKE1 (nhrp shortcut)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/08_spoke1_show_running_config_interface_tunnel0.png)
- 📸 [Figura 9 — Parámetros crypto IKEv2/IPSec en SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/09_spoke1_show_running_config_section_crypto.png)
- 📸 [Figura 10 — Configuración EIGRP AS 25 en SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/10_spoke1_show_running_config_section_router_eigrp.png)
- 📸 [Figura 11 — Interfaces de SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/11_spoke2_show_ip_interface_brief.png)
- 📸 [Figura 12 — Configuración Tunnel0 en SPOKE2 (nhrp shortcut)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/12_spoke2_show_running_config_interface_tunnel0.png)
- 📸 [Figura 13 — Parámetros crypto IKEv2/IPSec en SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/13_spoke2_show_running_config_section_crypto.png)
- 📸 [Figura 14 — Configuración EIGRP AS 25 en SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/14_spoke2_show_running_config_section_router_eigrp.png)
- 📸 [Figura 15 — show dmvpn en HUB (2 peers registrados)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/15_hub_show_dmvpn.png)
- 📸 [Figura 16 — Tabla NHRP en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/16_hub_show_ip_nhrp.png)
- 📸 [Figura 17 — Tabla NHRP en SPOKE1 (shortcut hacia SPOKE2)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/17_spoke1_show_ip_nhrp.png)
- 📸 [Figura 18 — Tabla NHRP en SPOKE2 (shortcut hacia SPOKE1)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/18_spoke2_show_ip_nhrp.png)
- 📸 [Figura 19 — Vecinos EIGRP en HUB](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/19_hub_show_ip_eigrp_neighbors.png)
- 📸 [Figura 20 — Vecinos EIGRP en SPOKE1](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/20_spoke1_show_ip_eigrp_neighbors.png)
- 📸 [Figura 21 — Vecinos EIGRP en SPOKE2](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/21_spoke2_show_ip_eigrp_neighbors.png)
- 📸 [Figura 22 — IKEv2 SA en HUB (READY)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/22_hub_show_crypto_ikev2_sa.png)
- 📸 [Figura 23 — IKEv2 SA en SPOKE1 (READY)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/23_spoke1_show_crypto_ikev2_sa.png)
- 📸 [Figura 24 — IKEv2 SA en SPOKE2 (READY)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/24_spoke2_show_crypto_ikev2_sa.png)
- 📸 [Figura 25 — IPSec SA en HUB (encaps/decaps)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/25_hub_show_crypto_ipsec_sa.png)
- 📸 [Figura 26 — IPSec SA en SPOKE1 (encaps/decaps)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/26_spoke1_show_crypto_ipsec_sa.png)
- 📸 [Figura 27 — IPSec SA en SPOKE2 (encaps/decaps)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/27_spoke2_show_crypto_ipsec_sa.png)
- 📸 [Figura 28 — Sesiones crypto activas en HUB (UP-ACTIVE)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/28_hub_show_crypto_session.png)
- 📸 [Figura 29 — Sesiones crypto activas en SPOKE1 (UP-ACTIVE)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/29_spoke1_show_crypto_session.png)
- 📸 [Figura 30 — Sesiones crypto activas en SPOKE2 (UP-ACTIVE)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/30_spoke2_show_crypto_session.png)
- 📸 [Figura 31 — Trace VPC1 → VPC2 (spoke-to-spoke directo)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/31_vpc1_show_ip_ping_trace_to_vpc2.png)
- 📸 [Figura 32 — Trace VPC2 → VPC1 (spoke-to-spoke directo)](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/32_vpc2_show_ip_ping_trace_to_vpc1.png)
- 📸 [Figura 33 — Prueba desde spoke hacia VPCH](SaelGerman_2025-0725_Capturas_DMVPN_Fase3_IKEV2_EIGRP_GitHub/33_prueba_vpc_hacia_vpch.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_DMVPN-Fase3-IKEV2_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/djtNHEr5R_k)

---

## 📚 Referencias

1. Cisco Systems. *Configuring DMVPN Phase 3 with IPSec IKEv2 and EIGRP*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
