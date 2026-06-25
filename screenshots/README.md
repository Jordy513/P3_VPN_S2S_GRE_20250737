# Capturas de pantalla — IPSec VPN Basado en Enrutamiento (IKEv1)

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_sin_vpn.png`](/screenshots/02_ping_sin_vpn.png) | Ping desde PC1 (`20.25.37.130`) hacia PC3 (`20.25.37.2`) **antes** de aplicar la configuración IPSec — demuestra conectividad WAN base. |
| 3 | [`03_config_r1_vti.png`](/screenshots/03_config_r1_vti.png) | Consola de R1 mostrando la configuración de la interfaz `Tunnel0` con `tunnel mode ipsec ipv4` y `tunnel protection ipsec profile IPSEC_PROFILE_VTI`. |
| 4 | [`04_config_r1_isakmp_profile.png`](/screenshots/04_config_r1_isakmp_profile.png) | Consola de R1 mostrando la ISAKMP policy, la PSK y el IPSec Profile configurado (sin Crypto Map). |
| 5 | [`05_config_r2_completa.png`](/screenshots/05_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la VTI con `tunnel destination 192.168.1.10` y el mismo IPSec Profile. |
| 6 | [`06_tunnel0_up.png`](/screenshots/06_tunnel0_up.png) | Salida de `show interface Tunnel0` en R1 mostrando estado `up/up` y el protocolo de protección IPSec activo. |
| 7 | [`07_isakmp_sa_qmidle.png`](/screenshots/07_isakmp_sa_qmidle.png) | Salida de `show crypto isakmp sa` mostrando estado `QM_IDLE` — Fase 1 completada. |
| 8 | [`08_ospf_route_aprendida.png`](/screenshots/08_ospf_route_aprendida.png) | Salida de `show ip route ospf` en R1 mostrando la ruta `20.25.37.0/25` aprendida vía OSPF a través de `Tunnel0`. |
| 9 | [`09_ping_con_vpn_activa.png`](/screenshots/09_ping_con_vpn_activa.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel IPSec activo, TTL=62. |
