# Laboratorio 6 — Alta disponibilidad, Device Trust y ConfigSync

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Alineación** | F5CAB4.01, F5CAB4.08 |
| **Versión BIG-IP** | F5 BIG-IP VE 17.1.1.3 |


**Guía autoformativa para construir, razonar y validar un diseño HA**

- **Nivel:** Fundamentos / Administración de BIG-IP
- **Duración:** 30–35 minutos
- **Modalidad:** individual o en parejas
- **Propósito:** comprender cómo dos BIG-IP trabajan juntos para mantener disponible una aplicación y cómo evolucionar de Active/Standby a Active/Active.

> **Cómo usar esta actividad:** lee el contexto de cada sección, completa los diagramas o decisiones solicitadas y no consultes el apartado de respuestas hasta terminar.

## Resultados de aprendizaje
- Distinguir Management IP, Self IP local, Floating Self IP y Virtual Address.
- Explicar la función de Device Trust, Sync-Failover Device Group, ConfigSync, Network Failover y Mirroring.
- Determinar qué objetos deben ser flotantes para que una aplicación sobreviva a un failover.
- Explicar la relación entre Traffic Groups y los modelos Active/Standby y Active/Active.
- Identificar dependencias de aplicación que deben moverse juntas, como Virtual Address, Floating Self IP y SNAT Translation.

---

## 1. El problema que debes resolver
La empresa **ACME** publica una aplicación web mediante un único BIG-IP. BIGIP01 recibe las conexiones y las distribuye a dos servidores web. Si BIGIP01 falla, la aplicación deja de estar disponible aunque los servidores sigan funcionando.

```text
USUARIOS
   |
10.1.10.100  (aplicación WEB)
   |
BIGIP01
   |
+-- WEB-SERVER01 10.1.20.11
+-- WEB-SERVER02 10.1.20.12
```

La empresa incorpora **BIGIP02**. Tu misión es convertir ambos equipos en una solución de alta disponibilidad.

### Referencia: tipos de direcciones
| Tipo | Para qué sirve | ¿Se mueve durante failover? |
|---|---|---|
| Management IP | Administrar el BIG-IP | No |
| Self IP local | El BIG-IP se comunica con una red | No |
| Floating Self IP | Dirección compartida que acompaña al equipo activo | Sí |
| Virtual Address | Dirección publicada de la aplicación | Sí |

### Datos de los dispositivos
| Función | BIGIP01 | BIGIP02 |
|---|---:|---:|
| Management | 10.1.1.245 | 10.1.1.246 |
| Client Self IP | 10.1.10.245 | 10.1.10.246 |
| Server Self IP | 10.1.20.245 | 10.1.20.246 |
| Self IP para comunicación HA | 10.1.30.245 | 10.1.30.246 |

La red `10.1.30.0/24` se utilizará exclusivamente para comunicación de alta disponibilidad entre los dos BIG-IP.

## 2. Construye la comunicación de HA — 5 min
### Contexto
Los dos dispositivos necesitan una red por la que intercambiar información de sincronización, failover y mirroring. En este caso se utiliza `vlan-30`, interfaz `1.3`, red `10.1.30.0/24`. Cada equipo tiene su propia Self IP local en esa red.

> **Idea clave:** `10.1.30.245` y `10.1.30.246` no son direcciones flotantes. Cada una pertenece permanentemente a su BIG-IP.

### Actividad
Completa el diseño:
```text
BIGIP01                              BIGIP02
Management: ____________            Management: ____________
Client Self IP: _________           Client Self IP: _________
Server Self IP: _________           Server Self IP: _________

           RED DE HA 10.1.30.0/24
      _____________ <--------> _____________
      Self IP HA               Self IP HA
```

Ahora completa:
| Función | BIGIP01 | BIGIP02 |
|---|---|---|
| ConfigSync | __________ | __________ |
| Network Failover | __________ | __________ |
| Mirroring | __________ | __________ |

En este laboratorio la Self IP de HA utiliza **Port Lockdown = Allow Defaults**. Los servicios relevantes incluyen TCP 4353 para ConfigSync, TCP 1026 para Network Failover y TCP 6699 para MCP.

**Comprueba que entendiste:**
- [ ] `10.1.30.245` debe moverse de BIGIP01 a BIGIP02 durante un failover.
- [ ] `10.1.30.245` pertenece permanentemente a BIGIP01 y se utiliza para comunicación de HA.

