# VPN Point-to-Point — Túnel GRE sobre IPSec (IKEv2)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Por qué GRE + IPSec?](#21-por-qué-gre--ipsec)
   - [Arquitectura GRE over IPSec con IKEv2](#22-arquitectura-gre-over-ipsec-con-ikev2)
   - [GRE over IPSec vs. VTI nativa (IPSec puro)](#23-gre-over-ipsec-vs-vti-nativa-ipsec-puro)
   - [Parámetros Criptográficos Utilizados](#24-parámetros-criptográficos-utilizados)
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

El objetivo de este laboratorio es **implementar y verificar una VPN Site-to-Site Point-to-Point utilizando un túnel GRE protegido por IPSec con IKEv2** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La combinación de GRE (que provee encapsulación multiprotocolo y soporte de multicast/broadcast) con IPSec negociado mediante IKEv2 (que provee cifrado, integridad y autenticación) — cerrando la brecha de seguridad que dejaba el túnel GRE puro del laboratorio anterior.
* La configuración de `tunnel protection ipsec profile` sobre una interfaz `Tunnel0` en modo `gre ip`, vinculando el perfil IPSec al `crypto ikev2 profile` mediante `set ikev2-profile`.
* La propagación dinámica de rutas mediante OSPF sobre el túnel GRE, demostrando que el cifrado IPSec no interfiere con el soporte multicast nativo de GRE.
* La comparación directa entre este modelo (GRE + IPSec) y la VTI nativa (`tunnel mode ipsec ipv4`) vista en el laboratorio de IKEv2 route-based, evidenciando cuándo conviene usar uno u otro.

---

## 2. Marco Teórico

### 2.1 ¿Por qué GRE + IPSec?

En el laboratorio de **Túnel GRE puro** se demostró que GRE encapsula tráfico multiprotocolo (incluyendo multicast, necesario para OSPF) pero **no cifra** — el contenido viaja en texto claro sobre Internet. En el laboratorio de **IPSec IKEv2 Route-Based (VTI)** se demostró que la VTI nativa cifra todo el tráfico, pero su soporte de multicast depende de la implementación interna de Cisco sobre `tunnel mode ipsec ipv4`.

GRE over IPSec resuelve esto combinando ambos protocolos explícitamente:

```
Paquete original (ICMP, OSPF Hello, protocolos no tan comunes, etc.)
        │
        ▼
   Encapsulación GRE   →  agrega header GRE (4 bytes) + header IP interno (20 bytes)
        │                  Soporta multicast/broadcast nativamente
        ▼
   Cifrado IPSec (ESP) →  cifra TODO el paquete GRE resultante
        │                  Modo transporte: solo cifra el payload GRE
        ▼
   Paquete final enviado por la WAN
```

El resultado es un túnel que **GRE hace transportable** (multicast, multiprotocolo) y que **IPSec hace seguro** (cifrado AES, integridad SHA, autenticación IKEv2).

### 2.2 Arquitectura GRE over IPSec con IKEv2

A diferencia de la VTI nativa (donde `tunnel mode ipsec ipv4` hace que la interfaz Tunnel0 sea IPSec puro), en GRE over IPSec la interfaz Tunnel0 sigue siendo **GRE** (`tunnel mode gre ip`, que es el modo por defecto) y se le agrega protección IPSec mediante `tunnel protection`:

```cisco
interface Tunnel0
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.20
 tunnel mode gre ip                          ! GRE — NO "ipsec ipv4"
 tunnel protection ipsec profile GRE_IPSEC_PROFILE_V2   ! IPSec protege el GRE
```

El IPSec Profile referenciado usa **modo transporte** (no túnel), ya que GRE ya provee su propia encapsulación — IPSec solo necesita cifrar el payload GRE, no volver a encapsularlo:

```cisco
crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport                              ! Transport, no tunnel
```

Esta es la diferencia de configuración más importante frente a la VTI nativa, donde el transform-set se configura en `mode tunnel`.

### 2.3 GRE over IPSec vs. VTI nativa (IPSec puro)

| Característica | GRE over IPSec (este lab) | VTI nativa — IPSec puro (lab anterior) |
|---|---|---|
| **Modo del túnel** | `tunnel mode gre ip` | `tunnel mode ipsec ipv4` |
| **Modo del transform-set** | `mode transport` | `mode tunnel` |
| **Overhead total** | ~24 bytes GRE + ~57-73 bytes IPSec transporte | ~50-90 bytes IPSec túnel |
| **Soporte multicast** | ✅ Nativo de GRE, garantizado | ✅ Soportado por Cisco IOS sobre VTI |
| **Multiprotocolo (no-IP)** | ✅ Puede transportar IPX, MPLS, etc. | ❌ Solo IPv4/IPv6 |
| **Compatibilidad histórica** | Alta — funciona desde IOS muy antiguos | Requiere IOS 12.3(14)T+ |
| **Caso de uso típico** | DMVPN, redes con protocolos legacy | VPNs site-to-site modernas simples |
| **Complejidad de config** | Ligeramente mayor (2 capas: GRE + IPSec) | Menor (1 capa: VTI) |

En la práctica, ambos modelos logran el mismo resultado funcional para este laboratorio (cifrado + OSPF dinámico). GRE over IPSec es históricamente el modelo más usado en producción porque es la base de DMVPN, mientras que la VTI nativa es más simple quando solo se necesita un túnel punto a punto.

### 2.4 Parámetros Criptográficos Utilizados

| Parámetro | Valor configurado | Propósito |
|---|---|---|
| **Cifrado IKE SA** | AES-256-CBC | Cifrado del canal IKE |
| **Integridad IKE SA** | SHA-256 | Integridad de mensajes IKE |
| **Grupo Diffie-Hellman** | Grupo 14 (2048-bit MODP) | Intercambio seguro de material de clave |
| **Autenticación** | Pre-Shared Key (PSK) | Autenticación mutua de los peers |
| **Lifetime IKE SA** | 86400 segundos (24 horas) | Duración del canal IKE antes de renegociar |
| **Cifrado ESP (Child SA)** | AES-256 | Cifrado del tráfico GRE encapsulado |
| **Autenticación ESP (Child SA)** | SHA-256 HMAC | Integridad del tráfico GRE encapsulado |
| **Modo IPSec** | **Transport** (no Tunnel) | GRE ya encapsula — IPSec solo cifra el payload |
| **Modo del túnel GRE** | `gre ip` (por defecto) | Encapsulación multiprotocolo con soporte multicast |
| **Subnet del túnel GRE** | 10.25.37.0/30 | Enlace lógico punto a punto entre R1 y R2 |
| **Enrutamiento sobre el túnel** | OSPF (Área 0) | Propagación dinámica de rutas LAN entre sitios |

> **Nota sobre PRF:** En IOSv/IOS 15.x el comando `prf` dentro de `crypto ikev2 proposal` no está disponible. IOS deriva automáticamente la PRF del algoritmo de integridad configurado.

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio simula la interconexión segura de dos sitios corporativos a través de Internet. El segmento `192.168.1.0/24` representa la nube pública/ISP. Las subredes LAN privadas están derivadas de la matrícula `20250737`. La interfaz `Tunnel0` en cada router opera en modo GRE, con IKEv2/IPSec protegiendo el tráfico mediante `tunnel protection`.

```
                              [ INTERNET / ISP ]
                              192.168.1.0/24
                              (Router ISP: 192.168.1.2)
                                    │
                   ┌────────────────┴────────────────┐
                   │ e0/0: 192.168.1.10              │ e0/0: 192.168.1.20
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Router R1    │◄═══ TÚNEL ══════►  Router R2    │
           │  (Site A)     │  GRE + IPSec    │  (Site B)     │
           │  Tunnel0:     │  IKEv2          │  Tunnel0:     │
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
  Flujo del túnel GRE over IPSec (IKEv2):
    1. PC1 (20.25.37.130) hace ping a PC3 (20.25.37.2).
    2. R1 consulta su tabla de rutas: 20.25.37.0/25 fue
       aprendida por OSPF vía Tunnel0.
    3. El paquete entra a Tunnel0 → se encapsula en GRE
       (header GRE + header IP interno, soporta multicast).
    4. tunnel protection cifra el paquete GRE resultante
       con IPSec en modo transporte (IKEv2 ya negoció la SA).
    5. El paquete viaja cifrado: 192.168.1.10 → 192.168.1.20.
    6. R2 descifra (IPSec) y desencapsula (GRE) → entrega a PC3.
  ════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **ISP** | Router ISP | e0/0 | 192.168.1.2 | /24 | — | Enlace hacia R1 |
| | | e0/1 | 192.168.1.2 | /24 | — | Enlace hacia R2 |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.10 | /24 | 192.168.1.2 | Gateway WAN — Site A |
| | | e0/1 | 20.25.37.129 | /25 | — | Gateway LAN — Site A |
| | | Tunnel0 | 10.25.37.1 | /30 | — | Endpoint local — GRE over IPSec |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.20 | /24 | 192.168.1.2 | Gateway WAN — Site B |
| | | e0/1 | 20.25.37.1 | /25 | — | Gateway LAN — Site B |
| | | Tunnel0 | 10.25.37.2 | /30 | — | Endpoint remoto — GRE over IPSec |
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
! R1 — Site A | GRE over IPSec con IKEv2
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

! ══════════════════════════════════════════════════════
! BLOQUE IKEv2
! ══════════════════════════════════════════════════════

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
! NOTA: no incluir "prf sha256" — no soportado en IOSv.
crypto ikev2 proposal PROP_IKEv2_GRE
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
crypto ikev2 policy POL_IKEv2_GRE
 proposal PROP_IKEv2_GRE

! ─── PASO 3: IKEv2 Keyring ─────────────────────────────
crypto ikev2 keyring KR_SITEB_GRE
 peer R2_SITEB
  address 192.168.1.20
  pre-shared-key local  JordyITLA2026!
  pre-shared-key remote JordyITLA2026!

! ─── PASO 4: IKEv2 Profile ─────────────────────────────
crypto ikev2 profile PROF_IKEv2_GRE
 match identity remote address 192.168.1.20 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_SITEB_GRE

! ══════════════════════════════════════════════════════
! BLOQUE IPSEC — protege el GRE en modo TRANSPORTE
! ══════════════════════════════════════════════════════

! ─── PASO 5: Transform Set en modo TRANSPORTE ──────────
! Diferencia clave vs VTI nativa: aquí es "transport",
! no "tunnel", porque GRE ya hace su propia encapsulación.
crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 6: IPSec Profile ─────────────────────────────
crypto ipsec profile GRE_IPSEC_PROFILE_V2
 set transform-set TS_GRE_V2
 set ikev2-profile PROF_IKEv2_GRE
 set security-association lifetime seconds 3600

! ─── PASO 7: Interfaz de Túnel GRE protegida por IPSec ─
! tunnel mode permanece GRE (no "ipsec ipv4").
! tunnel protection agrega el cifrado IKEv2/IPSec encima.
interface Tunnel0
 description GRE-over-IPSec-IKEv2-hacia-R2
 ip address 10.25.37.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.20
 tunnel mode gre ip
 tunnel protection ipsec profile GRE_IPSEC_PROFILE_V2
 no shutdown

! ─── PASO 8: OSPF sobre el túnel GRE ───────────────────
! GRE transporta el multicast de OSPF de forma nativa,
! incluso con IPSec protegiendo el túnel.
router ospf 1
 network 20.25.37.128 0.0.0.127 area 0    ! LAN local Site A
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel GRE
```

### 4.2 Router R2 — Site B

```cisco
! ══════════════════════════════════════════════════════
! R2 — Site B | GRE over IPSec con IKEv2
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

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
crypto ikev2 proposal PROP_IKEv2_GRE
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
crypto ikev2 policy POL_IKEv2_GRE
 proposal PROP_IKEv2_GRE

! ─── PASO 3: IKEv2 Keyring ─────────────────────────────
crypto ikev2 keyring KR_SITEA_GRE
 peer R1_SITEA
  address 192.168.1.10
  pre-shared-key local  JordyITLA2026!
  pre-shared-key remote JordyITLA2026!

! ─── PASO 4: IKEv2 Profile ─────────────────────────────
crypto ikev2 profile PROF_IKEv2_GRE
 match identity remote address 192.168.1.10 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_SITEA_GRE

! ─── PASO 5: Transform Set en modo TRANSPORTE ──────────
crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 6: IPSec Profile ─────────────────────────────
crypto ipsec profile GRE_IPSEC_PROFILE_V2
 set transform-set TS_GRE_V2
 set ikev2-profile PROF_IKEv2_GRE
 set security-association lifetime seconds 3600

! ─── PASO 7: Interfaz de Túnel GRE protegida por IPSec ─
interface Tunnel0
 description GRE-over-IPSec-IKEv2-hacia-R1
 ip address 10.25.37.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.10
 tunnel mode gre ip
 tunnel protection ipsec profile GRE_IPSEC_PROFILE_V2
 no shutdown

! ─── PASO 8: OSPF sobre el túnel GRE ───────────────────
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

### 5.1 Disparar la negociación IKEv2 (paso obligatorio)

> ⚠️ **IKEv2 es lazy por defecto** — al igual que en los laboratorios IKEv2 anteriores, el túnel no se negocia al aplicar la configuración. La SA se crea cuando hay tráfico que atraviesa la interfaz `Tunnel0`.

```cisco
R1# ping 20.25.37.2 source 20.25.37.129 repeat 5
```

> Si el `tunnel protection` aún no terminó de aplicarse y el ping falla en los primeros intentos, repetir el comando — la negociación IKEv2 toma unos segundos en completarse.

---

### 5.2 Verificar el estado de la interfaz Tunnel0

```cisco
R1# show interface Tunnel0
```

*Salida esperada:*

```
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Description: GRE-over-IPSec-IKEv2-hacia-R2
  Internet address is 10.25.37.1/30
  MTU 1400 bytes, BW 100 Kbit/sec, DLY 50000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Tunnel source 192.168.1.10 (Ethernet0/0), destination 192.168.1.20
  Tunnel protocol/transport GRE/IP
  Tunnel protection via IPSec (profile "GRE_IPSEC_PROFILE_V2")
```

> Nótese `Tunnel protocol/transport GRE/IP` — confirma que el modo de la interfaz sigue siendo GRE (no IPSec puro). La línea `Tunnel protection via IPSec` confirma que el cifrado IKEv2/IPSec está aplicado encima del GRE. Esta combinación de ambas líneas es la firma distintiva de GRE over IPSec frente a la VTI nativa del laboratorio anterior.

---

### 5.3 Verificar el estado de la IKEv2 SA

```cisco
R1# show crypto ikev2 sa
```

*Salida esperada:*

```
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         192.168.1.10/500      192.168.1.20/500      none/none            READY
      Encr: AES-CBC, keysize: 256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/XX sec
```

> Estado `READY` confirma la IKE SA. Como en los labs anteriores, la sección Child SA no aparece en este output en IOSv — sus detalles se verifican con `show crypto ipsec sa`.

---

### 5.4 Verificar las IPSec SAs (modo transporte)

```cisco
R1# show crypto ipsec sa
```

*Salida esperada (fragmento):*

```
interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 192.168.1.10

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (192.168.1.10/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (192.168.1.20/255.255.255.255/47/0)
   current_peer 192.168.1.20 port 500

    #pkts encaps: 12, #pkts encrypt: 12, #pkts digest: 12
    #pkts decaps: 12, #pkts decrypt: 12, #pkts verify: 12
```

> **Diferencia clave vs. la VTI nativa:** los identifiers muestran `192.168.1.10/.../47/0` — el protocolo `47` es GRE. Esto confirma que IPSec está protegiendo específicamente el tráfico GRE entre las IPs WAN de los routers, no las subredes LAN directamente (eso lo resuelve GRE por dentro).

---

### 5.5 Verificar la tabla de enrutamiento OSPF

```cisco
R1# show ip route ospf
```

*Salida esperada:*

```
      20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O        20.25.37.0/25 [110/1001] via 10.25.37.2, 00:01:10, Tunnel0
```

---

### 5.6 Verificar adyacencia OSPF sobre el túnel

```cisco
R1# show ip ospf neighbor
```

*Salida esperada:*

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.25.37.2        1   FULL/DR         00:00:35    10.25.37.2      Tunnel0
```

> Confirma que el multicast OSPF (224.0.0.5) atraviesa correctamente el túnel GRE incluso estando cifrado por IPSec — la prueba definitiva de que GRE over IPSec combina ambas ventajas.

---

### 5.7 Verificar conectividad extremo a extremo

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

### 5.8 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show interface Tunnel0` | Confirma `GRE/IP` + `Tunnel protection via IPSec`. |
| `show crypto ikev2 sa` | Estado IKEv2 SA. Debe ser `READY`. |
| `show crypto ikev2 sa detail` | Parámetros negociados, SPIs IKE, identidades. |
| `show crypto ipsec sa` | SAs en modo transporte, protocolo 47 (GRE) en los identifiers. |
| `show ip route ospf` | Rutas aprendidas dinámicamente a través del túnel. |
| `show ip ospf neighbor` | Adyacencia OSPF sobre `Tunnel0` en estado `FULL`. |
| `show crypto ikev2 session` | Resumen de la sesión IKEv2 activa. |
| `debug crypto ikev2` | Debug de la negociación IKEv2 en tiempo real. |

---

## 6. Capturas de Pantalla

A continuación se detalla el índice de evidencias correspondientes a las fases de configuración, verificación y funcionamiento del túnel GRE over IPSec, alojadas en la carpeta [`screenshots/`](screenshots/README.md):

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_wan_previo.png`](screenshots/02_ping_wan_previo.png) | Ping desde R1 (`192.168.1.10`) hacia R2 (`192.168.1.20`) confirmando conectividad WAN base antes de configurar el túnel. |
| 3 | [`03_config_r1_ikev2.png`](screenshots/03_config_r1_ikev2.png) | Consola de R1 mostrando `crypto ikev2 proposal`, `crypto ikev2 policy` y `crypto ikev2 keyring`. |
| 4 | [`04_config_r1_transport_mode.png`](screenshots/04_config_r1_transport_mode.png) | Consola de R1 mostrando el `crypto ipsec transform-set` con `mode transport` — diferencia clave vs. la VTI nativa. |
| 5 | [`05_config_r1_tunnel_gre.png`](screenshots/05_config_r1_tunnel_gre.png) | Consola de R1 mostrando la interfaz `Tunnel0` con `tunnel mode gre ip` y `tunnel protection ipsec profile`. |
| 6 | [`06_config_r2_completa.png`](screenshots/06_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la simetría con R1. |
| 7 | [`07_tunnel0_gre_ip_protected.png`](screenshots/07_tunnel0_gre_ip_protected.png) | Salida de `show interface Tunnel0` mostrando `Tunnel protocol/transport GRE/IP` junto con `Tunnel protection via IPSec`. |
| 8 | [`08_ikev2_sa_ready.png`](screenshots/08_ikev2_sa_ready.png) | Salida de `show crypto ikev2 sa` mostrando estado `READY` tras disparar la negociación con ping extendido. |
| 9 | [`09_ipsec_sa_protocolo_47.png`](screenshots/09_ipsec_sa_protocolo_47.png) | Salida de `show crypto ipsec sa` mostrando los identifiers con protocolo `47` (GRE) — evidencia de que IPSec protege específicamente el tráfico GRE. |
| 10 | [`10_ospf_neighbor_full.png`](screenshots/10_ospf_neighbor_full.png) | Salida de `show ip ospf neighbor` mostrando adyacencia `FULL` sobre `Tunnel0` con IPSec activo. |
| 11 | [`11_ping_extremo_a_extremo.png`](screenshots/11_ping_extremo_a_extremo.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel GRE over IPSec activo, TTL=62. |

---

## 7. Consideraciones de Seguridad

### 7.1 Ventajas de seguridad de GRE over IPSec

* **Cifrado completo del payload GRE:** A diferencia del laboratorio de GRE puro, todo el contenido — incluyendo los Hello de OSPF y el tráfico de datos de los hosts — viaja cifrado con AES-256.
* **Autenticación robusta vía IKEv2:** La negociación de claves hereda todas las ventajas de IKEv2 (DPD integrado, resistencia a DoS, sin Aggressive Mode inseguro).
* **Separación de responsabilidades:** GRE maneja el enrutamiento y el transporte multiprotocolo; IPSec maneja exclusivamente la seguridad — una arquitectura modular más fácil de auditar por capas.

### 7.2 Hardening recomendado

```cisco
! Eliminar la policy ISAKMP legacy por defecto de Cisco
no crypto isakmp policy 65535

! Deshabilitar IKEv1 si solo se usa IKEv2
no crypto isakmp enable

! Forzar Perfect Forward Secrecy en el IPSec Profile
crypto ipsec profile GRE_IPSEC_PROFILE_V2
 set pfs group14

! Restringir tráfico WAN solo al peer conocido:
! IKE (UDP 500/4500) y el propio GRE (protocolo 47) ya cifrado
ip access-list extended ACL_WAN_GRE_IPSEC
 permit udp host 192.168.1.20 host 192.168.1.10 eq 500
 permit udp host 192.168.1.20 host 192.168.1.10 eq 4500
 permit esp host 192.168.1.20 host 192.168.1.10
 permit gre host 192.168.1.20 host 192.168.1.10
 deny ip any any log

interface Ethernet0/0
 ip access-group ACL_WAN_GRE_IPSEC in
```

> **Nota:** Aunque IPSec ya cifra el contenido del GRE, el paquete ESP resultante sigue siendo visible como tráfico UDP 500/4500 y ESP en la WAN — el ACL anterior limita qué puede llegar a esos puertos, no reemplaza el cifrado.

### 7.3 Overhead acumulado — cálculo de MTU

A diferencia de la VTI nativa (un solo nivel de overhead IPSec), GRE over IPSec acumula **dos capas**:

| Capa | Overhead aproximado |
|---|---|
| GRE | 24 bytes (20 IP interno + 4 GRE) |
| IPSec ESP modo transporte (AES-256 + SHA-256) | ~37–53 bytes |
| **Total estimado** | **~61–77 bytes** |

Por eso en este laboratorio se mantiene `ip mtu 1400` (más conservador que los 1476 del GRE puro) y `ip tcp adjust-mss 1360`, dejando margen suficiente para evitar fragmentación incluso en el peor caso de overhead.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Demostración de los scripts de configuración aplicados en R1 y R2.
* ✅ Ping extendido para disparar la negociación IKEv2 (`ping 20.25.37.2 source 20.25.37.129`).
* ✅ Verificación de `show interface Tunnel0` mostrando `GRE/IP` + `Tunnel protection via IPSec`.
* ✅ Verificación de `show crypto ikev2 sa` mostrando estado `READY`.
* ✅ Verificación de `show crypto ipsec sa` mostrando protocolo `47` en los identifiers.
* ✅ Verificación de `show ip ospf neighbor` mostrando adyacencia `FULL` sobre `Tunnel0`.
* ✅ Ping exitoso extremo a extremo entre PC1 (`20.25.37.130`) y PC3/PC4 (`20.25.37.2/3`).

---

## 9. Referencias

* Hanks, S. et al. (1994). *RFC 1701 — Generic Routing Encapsulation (GRE)*. IETF.
* Farinacci, D. et al. (2000). *RFC 2784 — Generic Routing Encapsulation (GRE)*. IETF.
* Kaufman, C. et al. (2014). *RFC 7296 — Internet Key Exchange Protocol Version 2 (IKEv2)*. IETF.
* Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide: Secure Connectivity — GRE over IPSec*.
* Cisco Systems. (2024). *Cisco IOS Security Command Reference — tunnel protection ipsec profile*.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
