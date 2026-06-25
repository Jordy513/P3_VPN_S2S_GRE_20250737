# Capturas de pantalla — IPSec VPN Túnel GRE (Generic Routing Encapsulation) (IKEv1)

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_wan_previo.png`](/screenshots/02_ping_wan_previo.png) | Ping desde R1 (`192.168.1.10`) hacia R2 (`192.168.1.20`) confirmando conectividad WAN base antes de levantar el túnel. |
| 3 | [`03_config_r1_tunnel.png`](/screenshots/03_config_r1_tunnel.png) | Consola de R1 mostrando la configuración de la interfaz `Tunnel0` con `tunnel mode gre ip`, `tunnel source` y `tunnel destination`. |
| 4 | [`04_config_r1_ospf.png`](/screenshots/04_config_r1_ospf.png) | Consola de R1 mostrando la configuración de OSPF con las redes LAN y la subred del túnel incluidas en el Área 0. |
| 5 | [`05_config_r2_completa.png`](/screenshots/05_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la VTI GRE con `tunnel destination 192.168.1.10` y OSPF configurado. |
| 6 | [`06_tunnel0_up.png`](/screenshots/06_tunnel0_up.png) | Salida de `show interface Tunnel0` en R1 mostrando estado `up/up` y `Tunnel protocol/transport GRE/IP`. |
| 7 | [`07_ospf_neighbor_full.png`](/screenshots/07_ospf_neighbor_full.png) | Salida de `show ip ospf neighbor` mostrando la adyacencia en estado `FULL` sobre la interfaz `Tunnel0`. |
| 8 | [`08_ospf_route_aprendida.png`](/screenshots/08_ospf_route_aprendida.png) | Salida de `show ip route ospf` en R1 mostrando la ruta `20.25.37.0/25` aprendida vía OSPF a través de `Tunnel0`. |
| 9 | [`09_ping_extremo_a_extremo.png`](/screenshots/09_ping_extremo_a_extremo.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel GRE activo, TTL=62. |
