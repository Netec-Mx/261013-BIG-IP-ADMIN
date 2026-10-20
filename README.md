<a href="https://netec.com">
  <img src="https://static.wixstatic.com/media/96edd2_934dddb3230b4b16b1d1c56c51143738~mv2.png" alt="Logo de Netec" width="300"/>
</a>

# Administración de BIG-IP: configuración, alta disponibilidad y troubleshooting

## Plataforma de laboratorios

Te damos la bienvenida a la **plataforma de laboratorios** del curso **Administración de BIG-IP: configuración, alta disponibilidad y troubleshooting**. Aquí aplicarás la configuración, seguridad, operación y diagnóstico de F5 BIG-IP mediante laboratorios guiados y un caso de análisis de alta disponibilidad.


## Lista de laboratorios

Haz clic en el título de cada actividad para ir a su sección dentro de la guía correspondiente. Los nombres de los módulos, las actividades y sus duraciones siguen el temario del curso.

## Módulo 1: Instalación, configuración inicial y administración segura de BIG-IP

### [Laboratorio 1 — Aseguramiento del acceso administrativo: Management IP, ACLs, Port Lockdown, políticas administrativas y provisionamiento](Capitulo01/README.md#laboratorio-1--aseguramiento-del-acceso-administrativo-management-ip-acls-port-lockdown-políticas-administrativas-y-provisionamiento)

- **Descripción**: Configura la Management IP, ACLs administrativas, Port Lockdown y políticas de contraseñas. Comprueba el licenciamiento y el provisionamiento de LTM, y valida el acceso permitido y rechazado mediante TMUI y TMSH.
- ⏱️ **Duración estimada**: 35 min.

### [Laboratorio 2 — Gestión del ciclo de vida del sistema: licenciamiento, Service Check Date, boot locations, software images y preparación de la actualización](Capitulo01/README.md#laboratorio-2--gestión-del-ciclo-de-vida-del-sistema-licenciamiento-service-check-date-boot-locations-software-images-y-preparación-de-la-actualización)

- **Descripción**: Revisa el licenciamiento, Service Check Date, provisionamiento, volúmenes de arranque e imágenes de software. Instala una imagen en un volumen inactivo y documenta la actualización controlada y su procedimiento de rollback.
- ⏱️ **Duración estimada**: 35 min.

## Módulo 2: Arquitectura del Data Plane y procesamiento de tráfico

### [Laboratorio 3 — Construcción y análisis del camino de datos: interfaces, VLANs, Self IPs, rutas, Virtual Servers y análisis del flujo de tráfico](Capitulo02/README.md#laboratorio-3--construcción-y-análisis-del-camino-de-datos-interfaces-vlans-self-ips-rutas-virtual-servers-y-análisis-del-flujo-de-tráfico)

- **Descripción**: Construye el camino de datos con interfaces, VLANs, Self IPs, rutas, un pool, un monitor HTTP y un Virtual Server. Verifica la conectividad y analiza con tcpdump las conexiones cliente–BIG-IP–servidor y el comportamiento full-proxy.
- ⏱️ **Duración estimada**: 70 min.

## Módulo 3: Configuración del Data Plane y entrega de aplicaciones

### [Laboratorio 4 — Publicación de una aplicación mediante BIG-IP: Pools, Pool Members, Health Monitors, load balancing y Virtual Servers](Capitulo03/README.md#laboratorio-4--publicación-de-una-aplicación-mediante-big-ip-pools-pool-members-health-monitors-load-balancing-y-virtual-servers)

- **Descripción**: Publica una aplicación web mediante Nodes, Pool Members, un Pool Round Robin, un Health Monitor HTTP y un Virtual Server. Comprueba la distribución del tráfico, la detección de fallos de los servidores y la recuperación del servicio.
- ⏱️ **Duración estimada**: 60 min.

### [Laboratorio 5 — Control avanzado de sesiones y aplicaciones: Persistence, TCP/HTTP Profiles, SSL, Priority Groups, iRules e identificación de objetos iApp](Capitulo03/README.md#laboratorio-5--control-avanzado-de-sesiones-y-aplicaciones-persistence-tcphttp-profiles-ssl-priority-groups-irules-e-identificación-de-objetos-iapp)

- **Descripción**: Configura persistencia de sesiones, perfiles TCP/HTTP y SSL, Priority Groups y una iRule condicional. Valida el comportamiento de la aplicación e identifica dependencias y restricciones de objetos generados por iApps.
- ⏱️ **Duración estimada**: 60 min.

## Módulo 4: Control Plane, alta disponibilidad y administración del sistema

### [Caso de análisis 6 — Alta disponibilidad, Device Trust y ConfigSync: arquitectura Active/Standby, Device Trust, Device Groups, ConfigSync, failover y recuperación de estados incorrectos](Capitulo04/README.md#laboratorio-6--alta-disponibilidad-device-trust-y-configsync)

- **Descripción**: Interpreta evidencias de un par BIG-IP Active/Standby y analiza Device Trust, Device Groups y ConfigSync. Diferencia Force to Standby de Force Offline, evalúa un escenario de failover e identifica acciones para recuperar estados de sincronización incorrectos. Se realiza como caso de análisis, conforme al temario.
- ⏱️ **Duración estimada**: 60 min.

### [Laboratorio 7 — Administración y recuperación del Control Plane: UCS, usuarios, servicios, logs, estado del sistema y validación de servicios](Capitulo04/README.md#laboratorio-7--administración-y-recuperación-del-control-plane-ucs-usuarios-servicios-logs-estado-del-sistema-y-validación-de-servicios)

- **Descripción**: Crea y verifica respaldos UCS, administra usuarios y configura servicios NTP, DNS, Syslog y SNMP. Revisa logs, puertos y servicios del Control Plane, y ejecuta una restauración con validación posterior.
- ⏱️ **Duración estimada**: 60 min.

## Módulo 5: Monitoreo, soporte y troubleshooting de BIG-IP

### [Laboratorio 8 — Diagnóstico de red y rendimiento en BIG-IP: estadísticas, interfaces, tcpdump, drops, TCP y utilización de recursos](Capitulo05/README.md#laboratorio-8--diagnóstico-de-red-y-rendimiento-en-big-ip-estadísticas-interfaces-tcpdump-drops-tcp-y-utilización-de-recursos)

- **Descripción**: Recopila métricas de CPU, memoria, disco e interfaces y analiza fallos controlados mediante estadísticas y capturas tcpdump. Correlaciona evidencias para diagnosticar degradaciones y genera un QKView para su análisis en iHealth.
- ⏱️ **Duración estimada**: 60 min.

### [Laboratorio 9 — Troubleshooting integral de una aplicación: Virtual Server, Pool, Health Monitor, Persistence, Profiles, Priority Groups, límites y análisis del tráfico de extremo a extremo](Capitulo05/README.md#laboratorio-9--troubleshooting-integral-de-una-aplicación-virtual-server-pool-health-monitor-persistence-profiles-priority-groups-límites-y-análisis-del-tráfico-de-extremo-a-extremo)

- **Descripción**: Diagnostica y corrige seis fallas simultáneas en una aplicación HTTPS mediante una metodología de identificación, aislamiento, diagnóstico, corrección y verificación. Revisa objetos LTM, genera un QKView y documenta las evidencias de recuperación.
- ⏱️ **Duración estimada**: 60 min.

---

## 📬 Contacto y más información

Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).

---

¡Gracias por visitar nuestra plataforma! Revisa las actividades y comienza tu experiencia de aprendizaje.
