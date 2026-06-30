# Capturas de pantalla — IPSec VPN Túnel GRE (Generic Routing Encapsulation) (IKEv2)

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_wan_previo.png`](/screenshots/02_ping_wan_previo.png) | Ping desde R1 (`192.168.1.10`) hacia R2 (`192.168.1.20`) confirmando conectividad WAN base antes de configurar el túnel. |
| 3 | [`03_config_r1_ikev2.png`](/screenshots/03_config_r1_ikev2.png) | Consola de R1 mostrando `crypto ikev2 proposal`, `crypto ikev2 policy` y `crypto ikev2 keyring`. |
| 4 | [`04_config_r1_transport_mode.png`](/screenshots/04_config_r1_transport_mode.png) | Consola de R1 mostrando el `crypto ipsec transform-set` con `mode transport` — diferencia clave vs. la VTI nativa. |
| 5 | [`05_config_r1_tunnel_gre.png`](/screenshots/05_config_r1_tunnel_gre.png) | Consola de R1 mostrando la interfaz `Tunnel0` con `tunnel mode gre ip` y `tunnel protection ipsec profile`. |
| 6 | [`06_config_r2_completa.png`](/screenshots/06_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la simetría con R1. |
| 7 | [`07_tunnel0_gre_ip_protected.png`](/screenshots/07_tunnel0_gre_ip_protected.png) | Salida de `show interface Tunnel0` mostrando `Tunnel protocol/transport GRE/IP` junto con `Tunnel protection via IPSec`. |
| 8 | [`08_ikev2_sa_ready.png`](/screenshots/08_ikev2_sa_ready.png) | Salida de `show crypto ikev2 sa` mostrando estado `READY` tras disparar la negociación con ping extendido. |
| 9 | [`09_ipsec_sa_protocolo_47.png`](/screenshots/09_ipsec_sa_protocolo_47.png) | Salida de `show crypto ipsec sa` mostrando los identifiers con protocolo `47` (GRE) — evidencia de que IPSec protege específicamente el tráfico GRE. |
| 10 | [`10_ospf_neighbor_full.png`](/screenshots/10_ospf_neighbor_full.png) | Salida de `show ip ospf neighbor` mostrando adyacencia `FULL` sobre `Tunnel0` con IPSec activo. |
| 11 | [`11_ping_extremo_a_extremo.png`](/screenshots/11_ping_extremo_a_extremo.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel GRE over IPSec activo, TTL=62. |
