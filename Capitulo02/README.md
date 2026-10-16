# Laboratorio 3 — Construcción y análisis del camino de datos: interfaces, VLANs, Self IPs, rutas, Virtual Servers y análisis del flujo de tráfico

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 70 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio se construye de forma incremental el camino de datos completo de un BIG-IP VE 17.1.2 partiendo de una configuración de fábrica con solo la Management IP operativa. El estudiante creará VLANs, Self IPs, una ruta estática, un pool con monitor de salud HTTP y un Virtual Server de tipo Standard, verificando cada objeto mediante TMUI y tmsh. Finalmente, se analizará el flujo de tráfico extremo a extremo con `tcpdump` para evidenciar el comportamiento full-proxy de BIG-IP, correlacionando las traducciones de dirección IP observadas con la arquitectura de doble conexión cliente–BIG-IP–servidor.

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Verificar el estado operativo de las interfaces físicas/virtuales de BIG-IP (Management, 1.1, 1.2) y comprender su rol en la segmentación del plano de datos.
- [ ] Crear VLANs, Self IPs con port lockdown apropiado y una ruta estática por defecto que habiliten la comunicación bidireccional entre los segmentos external e internal.
- [ ] Construir un Virtual Server de tipo Standard con un pool de servidores backend y un health monitor HTTP, verificando que todos los objetos alcancen estado Available (verde).
- [ ] Analizar capturas de tráfico simultáneas en las interfaces external e internal con `tcpdump`, identificando las traducciones de IP que evidencian el comportamiento full-proxy y SNAT AutoMap de BIG-IP.
- [ ] Relacionar el estado de cada componente del camino de datos con la disponibilidad del servicio, identificando puntos únicos de fallo.

## Prerrequisitos

### Conocimientos previos

| Área | Detalle |
|---|---|
| TCP/IP | Modelo OSI capas 2–3, direccionamiento IPv4, subnetting CIDR |
| VLANs | IEEE 802.1Q: concepto de tagged vs. untagged, segmentación de red |
| Routing | Tablas de routing, rutas por defecto (0.0.0.0/0), concepto de gateway |
| HTTP | Modelo request-response, puerto TCP/80, cabeceras básicas |
| Lectura teórica | Temas 2.1 a 2.5 del Módulo 2 del curso completados |

### Acceso requerido

| Recurso | Credenciales / Conectividad |
|---|---|
| BIG-IP VE Management GUI | `https://192.0.2.245` — usuario `admin`, contraseña definida en la preparación del entorno |
| BIG-IP VE CLI (SSH) | `ssh admin@192.0.2.245` desde el cliente de administración `192.0.2.10` |
| Cliente de pruebas | `10.10.10.10/24` — acceso SSH y capacidad para ejecutar `curl` y navegador web |
| Servidor web01 | `10.20.20.11/24` — SSH disponible, Apache HTTP Server 2.4.58 activo en TCP/80 |
| Servidor web02 | `10.20.20.12/24` — SSH disponible, Apache HTTP Server 2.4.58 activo en TCP/80 |

## Entorno de Laboratorio

### Topología de red

```
                    Red Management: 192.0.2.0/24
                    ┌──────────────────────────────┐
                    │  Cliente Admin: 192.0.2.10    │
                    │  BIG-IP Mgmt:   192.0.2.245   │
                    └──────────────────────────────┘

  Red External: 10.10.10.0/24              Red Internal: 10.20.20.0/24
  ┌─────────────────────────┐              ┌─────────────────────────┐
  │ Cliente: 10.10.10.10    │              │ web01: 10.20.20.11      │
  │ VIP:     10.10.10.100   │              │ web02: 10.20.20.12      │
  │ BIG-IP:  10.10.10.245   │              │ BIG-IP: 10.20.20.245    │
  └────────┬────────────────┘              └────────┬────────────────┘
           │  Interfaz 1.1                          │  Interfaz 1.2
           └────────────── BIG-IP VE ───────────────┘
```

### Software del laboratorio

