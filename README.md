<p align="center">
  <img src="https://raw.githubusercontent.com/Netec-Mx/261013-BIG-IP-ADMIN/main/assets/LogoNetec.png" alt="NETEC" width="180" />
</p>

# Administración de BIG-IP: configuración, alta disponibilidad y troubleshooting

Curso instructor-led de nivel básico orientado al desarrollo de competencias para administrar, configurar, asegurar, operar y diagnosticar soluciones F5 BIG-IP en entornos empresariales. El programa integra los cinco dominios de la certificación F5 Certified Administrator, BIG-IP (F5-CA, BIG-IP) y articula fundamentos de arquitectura, análisis del flujo de tráfico, operación mediante TMUI y TMSH, configuración de servicios de entrega de aplicaciones, alta disponibilidad, monitoreo y troubleshooting. La metodología combina explicación técnica, casos de análisis y laboratorios, con énfasis tanto en la preparación para la certificación como en la aplicación práctica de las competencias de administración cotidiana.

## Accesos rápidos

- [**Setup Guide del curso**](https://github.com/Netec-Mx/261013-BIG-IP-ADMIN/blob/main/SETUP_GUIDE.md)
- [Laboratorios por capítulo](#lista-de-laboratorios)

## Estructura

- `SETUP_GUIDE.md`: guía de instalación y preparación del entorno.
- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Laboratorio 1 — Aseguramiento del acceso administrativo: Management IP, ACLs, Port Lockdown, políticas administrativas y provisionamiento.](Capitulo01/README.md#laboratorio-1-aseguramiento-del-acceso-administrativo-management-ip-acls-port-lockdown-políticas-administrativas-y-provisionamiento)
  - Descripción: aplicar y validar controles de acceso administrativo, Port Lockdown y provisioning, y comprobar el acceso y el estado de la configuración mediante TMUI y TMSH. Alineación: F5CAB1.01, F5CAB1.02 y F5CAB1.05. Actividad orientada al examen F5CAB1.
  - Duración estimada: 35 min
- [Laboratorio 2 — Gestión del ciclo de vida del sistema: licenciamiento, Service Check Date, boot locations, software images y preparación de la actualización.](Capitulo01/README.md#laboratorio-2-gestión-del-ciclo-de-vida-del-sistema-licenciamiento-service-check-date-boot-locations-software-images-y-preparación-de-la-actualización)
  - Descripción: validar el estado de licenciamiento y provisioning, revisar boot locations e imágenes de software y preparar un procedimiento de actualización controlada. Alineación: F5CAB1.03–F5CAB1.05; relación con F5CAB4.09. Actividad orientada al examen F5CAB1.
  - Duración estimada: 35 min
  - [Ver capítulo completo](Capitulo01/README.md)

### Capítulo 2

- [Laboratorio 3 — Construcción y análisis del camino de datos: interfaces, VLANs, Self IPs, rutas, Virtual Servers y análisis del flujo de tráfico.](Capitulo02/README.md#laboratorio-3-construcción-y-análisis-del-camino-de-datos-interfaces-vlans-self-ips-rutas-virtual-servers-y-análisis-del-flujo-de-tráfico)
  - Descripción: construir y validar el camino de datos desde el cliente hasta la aplicación, relacionando interfaces, VLANs, Self IPs, routing y la selección del Virtual Server, y verificando el comportamiento esperado del tráfico. Alineación: F5CAB2.01, F5CAB2.03 y F5CAB2.04. Actividad orientada al examen F5CAB2.
  - Duración estimada: 70 min
  - [Ver capítulo completo](Capitulo02/README.md)

### Capítulo 3

- [Laboratorio 4 — Publicación de una aplicación mediante BIG-IP: Pools, Pool Members, Health Monitors, load balancing y Virtual Servers.](Capitulo03/README.md#laboratorio-4-publicación-de-una-aplicación-mediante-big-ip-pools-pool-members-health-monitors-load-balancing-y-virtual-servers)
  - Descripción: publicar una aplicación mediante la creación y validación de un Pool, Pool Members, un Health Monitor, un método de load balancing y un Virtual Server, comprobando el funcionamiento del servicio de extremo a extremo. Alineación: F5CAB3.01–F5CAB3.02. Actividad orientada al examen F5CAB3.
  - Duración estimada: 60 min
- [Laboratorio 5 — Control avanzado de sesiones y aplicaciones: Persistence, TCP/HTTP Profiles, SSL, Priority Groups, iRules e identificación de objetos iApp.](Capitulo03/README.md#laboratorio-5-control-avanzado-de-sesiones-y-aplicaciones-persistence-tcphttp-profiles-ssl-priority-groups-irules-e-identificación-de-objetos-iapp)
  - Descripción: ampliar la configuración de la aplicación mediante Persistence, Profiles de protocolo y SSL y Priority Groups, y analizar el efecto de iRules e iApps sobre los objetos y el procesamiento del tráfico. Alineación: F5CAB3.01–F5CAB3.02. Actividad orientada al examen F5CAB3.
  - Duración estimada: 60 min
  - [Ver capítulo completo](Capitulo03/README.md)

### Capítulo 4

- [Laboratorio 6 — Alta disponibilidad, Device Trust y ConfigSync: arquitectura Active/Standby, Device Trust, Device Groups, ConfigSync, failover y recuperación de estados incorrectos.](Capitulo04/README.md#laboratorio-6-alta-disponibilidad-device-trust-y-configsync-arquitectura-activestandby-device-trust-device-groups-configsync-failover-y-recuperación-de-estados-incorrectos)
  - Descripción: interpretar evidencias de una implementación de alta disponibilidad con dos BIG-IP; determinar los estados Active/Standby, Device Trust y ConfigSync; diferenciar Force to Standby de Force Offline; analizar un escenario de failover e identificar las acciones requeridas para recuperar la operación. Alineación: F5CAB4.01 y F5CAB4.08. Actividad orientada al examen F5CAB4. Se desarrolla como caso de análisis de evidencias de dos BIG-IP; la topología disponible contiene un solo BIG-IP VE.
  - Duración estimada: 60 min
- [Laboratorio 7 — Administración y recuperación del Control Plane: UCS, usuarios, servicios, logs, estado del sistema y validación de servicios.](Capitulo04/README.md#laboratorio-7-administración-y-recuperación-del-control-plane-ucs-usuarios-servicios-logs-estado-del-sistema-y-validación-de-servicios)
  - Descripción: administrar y recuperar componentes del Control Plane mediante UCS, gestión de usuarios y servicios, análisis de logs, revisión del estado de los objetos y validación de puertos y servicios. Alineación: F5CAB4.03–F5CAB4.07 y F5CAB4.10. Actividad orientada al examen F5CAB4.
  - Duración estimada: 60 min
  - [Ver capítulo completo](Capitulo04/README.md)

### Capítulo 5

- [Laboratorio 8 — Diagnóstico de red y rendimiento en BIG-IP: estadísticas, interfaces, tcpdump, drops, TCP y utilización de recursos.](Capitulo05/README.md#laboratorio-8-diagnóstico-de-red-y-rendimiento-en-big-ip-estadísticas-interfaces-tcpdump-drops-tcp-y-utilización-de-recursos)
  - Descripción: diagnosticar problemas de rendimiento mediante la correlación de utilización de recursos, estadísticas de interfaces y packet captures con tcpdump, documentando la evidencia que sustenta la causa identificada. Alineación: F5CAB5.01, F5CAB5.02, F5CAB5.06 y F5CAB5.07. Actividad orientada al examen F5CAB5.
  - Duración estimada: 60 min
- [Laboratorio 9 — Troubleshooting integral de una aplicación: Virtual Server, Pool, Health Monitor, Persistence, Profiles, Priority Groups, límites y análisis del tráfico de extremo a extremo.](Capitulo05/README.md#laboratorio-9-troubleshooting-integral-de-una-aplicación-virtual-server-pool-health-monitor-persistence-profiles-priority-groups-límites-y-análisis-del-tráfico-de-extremo-a-extremo)
  - Descripción: resolver un escenario integral con múltiples fallas mediante una metodología sistemática de troubleshooting, priorizando evidencias, identificando causas y restableciendo el servicio de extremo a extremo. Alineación: F5CAB5.03–F5CAB5.07; integra competencias desarrolladas en F5CAB1–F5CAB4. Actividad orientada al examen F5CAB5.
  - Duración estimada: 60 min
  - [Ver capítulo completo](Capitulo05/README.md)

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.

---

*Material didáctico preparado por Global K, S.A. de C.V.*