## 3. Haz que los dispositivos confíen entre sí — 4 min
### Contexto
Aunque los BIG-IP ya pueden comunicarse por IP, todavía son dispositivos independientes. Antes de trabajar como clúster necesitan establecer una relación de confianza basada en certificados. BIG-IP denomina a esta relación **Device Trust**.

> **Modelo mental:** Device Trust responde a “¿qué otros BIG-IP son dispositivos de confianza?”.

### Actividad
Relaciona cada concepto con su función:
- A. Device Trust
- B. Network Failover
- C. Traffic Group

1. ___ Permite que dos BIG-IP confíen y se autentiquen.
2. ___ Permite detectar problemas de disponibilidad entre peers.
3. ___ Agrupa objetos que deben moverse juntos.

## 4. Forma el clúster — 4 min
### Contexto
Device Trust establece confianza, pero aún debes indicar qué dispositivos compartirán configuración y participarán en failover. Para ello se utiliza un **Device Group**. Esta solución requiere un grupo de tipo **Sync-Failover**.

Datos solicitados: `my_device_group`, miembros BIGIP01 y BIGIP02, Network Failover habilitado.

### Actividad
Completa con tus propias palabras:
- **Device Trust determina:** ________________________________________________
- **Device Group determina:** ______________________________________________

## 5. Primera sincronización — 3 min
### Contexto
BIGIP01 ya contiene la aplicación. BIGIP02 acaba de incorporarse. Después de crear el Device Group aparece **Awaiting Initial Sync**. Debes decidir qué configuración conservar.

### Actividad
- [ ] BIGIP02 → Sync Device to Group
- [ ] BIGIP01 → Sync Device to Group
- [ ] BIGIP01 → Force to Standby

Explica por qué elegiste ese dispositivo:

____________________________________________________________________________
____________________________________________________________________________

## 6. Haz que la aplicación sobreviva al failover — 4 min
### Contexto
La aplicación se publica mediante el Virtual Address `10.1.10.100`. Si BIGIP01 falla, BIGIP02 debe poder asumir esa misma dirección para que los usuarios sigan conectándose.

BIG-IP utiliza **Traffic Groups** para agrupar objetos flotantes que deben moverse juntos entre dispositivos.

### Actividad
Clasifica los objetos como **L = Local** o **F = Floating**:
| Objeto | L / F |
|---|:---:|
| Management IP 10.1.1.245 | ___ |
| HA Self IP 10.1.30.245 | ___ |
| Virtual Address 10.1.10.100 | ___ |
| Management IP 10.1.1.246 | ___ |
| HA Self IP 10.1.30.246 | ___ |

## 7. Descubre una dependencia escondida — 3 min
### Contexto
Los servidores web usan `10.1.20.240` como default gateway. El Virtual Address puede moverse a BIGIP02, pero si `10.1.20.240` permanece únicamente en BIGIP01, el retorno de tráfico puede romperse.

### Actividad
¿Qué problema podría producirse si BIGIP01 falla y `10.1.20.240` no se mueve?

____________________________________________________________________________
____________________________________________________________________________

Decide cómo debería configurarse `10.1.20.240`:
- [ ] Local Self IP
- [ ] Floating Self IP

¿A qué Traffic Group debería pertenecer? ______________________________

## 8. Mantén la sesión del usuario — 3 min
### Contexto
La aplicación usa Source Address Persistence. El cliente `192.168.10.50` quedó asociado a `WEB-SERVER01 10.1.20.11`. BIGIP01 conoce esa relación; BIGIP02 no.

### Actividad
¿Qué función permite que el peer conozca los registros de persistencia necesarios después de un failover?

**Respuesta:** ______________________________________________

¿Los persistence records que existían antes de habilitar esta función se copian automáticamente al standby?
- [ ] Sí
- [ ] No

## 9. Integra el modelo — 2 min
Completa el flujo:
```text
____________________
        |
        v
____________________
        |
        v
____________________
        |
        v
____________________
```

Y relaciona cada necesidad con una función BIG-IP:
| Necesidad | Función BIG-IP |
|---|---|
| Los dispositivos deben confiar entre sí | __________________ |
| Compartir configuración | __________________ |
| Detectar pérdida del peer | __________________ |
| Mover objetos entre dispositivos | __________________ |
| Conservar registros de persistencia | __________________ |

