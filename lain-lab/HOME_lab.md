# Homelab

Laboratorio doméstico de redes y seguridad: firewall OPNsense en bare metal, virtualización con Proxmox y servicios autogestionados.

*Última actualización: noviembre 2026.*

## Por qué existe

Soy estudiante de tercer semestre y me gustan las redes, incluso cuando no cooperan: doble NAT, un proveedor que no deja abrir puertos, un firewall que llega a la LAN por un adaptador USB. Armé este lab para estudiar el CCNA con tráfico real y para operar un firewall como se hace en una empresa, en lugar de quedarme en la teoría.

Nextcloud y un servidor de Minecraft que usan otras personas me obligan a mantenerlo estable. El acceso remoto va solo por VPN, y la salida a internet del lab pasa por un túnel de Mullvad.


## Inventario

| Equipo | Rol | Interfaz / IP | Servicios | Limitaciones |
|---|---|---|---|---|
| Router Megacable | Acceso a internet | Hacia la WAN (`re0`) | Ninguno | El proveedor no permite abrir puertos; sin respaldo de batería |
| Laptop con OPNsense | Firewall y router | WAN `re0`, LAN `ue0` (USB-Ethernet), 192.168.1.1/24 | AdGuard Home, DNS, DHCP, IDS, WireGuard (Mullvad) | LAN por USB (punto único de falla); se calienta |
| Switch | Conecta la LAN | Sin IP (no administrable) | Ninguno | Sin VLANs |
| NAS (Proxmox) | NAS y virtualización | 192.168.1.2/24 | NAS, Minecraft, Nextcloud, Debian (pruebas) | CPU de pocos hilos |
| PC principal | Estación de trabajo y lab ofensivo | 192.168.1.102/24 | VMs Kali y Ubuntu (MVP de LUMI), IA local | Es el equipo donde descargo y pruebo software |
| Laptop personal | Uso personal | 192.168.1.103/24 | VM Kali | Sin respaldo de batería |
| AP Wi-Fi | Acceso inalámbrico | 192.168.10.1/24 | ESP32 (métricas del NAS o del firewall) y celulares | Solo 2.4 GHz; aislamiento hacia la LAN pendiente |

## Componentes

### Router Megacable

Es el módem/router del proveedor y su único uso es darle salida a internet a la WAN de OPNsense. Como no permite abrir puertos, el acceso remoto se resuelve con Tailscale.

Su limitación es la energía: sin respaldo de batería, una falla de luz deja al lab sin internet.



### Laptop con OPNsense (firewall y router)

Es el firewall y router de todo el lab. Tiene reglas de salida que restringen los puertos hacia internet, bloqueo de dominios por DNS con AdGuard Home y registro de conexiones. Servicios activos: AdGuard Home, DHCP Relay, Intrusion Detection, Unbound DNS, NTP, Kea DHCP, Router Advertisements, Monit y el cliente WireGuard de Mullvad.

Consideré nftables, que enseña más sobre reglas porque se configura casi desde cero. Me quedé con OPNsense porque se parece más a lo que se usa en empresas y me permite ver qué entra y qué sale hacia el NAS.

La salida a internet pasa por un túnel WireGuard hacia Mullvad, configurado directamente en OPNsense. Una regla manda el tráfico de la LAN por el túnel y, por encima de ella, una regla flotante deja dentro de la LAN el tráfico interno, incluido el DNS hacia AdGuard Home. Sin esa excepción, forzar todo por el túnel impide llegar a AdGuard. Configuré un kill switch para que, si el túnel cae, el lab se quede sin conexión hasta que se restablezca. Prefiero perder internet a que el tráfico salga directo por Megacable.

Tiene dos puntos débiles. La laptop trae un solo puerto Ethernet Gb integrado, así que la LAN sale por un adaptador USB Gb; si falla el router, el adaptador o su driver, cae todo el lab. Además se calienta: si se sobrecalienta se cuelga un momento y tumba la LAN, por eso debe mantenerse ventilada.

### Switch

Conecta por cable todo el lab. No es administrable y fue la mejor opción por precio con esas características.

Sin VLANs, todos los equipos comparten el mismo dominio de red y se ven entre sí, y ese tráfico no pasa por OPNsense. La protección depende de reglas y endurecimiento en cada equipo.