| Componente | Versión exacta | Fuente oficial |
|---|---|---|
| F5 BIG-IP VE | 17.1.2 | [https://my.f5.com/manage/s/downloads](https://my.f5.com/manage/s/downloads) |
| Ubuntu Server (web01, web02) | 22.04.4 LTS (amd64) | [https://releases.ubuntu.com/22.04/](https://releases.ubuntu.com/22.04/) |
| Apache HTTP Server | 2.4.58 | Incluido en repositorios de Ubuntu 22.04.4 LTS |
| curl (cliente de pruebas) | 8.5.0 | Incluido en repositorios de Ubuntu 22.04.4 LTS |
| tcpdump (BIG-IP) | Incluido en BIG-IP 17.1.2 | Preinstalado en TMOS |

### Verificación previa del entorno

Antes de iniciar, confirma que los servidores backend están operativos. Desde el cliente de administración (`192.0.2.10`), ejecuta:

```bash
## Verificar que web01 responde en HTTP
ssh usuario@10.20.20.11 "curl -s -o /dev/null -w '%{http_code}' http://localhost/"
## Resultado esperado: 200

## Verificar que web02 responde en HTTP
ssh usuario@10.20.20.12 "curl -s -o /dev/null -w '%{http_code}' http://localhost/"
## Resultado esperado: 200

## Verificar que el gateway por defecto de ambos servidores apunta a BIG-IP internal
ssh usuario@10.20.20.11 "ip route show default"
## Resultado esperado: default via 10.20.20.245 dev <interfaz>

ssh usuario@10.20.20.12 "ip route show default"
## Resultado esperado: default via 10.20.20.245 dev <interfaz>
```

Confirma también que cada servidor web muestra contenido diferenciado para facilitar la identificación durante las pruebas de balanceo:

```bash
## En web01 (10.20.20.11):
echo "<h1>Servidor: web01 — 10.20.20.11</h1>" | sudo tee /var/www/html/index.html

## En web02 (10.20.20.12):
echo "<h1>Servidor: web02 — 10.20.20.12</h1>" | sudo tee /var/www/html/index.html
```

## Instrucciones Paso a Paso

### Paso 1: Exploración del estado inicial de interfaces

**Objetivo:** Examinar las interfaces disponibles en BIG-IP VE (Management, 1.1, 1.2), verificar su estado operativo y documentar la configuración de red existente de fábrica.

**Instrucciones:**

1. Accede a la GUI de BIG-IP desde el navegador del cliente de administración:

   ```
   https://192.0.2.245
   ```

   Inicia sesión con el usuario `admin` y la contraseña configurada durante el despliegue.

2. En TMUI, navega a **Network > Interfaces**. Observa las interfaces listadas. Deberías ver al menos `mgmt`, `1.1` y `1.2`. Registra el estado (`Status`) de cada una: debe mostrar **up** con indicador verde si el enlace está activo.

3. Navega a **Network > VLANs**. Confirma que no existen VLANs configuradas (estado de fábrica).

4. Navega a **Network > Self IPs**. Confirma que no existen Self IPs de plano de datos (solo la Management IP existente en la interfaz `mgmt`).

5. Navega a **Network > Routes**. Confirma que no existen rutas estáticas configuradas para el plano de datos.

6. Abre una sesión SSH al BIG-IP y accede a tmsh:

   ```bash
   ssh admin@192.0.2.245
   ```

   ```bash
   tmsh
   ```

7. Ejecuta los siguientes comandos para verificar el estado de las interfaces desde la CLI:

   ```bash
   show net interface
   ```

8. Verifica que no existen VLANs, Self IPs ni rutas de plano de datos:

   ```bash
   list net vlan
   list net self
   list net route
   ```

9. Consulta la configuración de la interfaz de gestión:

   ```bash
   list sys management-ip
   ```

**Resultado esperado:**

La salida de `show net interface` debe mostrar tres interfaces con estado `up`:

```
--------------------------------------------------
Net::Interface
Name  Status  Bits  Bits  Pkts  Pkts  Drops  Errs
                In   Out    In   Out
--------------------------------------------------
1.1      up      0     0     0     0      0     0
1.2      up      0     0     0     0      0     0
mgmt     up   ...   ...   ...   ...    ...   ...
```

Los comandos `list net vlan`, `list net self` y `list net route` no deben devolver objetos (o solo la configuración mínima de fábrica).

El comando `list sys management-ip` debe mostrar:

```
sys management-ip 192.0.2.245/24 { }
```

**Verificación:**

- [ ] Las tres interfaces (mgmt, 1.1, 1.2) muestran estado `up` tanto en TMUI como en tmsh.
- [ ] No existen VLANs, Self IPs de plano de datos ni rutas estáticas previas.
- [ ] La Management IP es `192.0.2.245/24`.

---

### Paso 2: Configuración de VLANs

**Objetivo:** Crear dos VLANs — `external` (asignada a la interfaz 1.1 como untagged) e `internal` (asignada a la interfaz 1.2 como untagged) — para segmentar el tráfico de clientes y servidores.

**Instrucciones:**

1. **Crear la VLAN `external` mediante TMUI:**

   - Navega a **Network > VLANs > VLAN List** y haz clic en **Create**.
   - Configura los siguientes campos:
     - **Name:** `external`
     - **Tag:** `10`
     - En la sección **Interfaces**, haz clic en **Add**:
       - **Interface:** `1.1`
       - **Tagging:** `Untagged`
     - Haz clic en **Finished**.

2. **Crear la VLAN `internal` mediante TMUI:**

   - Repite el proceso con los siguientes valores:
     - **Name:** `internal`
     - **Tag:** `20`
     - **Interface:** `1.2`
     - **Tagging:** `Untagged`
     - Haz clic en **Finished**.

3. **Verificar las VLANs creadas mediante tmsh:**

   ```bash
   list net vlan external
   ```

   ```bash
   list net vlan internal
   ```

4. **Verificar el estado operativo de las VLANs:**

   ```bash
   show net vlan
   ```

> **Nota técnica:** Al configurar una interfaz como **untagged** en una VLAN, todo el tráfico que ingresa por esa interfaz sin etiqueta 802.1Q se asocia a dicha VLAN. El tag numérico (10, 20) se utiliza internamente por TMOS para identificar la VLAN, pero no se inserta en las tramas de la interfaz untagged. Si la infraestructura de virtualización ya etiqueta las tramas con tags 802.1Q específicos, sería necesario configurar las interfaces como **tagged** y hacer coincidir los tags.

**Resultado esperado:**

```
net vlan external {
    if-index 144
    interfaces {
        1.1 {
            untagged
        }
    }
    tag 10
}
```

```
net vlan internal {
    if-index 160
    interfaces {
        1.2 {
            untagged
        }
    }
    tag 20
}
```

**Verificación:**

- [ ] La VLAN `external` aparece asociada a la interfaz `1.1` como untagged con tag `10`.
- [ ] La VLAN `internal` aparece asociada a la interfaz `1.2` como untagged con tag `20`.
- [ ] Ambas VLANs muestran estado operativo activo en `show net vlan`.

---

### Paso 3: Configuración de Self IPs

**Objetivo:** Crear Self IPs no flotantes en cada VLAN con port lockdown `Allow Default` para habilitar la comunicación del plano de datos y los servicios básicos de gestión en banda.

**Instrucciones:**

1. **Crear la Self IP para la VLAN `external` mediante tmsh:**

   ```bash
   create net self self_external address 10.10.10.245/24 vlan external allow-service default
   ```

2. **Crear la Self IP para la VLAN `internal` mediante tmsh:**

   ```bash
   create net self self_internal address 10.20.20.245/24 vlan internal allow-service default
   ```

3. **Guardar la configuración:**

   ```bash
   save sys config
   ```

4. **Verificar las Self IPs creadas:**

   ```bash
   list net self
   ```

5. **Verificar mediante TMUI:** Navega a **Network > Self IPs**. Confirma que ambas Self IPs aparecen con los valores correctos.

6. **Probar la conectividad desde el cliente de pruebas** (`10.10.10.10`):

   ```bash
   ping -c 4 10.10.10.245
   ```

7. **Probar la conectividad desde web01** (`10.20.20.11`):

   ```bash
   ping -c 4 10.20.20.245
   ```

> **Reflexión de seguridad:** La opción `allow-service default` permite protocolos como ICMP, SSH (TCP/22) y ciertos servicios de TMOS en la Self IP. En un entorno de producción, se recomienda evaluar `allow-service none` o `allow-service` con una lista explícita de puertos para minimizar la superficie de ataque. En este laboratorio se utiliza `default` para facilitar la verificación y el diagnóstico.

**Resultado esperado:**

```
net self self_external {
    address 10.10.10.245/24
    allow-service {
        default
    }
    traffic-group traffic-group-local-only
    vlan external
}
net self self_internal {
    address 10.20.20.245/24
    allow-service {
        default
    }
    traffic-group traffic-group-local-only
    vlan internal
}
```

El ping desde el cliente (`10.10.10.10`) hacia `10.10.10.245` debe ser exitoso:

```
PING 10.10.10.245 (10.10.10.245) 56(84) bytes of data.
64 bytes from 10.10.10.245: icmp_seq=1 ttl=255 time=0.8 ms
...
4 packets transmitted, 4 received, 0% packet loss
```

El ping desde web01 (`10.20.20.11`) hacia `10.20.20.245` debe ser igualmente exitoso.

**Verificación:**

- [ ] `list net self` muestra ambas Self IPs con las direcciones, VLANs y port lockdown correctos.
- [ ] El cliente de pruebas puede hacer ping a `10.10.10.245`.
- [ ] El servidor web01 puede hacer ping a `10.20.20.245`.
- [ ] Ambas Self IPs aparecen en TMUI con estado operativo correcto.

---

### Paso 4: Configuración de routing estático

**Objetivo:** Definir una ruta estática por defecto en BIG-IP que garantice que el tráfico de retorno desde el segmento de servidores pueda alcanzar redes externas, y verificar la alcanzabilidad extremo a extremo.

**Instrucciones:**

1. **Analizar la tabla de rutas actual:**

   ```bash
   show net route
   ```

   Observa que BIG-IP ya conoce las redes directamente conectadas (`10.10.10.0/24` y `10.20.20.0/24`) pero no tiene ruta por defecto para alcanzar redes externas.

2. **Crear la ruta por defecto mediante tmsh.** En esta topología de laboratorio, el gateway por defecto del BIG-IP en la red external es `10.10.10.1` (si existe un router/gateway en la VLAN external). Sin embargo, dado que el cliente de pruebas está en la misma subred (`10.10.10.0/24`), la ruta por defecto no es estrictamente necesaria para la comunicación directa cliente–BIG-IP. No obstante, se configura como práctica estándar:

   ```bash
   create net route default_route network 0.0.0.0/0 gw 10.10.10.1
   ```

   > **Nota:** Si en tu entorno de laboratorio no existe un gateway en `10.10.10.1`, puedes omitir este paso o configurar una ruta alternativa. La comunicación entre el cliente (`10.10.10.10`) y el Virtual Server (`10.10.10.100`) funcionará sin ruta por defecto porque ambos están en la misma subred. La ruta por defecto es necesaria cuando los clientes provienen de redes remotas.

3. **Verificar la tabla de rutas:**

   ```bash
   list net route
   ```

4. **Verificar mediante TMUI:** Navega a **Network > Routes**. Confirma que la ruta aparece listada.

5. **Verificar la conectividad bidireccional completa.** Desde BIG-IP, prueba la alcanzabilidad hacia los servidores backend:

   ```bash
   run util ping -c 4 10.20.20.11
   run util ping -c 4 10.20.20.12
   ```

6. **Verificar la alcanzabilidad desde BIG-IP hacia el cliente:**

   ```bash
   run util ping -c 4 10.10.10.10
   ```

**Resultado esperado:**

```
net route default_route {
    gw 10.10.10.1
    network default
}
```

Los pings desde BIG-IP hacia los servidores y el cliente deben ser exitosos con 0% de pérdida.

**Verificación:**

- [ ] `list net route` muestra la ruta por defecto configurada.
- [ ] BIG-IP puede hacer ping a `10.20.20.11` y `10.20.20.12`.
- [ ] BIG-IP puede hacer ping a `10.10.10.10`.
- [ ] La tabla de rutas muestra las redes conectadas y la ruta por defecto.

---

### Paso 5: Construcción del Virtual Server con pool y health monitor

**Objetivo:** Crear un health monitor HTTP, un pool con dos miembros (web01 y web02) y un Virtual Server de tipo Standard en `10.10.10.100:80`, habilitando SNAT AutoMap, y verificar que todos los objetos alcancen estado Available (verde).

**Instrucciones:**

1. **Crear un health monitor HTTP personalizado mediante tmsh:**

   ```bash
   create ltm monitor http monitor_http_lab send "GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n" recv "200 OK" interval 10 timeout 31
   ```

   > **Explicación:** Este monitor envía una solicitud HTTP GET a la raíz (`/`) de cada pool member cada 10 segundos. Si no recibe una respuesta que contenga `200 OK` en 31 segundos (3 intentos fallidos + 1 segundo de margen), marca el miembro como no disponible.

2. **Crear el pool con los dos servidores backend:**

   ```bash
   create ltm pool pool_web_lab members add { 10.20.20.11:80 { address 10.20.20.11 } 10.20.20.12:80 { address 10.20.20.12 } } monitor monitor_http_lab load-balancing-mode round-robin
   ```

3. **Verificar el estado del pool y sus miembros:**

   ```bash
   show ltm pool pool_web_lab
   show ltm pool pool_web_lab members
   ```

4. **Crear el Virtual Server de tipo Standard:**

   ```bash
   create ltm virtual vs_web_lab destination 10.10.10.100:80 ip-protocol tcp pool pool_web_lab profiles add { tcp { } http { } } source-address-translation { type automap }
   ```

   > **Nota sobre SNAT AutoMap:** Al configurar `source-address-translation { type automap }`, BIG-IP reemplaza la dirección IP de origen del cliente con la Self IP de la VLAN por la cual envía el tráfico al servidor (en este caso, `10.20.20.245`). Esto garantiza que las respuestas del servidor regresen a BIG-IP y no intenten llegar directamente al cliente. Esta traducción es la evidencia clave del comportamiento full-proxy que se analizará en el Paso 6.

5. **Guardar la configuración:**

   ```bash
   save sys config
   ```

6. **Verificar el estado del Virtual Server:**

   ```bash
   show ltm virtual vs_web_lab
   ```

7. **Verificar todos los objetos en TMUI:**

   - Navega a **Local Traffic > Virtual Servers > Virtual Server List**. Confirma que `vs_web_lab` muestra un círculo verde (Available).
   - Navega a **Local Traffic > Pools > Pool List**. Confirma que `pool_web_lab` muestra un círculo verde.
   - Haz clic en `pool_web_lab` y luego en la pestaña **Members**. Confirma que ambos miembros (`10.20.20.11:80` y `10.20.20.12:80`) muestran círculos verdes.

8. **Probar el acceso desde el cliente de pruebas** (`10.10.10.10`):

   ```bash
   curl -s http://10.10.10.100/
   ```

9. **Ejecutar múltiples solicitudes para observar el balanceo Round Robin:**

   ```bash
   for i in $(seq 1 6); do echo "--- Solicitud $i ---"; curl -s http://10.10.10.100/; echo; done
   ```

**Resultado esperado:**

El estado del Virtual Server debe mostrar `available` y estadísticas de tráfico:

```
Ltm::Virtual Server: vs_web_lab
  Availability     : available
  State            : enabled
  Reason           : The virtual server is available
  ...
```

El estado del pool debe mostrar ambos miembros como `available`:

```
Ltm::Pool: pool_web_lab
  Availability : available
  State        : enabled
  ...
  Ltm::Pool Member: 10.20.20.11:80
    Availability : available
    State        : enabled
  Ltm::Pool Member: 10.20.20.12:80
    Availability : available
    State        : enabled
```

Las solicitudes `curl` deben alternar entre los dos servidores:

```
--- Solicitud 1 ---
<h1>Servidor: web01 — 10.20.20.11</h1>
--- Solicitud 2 ---
<h1>Servidor: web02 — 10.20.20.12</h1>
--- Solicitud 3 ---
<h1>Servidor: web01 — 10.20.20.11</h1>
--- Solicitud 4 ---
<h1>Servidor: web02 — 10.20.20.12</h1>
--- Solicitud 5 ---
<h1>Servidor: web01 — 10.20.20.11</h1>
--- Solicitud 6 ---
<h1>Servidor: web02 — 10.20.20.12</h1>
```

**Verificación:**

- [ ] El monitor `monitor_http_lab` existe y está configurado con `send`, `recv`, `interval` y `timeout` correctos.
- [ ] El pool `pool_web_lab` contiene dos miembros y ambos están en estado Available (verde).
- [ ] El Virtual Server `vs_web_lab` está en estado Available (verde) y escucha en `10.10.10.100:80`.
- [ ] Las solicitudes `curl` desde el cliente devuelven contenido HTTP alternando entre web01 y web02.
- [ ] SNAT AutoMap está habilitado en el Virtual Server.

---

### Paso 6: Análisis del flujo de tráfico con tcpdump

**Objetivo:** Ejecutar capturas de tráfico simultáneas en las interfaces external e internal de BIG-IP mientras se genera tráfico HTTP desde el cliente, para observar la traducción de direcciones IP que evidencia el comportamiento full-proxy y SNAT AutoMap.

**Instrucciones:**

1. **Abrir dos sesiones SSH simultáneas al BIG-IP** (`192.0.2.245`). Denomínalas **Terminal A** (captura en external) y **Terminal B** (captura en internal).

2. **En Terminal A**, inicia la captura en la VLAN external filtrando el tráfico del Virtual Server:

   ```bash
   tcpdump -nni external -c 50 host 10.10.10.100 and port 80
   ```

3. **En Terminal B**, inicia la captura en la VLAN internal filtrando el tráfico HTTP hacia los servidores:

   ```bash
   tcpdump -nni internal -c 50 port 80
   ```

4. **Desde el cliente de pruebas** (`10.10.10.10`), genera una solicitud HTTP:

   ```bash
   curl -s http://10.10.10.100/
   ```

5. **Analiza la captura en Terminal A** (VLAN external). Deberías observar paquetes entre el **cliente** y el **Virtual Server**:

   ```
   10.10.10.10.xxxxx > 10.10.10.100.80: Flags [S], ...
   10.10.10.100.80 > 10.10.10.10.xxxxx: Flags [S.], ...
   10.10.10.10.xxxxx > 10.10.10.100.80: Flags [.], ...
   10.10.10.10.xxxxx > 10.10.10.100.80: Flags [P.], ... HTTP GET /
   10.10.10.100.80 > 10.10.10.10.xxxxx: Flags [P.], ... HTTP 200 OK
   ...
   ```

   **Observación clave:** En la VLAN external, la dirección de destino es `10.10.10.100` (la VIP) y la dirección de origen es `10.10.10.10` (el cliente). BIG-IP responde como `10.10.10.100`.

6. **Analiza la captura en Terminal B** (VLAN internal). Deberías observar paquetes entre **BIG-IP** y el **servidor seleccionado**:

   ```
   10.20.20.245.xxxxx > 10.20.20.11.80: Flags [S], ...
   10.20.20.11.80 > 10.20.20.245.xxxxx: Flags [S.], ...
   10.20.20.245.xxxxx > 10.20.20.11.80: Flags [.], ...
   10.20.20.245.xxxxx > 10.20.20.11.80: Flags [P.], ... HTTP GET /
   10.20.20.11.80 > 10.20.20.245.xxxxx: Flags [P.], ... HTTP 200 OK
   ...
   ```

   **Observación clave:** En la VLAN internal:
   - La **dirección de origen** es `10.20.20.245` (la Self IP internal de BIG-IP), **no** `10.10.10.10` (el cliente). Esto es el efecto de SNAT AutoMap.
   - La **dirección de destino** es `10.20.20.11` o `10.20.20.12` (el pool member seleccionado), **no** `10.10.10.100` (la VIP).

7. **Documenta la correlación full-proxy** completando la siguiente tabla con los datos observados:

   | Atributo | Conexión cliente (external) | Conexión servidor (internal) |
   |---|---|---|
   | IP de origen | `10.10.10.10` (cliente) | `10.20.20.245` (Self IP BIG-IP) |
   | IP de destino | `10.10.10.100` (VIP) | `10.20.20.11` o `.12` (pool member) |
   | Puerto de origen | Efímero del cliente | Efímero de BIG-IP |
   | Puerto de destino | `80` | `80` |

8. **Genera múltiples solicitudes y observa la distribución:**

   ```bash
   for i in $(seq 1 4); do curl -s http://10.10.10.100/ > /dev/null; done
   ```

   En Terminal B, observa cómo las conexiones se distribuyen entre `10.20.20.11` y `10.20.20.12`.

9. **Detén las capturas** con `Ctrl+C` si aún están activas.

10. **(Opcional) Captura a archivo para análisis posterior:**

    ```bash
    tcpdump -nni external -c 100 -w /var/tmp/capture_external.pcap host 10.10.10.100 and port 80
    tcpdump -nni internal -c 100 -w /var/tmp/capture_internal.pcap port 80
    ```

    Estos archivos pueden descargarse y abrirse con Wireshark para un análisis más detallado.

**Resultado esperado:**

La comparación de las capturas debe evidenciar claramente **dos conexiones TCP independientes**:

- **Conexión 1 (external):** Cliente (`10.10.10.10`) ↔ VIP (`10.10.10.100`)
- **Conexión 2 (internal):** BIG-IP Self IP (`10.20.20.245`) ↔ Pool Member (`10.20.20.11` o `10.20.20.12`)

Esta separación demuestra que BIG-IP **no reenvía paquetes** del cliente al servidor. En su lugar, **termina** la conexión del cliente y **crea** una nueva conexión hacia el servidor. Este es el comportamiento definitorio de la arquitectura **full-proxy**.

**Verificación:**

- [ ] En la captura de la VLAN external, la IP de origen es la del cliente (`10.10.10.10`) y la de destino es la VIP (`10.10.10.100`).
- [ ] En la captura de la VLAN internal, la IP de origen es la Self IP de BIG-IP (`10.20.20.245`) — evidencia de SNAT AutoMap.
- [ ] En la captura de la VLAN internal, la IP de destino es la de un pool member (`10.20.20.11` o `10.20.20.12`), no la VIP.
- [ ] Se observan handshakes TCP (SYN, SYN-ACK, ACK) independientes en cada VLAN, confirmando dos conexiones separadas.
- [ ] Las múltiples solicitudes se distribuyen entre ambos pool members (Round Robin).

---

### Paso 7: Verificación integral del estado de todos los objetos

**Objetivo:** Realizar una verificación completa de todos los componentes del camino de datos y reflexionar sobre cómo la pérdida de cualquier componente afectaría la disponibilidad del servicio.

**Instrucciones:**

1. **Verificar el estado de todas las interfaces:**

   ```bash
   show net interface
   ```

2. **Verificar el estado de todas las VLANs:**

   ```bash
   show net vlan
   ```

3. **Verificar el estado de todas las Self IPs:**

   ```bash
   list net self
   ```

4. **Verificar la tabla de rutas completa:**

   ```bash
   show net route
   ```

5. **Verificar el estado de los nodos (nodes):**

   ```bash
   show ltm node
   ```

6. **Verificar el estado del pool y sus miembros:**

   ```bash
   show ltm pool pool_web_lab
   show ltm pool pool_web_lab members
   ```

7. **Verificar el estado del Virtual Server:**

   ```bash
   show ltm virtual vs_web_lab
   ```

8. **Verificar las estadísticas del Virtual Server:**

   ```bash
   show ltm virtual vs_web_lab stat
   ```

9. **En TMUI**, navega a **Local Traffic > Network Map**. Esta vista muestra gráficamente la relación entre Virtual Servers, pools y miembros con sus estados de disponibilidad codificados por colores.

10. **Reflexión sobre puntos de fallo.** Completa mentalmente o por escrito la siguiente tabla de análisis de impacto:

    | Componente que falla | Impacto en el servicio |
    |---|---|
    | Interfaz 1.1 (down) | El tráfico de la VLAN external no puede alcanzar BIG-IP; el Virtual Server se vuelve inalcanzable para los clientes. |
    | Interfaz 1.2 (down) | BIG-IP no puede comunicarse con los servidores backend; el pool pierde todos sus miembros. |
    | Self IP external eliminada | BIG-IP pierde presencia en la VLAN external; no puede responder al tráfico del Virtual Server. |
    | Self IP internal eliminada | BIG-IP pierde presencia en la VLAN internal; los health monitors fallan y los miembros se marcan como offline. |
    | Ruta por defecto eliminada | En esta topología, impacto limitado porque cliente y servidores están en subredes directamente conectadas. En producción, los clientes remotos no podrían recibir respuestas. |
    | Un pool member falla | El monitor marca al miembro como no disponible; el tráfico se redirige al miembro restante. El servicio continúa con capacidad reducida. |
    | Ambos pool members fallan | El pool se marca como offline (rojo); el Virtual Server deja de responder solicitudes (o responde con un error si hay un perfil de fallback configurado). |

**Resultado esperado:**

Todos los objetos deben mostrar estado `available` y `enabled`:

```
Ltm::Virtual Server: vs_web_lab
  Availability : available
  State        : enabled

Ltm::Pool: pool_web_lab
  Availability : available
  Active Members : 2

Ltm::Node: 10.20.20.11
  Availability : available
Ltm::Node: 10.20.20.12
  Availability : available
```

Las estadísticas del Virtual Server deben reflejar las solicitudes procesadas durante el laboratorio.

**Verificación:**

- [ ] Todas las interfaces muestran estado `up`.
- [ ] Todas las VLANs están operativas.
- [ ] Todas las Self IPs están configuradas correctamente.
- [ ] Los nodos, pool members, pool y Virtual Server están en estado Available (verde).
- [ ] Las estadísticas muestran contadores de conexiones y bytes procesados mayor que cero.
- [ ] El Network Map en TMUI muestra la cadena completa VS → Pool → Members en verde.

## Validación y Pruebas

### Prueba 1: Conectividad extremo a extremo

Desde el cliente de pruebas, ejecuta:

```bash
curl -v http://10.10.10.100/
```

**Criterio de éxito:** La respuesta incluye código HTTP `200`, las cabeceras del servidor y el contenido HTML de uno de los servidores backend. La salida verbose muestra la conexión TCP establecida contra `10.10.10.100:80`.

### Prueba 2: Distribución de carga

```bash
for i in $(seq 1 10); do curl -s http://10.10.10.100/ 2>/dev/null; done | sort | uniq -c
```

**Criterio de éxito:** Ambos servidores reciben aproximadamente el mismo número de solicitudes (5 cada uno, con posible variación de ±1 debido al estado de las conexiones).

### Prueba 3: Resiliencia ante fallo de un pool member

1. Detén Apache en web02:

   ```bash
   ssh usuario@10.20.20.12 "sudo systemctl stop apache2"
   ```

2. Espera 35 segundos (timeout del monitor: 31 segundos + margen).

3. Verifica el estado del pool:

   ```bash
   show ltm pool pool_web_lab members
   ```

4. Genera tráfico:

   ```bash
   for i in $(seq 1 4); do curl -s http://10.10.10.100/; echo; done
   ```

**Criterio de éxito:** El miembro `10.20.20.12:80` aparece como `offline` o `unavailable` (rojo). Todas las solicitudes se dirigen a `10.20.20.11`. El servicio no se interrumpe.

5. **Restaura el servicio:**

   ```bash
   ssh usuario@10.20.20.12 "sudo systemctl start apache2"
   ```

6. Espera 15 segundos y verifica que el miembro vuelve a estado Available.

### Prueba 4: Caso adverso — solicitud a un puerto no configurado

Desde el cliente, intenta acceder a un puerto no configurado en el Virtual Server:

```bash
curl -v --connect-timeout 5 http://10.10.10.100:8080/
```

**Criterio de éxito:** La conexión es rechazada o expira (timeout). BIG-IP no tiene un Virtual Server escuchando en `10.10.10.100:8080`, por lo que no procesa el tráfico. Esto confirma que BIG-IP solo responde en las combinaciones IP:puerto explícitamente configuradas.

### Prueba 5: Caso adverso — verificación con información contradictoria

Ejecuta desde el cliente:

```bash
curl -s -H "Host: servidor-inexistente.example.com" http://10.10.10.100/
```

**Criterio de éxito:** BIG-IP procesa la solicitud normalmente y la reenvía al pool (el Virtual Server no tiene políticas basadas en Host header). El servidor backend responde con su contenido estándar. Esta prueba verifica que un Virtual Server de tipo Standard sin iRules ni políticas de enrutamiento por contenido no discrimina por cabecera Host: todo el tráfico que llega a la VIP:puerto se envía al default pool.

## Solución de Problemas

### Problema 1: El pool member se muestra como Offline (rojo) inmediatamente después de crearlo

**Síntomas:** Al verificar el pool en TMUI o con `show ltm pool pool_web_lab members`, uno o ambos miembros muestran estado `offline` con razón `No successful responses received before deadline`.

**Causa:** El health monitor HTTP no recibe la cadena esperada (`200 OK`) en la respuesta del servidor. Las causas más comunes son:

- El servicio Apache/NGINX no está activo en el servidor backend.
- El servidor responde con un código diferente (por ejemplo, `301 Moved Permanently` si hay redirección configurada).
- Un firewall del host bloquea el tráfico desde la Self IP de BIG-IP (`10.20.20.245`).
- La cadena `recv` del monitor no coincide exactamente con la respuesta del servidor.

**Resolución:**

1. Verifica que el servicio web está activo en el servidor:

   ```bash
   ssh usuario@10.20.20.11 "sudo systemctl status apache2"
   ```

2. Prueba la conectividad HTTP desde BIG-IP al servidor:

   ```bash
   run util curl -s -o /dev/null -w '%{http_code}' http://10.20.20.11/
   ```

   Si devuelve `301` en lugar de `200`, ajusta la configuración del servidor o modifica el monitor:

   ```bash
   modify ltm monitor http monitor_http_lab recv "HTTP/1."
   ```

3. Verifica que no hay reglas de firewall bloqueando en el servidor:

   ```bash
   ssh usuario@10.20.20.11 "sudo iptables -L -n | grep -i drop"
   ```

4. Tras corregir, espera un ciclo de monitor (10 segundos) y verifica nuevamente.

### Problema 2: El cliente puede hacer ping a la VIP pero curl no devuelve respuesta

**Síntomas:** Desde el cliente (`10.10.10.10`), `ping 10.10.10.100` es exitoso, pero `curl http://10.10.10.100/` se queda colgado o devuelve `Connection refused`.

**Causa:** El ping responde porque la dirección `10.10.10.100` está en la misma subred que la Self IP external y BIG-IP responde a ARP para esa dirección (es la VIP). Sin embargo, el Virtual Server puede no estar procesando tráfico TCP/80 correctamente. Las causas posibles son:

- El Virtual Server no tiene `ip-protocol tcp` configurado (por defecto usa `all`, lo cual puede causar comportamientos inesperados).
- Falta el perfil `tcp` en el Virtual Server.
- El SNAT AutoMap no está configurado y los servidores no tienen ruta de retorno hacia el cliente a través de BIG-IP.
- El pool está offline (ambos miembros fallaron).

**Resolución:**

1. Verifica la configuración completa del Virtual Server:

   ```bash
   list ltm virtual vs_web_lab
   ```

   Confirma que incluye `ip-protocol tcp`, `profiles { tcp http }` y `source-address-translation { type automap }`.

2. Si falta SNAT AutoMap, agrégalo:

   ```bash
   modify ltm virtual vs_web_lab source-address-translation { type automap }
   ```

3. Verifica el estado del pool:

   ```bash
   show ltm pool pool_web_lab
   ```

   Si el pool está offline, resuelve primero el Problema 1.

4. Verifica con tcpdump en la VLAN internal si el tráfico llega al servidor:

   ```bash
   tcpdump -nni internal -c 20 port 80
   ```

   Si no se ven paquetes, el problema está en la configuración del Virtual Server o en la selección del pool member.

## Limpieza

Si deseas revertir la configuración del laboratorio a su estado inicial (solo Management IP), ejecuta los siguientes comandos en tmsh en el **orden inverso** al de creación, respetando las dependencias de objetos:

```bash
## 1. Eliminar el Virtual Server
delete ltm virtual vs_web_lab

## 2. Eliminar el Pool
delete ltm pool pool_web_lab

## 3. Eliminar el Monitor personalizado
delete ltm monitor http monitor_http_lab

## 4. Eliminar los Nodos (creados automáticamente con el pool)
delete ltm node 10.20.20.11
delete ltm node 10.20.20.12

## 5. Eliminar la ruta por defecto
delete net route default_route

## 6. Eliminar las Self IPs
delete net self self_external
delete net self self_internal

## 7. Eliminar las VLANs
delete net vlan external
delete net vlan internal

## 8. Guardar la configuración limpia
save sys config
```

> **Importante:** En un entorno de curso donde los laboratorios posteriores dependen de esta configuración, **NO ejecutes la limpieza** a menos que el instructor lo indique. Los laboratorios siguientes del Módulo 2 asumen que las VLANs, Self IPs, rutas, pool y Virtual Server creados aquí permanecen configurados.

**Verificación posterior a la limpieza (si se ejecutó):**

```bash
list net vlan
list net self
list net route
list ltm virtual
list ltm pool
```

Todos los comandos deben devolver resultados vacíos o solo la configuración de fábrica.

## Resumen

En este laboratorio se construyó de forma incremental el camino de datos completo de un BIG-IP VE, partiendo de una configuración de fábrica hasta lograr un servicio HTTP balanceado y funcional. Los conceptos clave reforzados son:

- **Interfaces** son los puntos de conexión física/virtual de BIG-IP a la red. Su estado operativo es el fundamento de toda comunicación.
- **VLANs** segmentan el tráfico en dominios de broadcast independientes (external para clientes, internal para servidores), asociándose a interfaces específicas.
- **Self IPs** otorgan a BIG-IP una identidad de capa 3 en cada VLAN, habilitando la comunicación IP y los servicios de gestión en banda. El port lockdown controla qué servicios son accesibles a través de cada Self IP.
- **Rutas estáticas** definen cómo BIG-IP alcanza redes no directamente conectadas, siendo esenciales para el tráfico de retorno en topologías con múltiples subredes.
- **Virtual Servers** son el punto de entrada lógico de las aplicaciones. Combinados con pools, health monitors y SNAT, implementan la selección inteligente de servidores.
- **Full-proxy:** Las capturas con `tcpdump` demostraron que BIG-IP mantiene dos conexiones TCP independientes — una con el cliente y otra con el servidor — transformando las direcciones IP en cada lado. Esto es fundamental para comprender que BIG-IP no es un simple router, sino un proxy completo que puede inspeccionar, modificar y optimizar el tráfico en ambas direcciones.

### Relación con alta disponibilidad

En esta topología de laboratorio con un único BIG-IP VE, cada componente del camino de datos (interfaz, VLAN, Self IP, ruta, pool member, Virtual Server) representa un posible punto único de fallo. En un entorno de producción, una arquitectura Active/Standby con Device Service Clustering (DSC) mitigaría el fallo del propio BIG-IP, mientras que múltiples pool members mitigan el fallo de servidores individuales. La reflexión del Paso 7 conecta directamente con los conceptos de alta disponibilidad del tema 2.5.

### Recursos adicionales

| Recurso | URL |
|---|---|
| F5 BIG-IP LTM: Conceptos básicos de tráfico local | [https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-local-traffic-management.html](https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-local-traffic-management.html) |
| F5 BIG-IP: Configuración de VLANs y Self IPs | [https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-tmos-routing-administration.html](https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-tmos-routing-administration.html) |
| F5 BIG-IP: Referencia de tmsh | [https://clouddocs.f5.com/cli/tmsh-reference/latest/](https://clouddocs.f5.com/cli/tmsh-reference/latest/) |
| F5 Glossary: Full Proxy | [https://www.f5.com/glossary/full-proxy](https://www.f5.com/glossary/full-proxy) |
| F5 BIG-IP: Uso de tcpdump en TMOS | [https://my.f5.com/manage/s/article/K13637](https://my.f5.com/manage/s/article/K13637) |