## 10. Convierte el clúster en Active/Active — 5–6 min
### Contexto
Hasta ahora, `traffic-group-1` está activo en BIGIP01 y BIGIP02 actúa como respaldo. BIG-IP permite crear más de un Traffic Group para distribuir servicios entre ambos dispositivos.

Se crea un segundo Traffic Group llamado `tg-2`. El objetivo es que:
```text
BIGIP01                         BIGIP02
traffic-group-1 ACTIVE          tg-2 ACTIVE
```

### 10.1 ¿Qué objeto debes mover a tg-2?
- [ ] Virtual Server `www_vs`
- [ ] Virtual Address `10.1.10.100`
- [ ] Pool de servidores
- [ ] HTTP Profile

### 10.2 Dos servicios comparten la misma dirección
Supón que `www_vs` usa `10.1.10.100:80` y `ftp_vs` usa `10.1.10.100:21`. Ambos comparten el mismo Virtual Address `10.1.10.100`. Si ese Virtual Address se asigna a `tg-2`, ¿qué sucede?

A. Sólo `www_vs` cambia a `tg-2`.
B. Sólo `ftp_vs` cambia a `tg-2`.
C. Los listeners asociados a ese Virtual Address quedan ligados al mismo Traffic Group.
D. Ningún servicio se ve afectado.

### 10.3 ¿Por qué el resultado es Active/Active?
A. Porque sólo puede existir un BIG-IP Active.
B. Porque cada BIG-IP posee al menos un Traffic Group activo.
C. Porque existen dos Device Groups.
D. Porque ambos usan la misma Management IP.

### 10.4 El sitio web funciona, pero FTP falla
El Virtual Address `10.1.10.100` ya está en `tg-2`, pero la SNAT Translation utilizada por FTP sigue en `traffic-group-1`. ¿Cuál es el problema?

A. Los objetos necesarios para FTP están activos en distintos BIG-IP.
B. FTP no puede funcionar en un Device Group.
C. Los Traffic Groups sólo soportan HTTP.
D. Es necesario volver a crear Device Trust.

### 10.5 Corrige el diseño
Completa la regla:

> Los objetos flotantes que dependen unos de otros para proporcionar una aplicación deberían ____________________________________________________________________.

### 10.6 Failover dentro de un diseño Active/Active
Estado normal: `traffic-group-1` activo en BIGIP01 y `tg-2` activo en BIGIP02. BIGIP02 falla. ¿Qué debería suceder con `tg-2`?

A. Se elimina.
B. Permanece en BIGIP02.
C. Se mueve a un dispositivo elegible, por ejemplo BIGIP01.
D. Convierte automáticamente el Device Group en Sync-Only.

---

# Respuestas y explicación

## 1. Problema inicial
BIGIP01 representa un punto único de falla. La incorporación de BIGIP02 permite crear redundancia para que el servicio continúe cuando uno de los dispositivos deja de estar disponible.

## 2. Comunicación de HA
| Elemento | BIGIP01 | BIGIP02 |
|---|---:|---:|
| Management | 10.1.1.245 | 10.1.1.246 |
| Client Self IP | 10.1.10.245 | 10.1.10.246 |
| Server Self IP | 10.1.20.245 | 10.1.20.246 |
| Self IP para HA | 10.1.30.245 | 10.1.30.246 |

| Función | BIGIP01 | BIGIP02 |
|---|---|---|
| ConfigSync | 10.1.30.245 | 10.1.30.246 |
| Network Failover | 10.1.30.245 | 10.1.30.246 |
| Mirroring | 10.1.30.245 | 10.1.30.246 |

La afirmación correcta es: **10.1.30.245 pertenece permanentemente a BIGIP01 y se utiliza para comunicación de HA.**

## 3. Device Trust
1. A — Device Trust permite confianza y autenticación entre los BIG-IP.
2. B — Network Failover permite detectar problemas de disponibilidad entre peers.
3. C — Traffic Group agrupa objetos flotantes que deben moverse juntos.