No tiene interfaz de administración, así que no hay captura que mostrar.

### NAS (Proxmox)

Es el servidor del lab, un Beelink con Proxmox. Aloja el NAS, el servidor de Minecraft, Nextcloud (acceso remoto solo por Tailscale) y una VM Debian para pruebas de pentesting o defensa.

Elegí Proxmox por su escalabilidad y su gestión, y porque es de lo más cercano a lo que se usa en la industria. Me da práctica en máquinas virtuales, redes y servicios en la nube propios.

Limitaciones:
- La CPU es un Pentium de 13.ª generación con pocos hilos. Asigno recursos justos y me apoyo en la RAM.
- Al arrancar fallaba el entorno gráfico por la iGPU. Lo resolví agregando `nomodeset` al final de la línea de arranque de Linux, y ya está aplicado.
- Los servicios del NAS todavía no están segmentados entre sí.


### PC principal

Es mi estación de trabajo y mi laboratorio ofensivo: programación, IA local (16 GB de VRAM, limitada por ser una iGPU) y VMs Kali y Ubuntu para el MVP de LUMI. También la uso en este lab, en HackTheBox y en TryHackMe. Concentré lo ofensivo en un solo equipo para no mezclarlo con los servicios.

Es mi equipo más fuerte y también el más expuesto, porque en él descargo y ejecuto software de terceros (riesgo de cadena de suministro, MITRE T1195).

### Laptop personal

Equipo portátil para programar y usar Kali desde cualquier lugar. Lo elegí por portabilidad y almacenamiento. Tiene poca batería y el mismo riesgo que la PC principal con el software de terceros.

### AP Wi-Fi

Da acceso inalámbrico y filtra por DNS, porque pasa por AdGuard. Conecta un ESP32 que muestra métricas del NAS o del firewall, y los celulares.

Limitaciones:
- Solo 2.4 GHz.
- Sin respaldo de batería.
- El aislamiento hacia la LAN está pendiente. Alguien con acceso al Wi-Fi, o a través de un celular, un ESP32 o una futura cámara, podría llegar al resto del lab, porque no hay segmentación.



## Decisiones de diseño

Cada una cede algo a cambio de otra cosa.

- **OPNsense en bare metal, no nftables.** Cedí aprender reglas desde cero y gané práctica con un firewall como los de empresa.
- **Proxmox.** Escalabilidad, gestión y cercanía con lo que se usa en la industria.
- **Tailscale en vez de abrir puertos.** El proveedor no los deja abrir, y así los servicios tampoco quedan expuestos a internet.
- **Mullvad para la salida, con kill switch.** Tailscale cubre el acceso remoto entrante y Mullvad el tráfico que sale. Cedí disponibilidad (si el túnel cae, no hay internet hasta que se restablece) a cambio de evitar que el tráfico salga directo por Megacable.
- **Switch no administrable.** Cedí VLANs a cambio de precio. Es una limitación asumida y está en la lista de abajo.

## Limitaciones conocidas

| Limitación | Impacto | Plan |
|---|---|---|
| LAN por USB y drivers | Caída total del lab si falla | Monitoreo con Monit; evaluar otra interfaz |
| Laptop del firewall se calienta | Caída momentánea de la LAN | Ventilación constante |
| Switch sin VLANs | Los equipos se ven entre sí; el tráfico no pasa por el firewall | Conseguir un switch empresarial |
| AP sin aislamiento | Movimiento lateral posible desde el Wi-Fi | Aislar el AP |
| Router y AP sin respaldo de batería | Sin internet ni Wi-Fi en un corte de luz | Respaldo de batería |
| Doble NAT y proveedor sin puertos | Sin servicios expuestos | Tailscale (ya aplicado) |
| Kill switch de la VPN | Si cae el túnel, la LAN pierde internet hasta que se restablece | Comportamiento buscado |
| CPU del NAS de pocos hilos | Límite de VMs | Asignar recursos justos |

## Pendientes

- Configurar el firewall para los dispositivos que se conectan al AP y llevar su DNS al de OPNsense.
- Aislar correctamente el AP.
- Segmentar los servicios del NAS entre sí.
- Conseguir un switch administrable para segmentar con VLANs.
- Endurecer las reglas y los servicios DNS.
- Agregar capturas de evidencia en `docs/img/`.