## 4. Device Group
- **Device Trust:** determina qué BIG-IP confían entre sí.
- **Device Group:** determina qué dispositivos trabajan juntos para sincronización y, cuando es Sync-Failover, para failover.
- Configuración del caso: `my_device_group`, tipo **Sync-Failover**, miembros BIGIP01 y BIGIP02, Network Failover habilitado.

## 5. Primera sincronización
Respuesta: **BIGIP01 → Sync Device to Group**. BIGIP01 contiene la configuración que se quiere conservar y distribuir. La regla es identificar el origen correcto antes de ejecutar el initial sync.

## 6. Objetos locales y flotantes
| Objeto | Tipo |
|---|---|
| Management IP 10.1.1.245 | Local |
| HA Self IP 10.1.30.245 | Local |
| Virtual Address 10.1.10.100 | Floating |
| Management IP 10.1.1.246 | Local |
| HA Self IP 10.1.30.246 | Local |

Regla: **Local pertenece al dispositivo; Floating pertenece al servicio.**

## 7. Gateway de servidores
Si `10.1.20.240` permanece únicamente en BIGIP01, después del failover los servidores pueden perder el camino de retorno correcto. Debe configurarse como **Floating Self IP** y asociarse a `traffic-group-1` para acompañar al servicio.

## 8. Persistence
La función es **Mirror Persistence**. En este laboratorio, los registros creados antes de habilitar mirroring **no** aparecen automáticamente en el standby; sólo se reflejan los nuevos registros creados después de habilitar la función.

## 9. Modelo integrado
Flujo esperado: **Device Trust → Sync-Failover Device Group → Traffic Group → Floating Objects**.

| Necesidad | Función BIG-IP |
|---|---|
| Los dispositivos deben confiar entre sí | Device Trust |
| Compartir configuración | ConfigSync |
| Detectar pérdida del peer | Network Failover |
| Mover objetos entre dispositivos | Traffic Group |
| Conservar registros de persistencia | Persistence Mirroring |

## 10. Active/Active
### 10.1
Respuesta: **Virtual Address `10.1.10.100`**. El Traffic Group se asigna al Virtual Address, no directamente al Virtual Server.

### 10.2
Respuesta: **C**. Los listeners que comparten ese Virtual Address quedan ligados al Traffic Group del Virtual Address.

### 10.3
Respuesta: **B**. Es Active/Active porque BIGIP01 tiene `traffic-group-1` activo y BIGIP02 tiene `tg-2` activo; ambos procesan tráfico simultáneamente.

### 10.4
Respuesta: **A**. El Virtual Address está en `tg-2`, pero la SNAT Translation necesaria para FTP sigue en otro Traffic Group; la aplicación queda dividida entre dispositivos.

### 10.5
Regla: **los objetos flotantes que dependen unos de otros para proporcionar una aplicación deberían pertenecer al mismo Traffic Group para moverse juntos durante failover.**

### 10.6
Respuesta: **C**. `tg-2` puede moverse a otro dispositivo elegible, por ejemplo BIGIP01. Active/Active distribuye la carga normal, pero conserva la capacidad de failover.

## Mapa mental final
```text
Device Trust
     ↓
Sync-Failover Device Group
     ↓
Traffic Groups
     ↓
Floating Objects

ConfigSync        = configuración
Network Failover  = detección/cambio de propietario
Mirroring         = estado de ejecución
```

### Recursos adicionales

| Recurso | URL |
|---|---|
| F5 BIG-IP TMSH Reference (17.x) | [https://clouddocs.f5.com/cli/tmsh-reference/latest/](https://clouddocs.f5.com/cli/tmsh-reference/latest/) |
| K13132: Backing up and restoring BIG-IP configuration files with a UCS archive | [https://my.f5.com/manage/s/article/K13132](https://my.f5.com/manage/s/article/K13132) |
| K4423: Overview of UCS archives | [https://my.f5.com/manage/s/article/K4423](https://my.f5.com/manage/s/article/K4423) |
| K13400: BIG-IP log files and their descriptions | [https://my.f5.com/manage/s/article/K13400](https://my.f5.com/manage/s/article/K13400) |
| K15930: Overview of BIG-IP user roles | [https://my.f5.com/manage/s/article/K15930](https://my.f5.com/manage/s/article/K15930) |
| K13309: Restricting access to the Configuration utility and SSH by source IP address | [https://my.f5.com/manage/s/article/K13309](https://my.f5.com/manage/s/article/K13309) |
