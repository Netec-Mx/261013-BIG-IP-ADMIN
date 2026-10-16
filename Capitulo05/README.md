# Laboratorio 8 — Diagnóstico de red y rendimiento en BIG-IP: estadísticas, interfaces, tcpdump, drops, TCP y utilización de recursos

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (Apply) |
| **Versión BIG-IP** | F5 BIG-IP VE 17.1.2 |
| **Alineación certificación** | F5CAB5.01, F5CAB5.02, F5CAB5.06, F5CAB5.07 |

## Descripción General

Este laboratorio integra las técnicas de diagnóstico de rendimiento y red en BIG-IP VE 17.1.2. A lo largo de seis fases secuenciales, se recopilan métricas de línea base (CPU, memoria, disco, interfaces), se provocan condiciones de falla controladas, se ejecutan capturas de paquetes con `tcpdump`, se diagnostican escenarios de degradación en Virtual Servers y Pools, y se genera un QKView para análisis en iHealth. Cada fase produce evidencia documentada que sustenta la causa raíz identificada, aplicando una metodología de troubleshooting estructurada de extremo a extremo.

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Consultar y correlacionar métricas de CPU, memoria, disco e interfaces del Control Plane y Data Plane de BIG-IP para identificar cuellos de botella y establecer una línea base operativa.
- [ ] Ejecutar capturas de paquetes filtradas con `tcpdump` en BIG-IP, transferirlas vía SCP y analizar en Wireshark retransmisiones TCP, resets y patrones anómalos del three-way handshake.
- [ ] Diagnosticar problemas de disponibilidad y rendimiento en Virtual Servers y Pool Members correlacionando estados configurados vs. disponibilidad, fallas de Health Monitors, connection limits y perfiles TCP.
- [ ] Generar un QKView, descargarlo y cargarlo en iHealth para obtener diagnósticos automatizados, documentando la evidencia que sustenta cada causa raíz identificada.
- [ ] Aplicar troubleshooting estructurado verificando estadísticas de interfaces (drops, errores, speed/duplex) y estadísticas de tráfico por objeto para clasificar problemas entre capacidad, red física o configuración.

## Prerrequisitos

### Conocimientos requeridos

| Área | Detalle |
|---|---|
| BIG-IP GUI y CLI | Navegación en Configuration Utility (TMUI) y acceso SSH con usuario `admin`; comandos básicos de `tmsh` (`show`, `list`, `modify`) |
| Redes TCP/IP | Modelo OSI capas 3-7, three-way handshake TCP, conceptos de MTU, speed/duplex |
| Laboratorios previos | Haber completado los Laboratorios 1-7 del curso con la topología persistente operativa |
| Herramientas de análisis | Uso básico de Wireshark para inspección de capturas `.pcap` |

### Acceso y herramientas requeridas

| Recurso | Versión exacta | Propósito |
|---|---|---|
| F5 BIG-IP VE | 17.1.2 ([Descarga MyF5](https://my.f5.com/manage/s/downloads)) | Plataforma de diagnóstico |
| Ubuntu Server (web01, web02) | 22.04.4 LTS ([Descarga Ubuntu](https://releases.ubuntu.com/22.04.4/)) | Servidores web del pool |
| Apache HTTP Server | 2.4.58 (incluido en Ubuntu 22.04.4 LTS) | Servicio HTTP en pool members |
| Apache Benchmark (`ab`) | 2.4.58 (paquete `apache2-utils`) | Generación de carga HTTP controlada |
| curl | 8.5.0 (incluido en Ubuntu 22.04.4 LTS) | Pruebas de conectividad HTTP |
| Wireshark | 4.2.3 ([Descarga Wireshark](https://www.wireshark.org/download.html)) | Análisis de capturas `.pcap` |
| Cliente SCP (WinSCP o `scp` CLI) | WinSCP 6.3.3 ([Descarga WinSCP](https://winscp.net/eng/download.php)) o `scp` nativo | Transferencia de archivos desde BIG-IP |
| Cuenta F5 iHealth | N/A ([Portal iHealth](https://ihealth.f5.com/)) | Análisis automatizado de QKView |
| Mozilla Firefox ESR | 128.0 ESR | Acceso a TMUI de BIG-IP |

## Entorno de Laboratorio

### Topología de red

```
┌──────────────┐        VLAN External         ┌──────────────┐        VLAN Internal        ┌──────────────┐
│   Cliente     │       10.10.10.0/24          │   BIG-IP VE  │       10.20.20.0/24         │   web01       │
│ 10.10.10.10   │◄────────────────────────────►│ ext: .245     │◄────────────────────────────►│ 10.20.20.11   │
│               │                              │ int: .245     │                              ├──────────────┤
│               │        Management            │ mgmt: .245    │                              │   web02       │
│               │       192.0.2.0/24           │               │                              │ 10.20.20.12   │
└──────────────┘                              └──────────────┘                              └──────────────┘
  Admin: 192.0.2.10 ◄─── mgmt ───► 192.0.2.245
```

### Direccionamiento IP

| Componente | Red | Dirección IP | Interfaz BIG-IP |
|---|---|---|---|
| Cliente (pruebas) | External | 10.10.10.10/24 | — |
| Cliente (admin) | Management | 192.0.2.10/24 | — |
| BIG-IP Management | Management | 192.0.2.245/24 | mgmt |
| BIG-IP External Self IP | External | 10.10.10.245/24 | 1.1 |
| BIG-IP Internal Self IP | Internal | 10.20.20.245/24 | 1.2 |
| Virtual Server HTTP | External | 10.10.10.100:80 | — |
| web01 | Internal | 10.20.20.11/24 | — |
| web02 | Internal | 10.20.20.12/24 | — |

### Estado esperado previo al laboratorio

Antes de iniciar, verifica que el entorno heredado de los Laboratorios 1-7 cumple estos requisitos mínimos:

```bash
## Desde el cliente de administración (192.0.2.10), conectar por SSH:
ssh admin@192.0.2.245

## En tmsh, verificar Virtual Server HTTP activo:
tmsh show ltm virtual vs_http

## Verificar Pool con 2 members disponibles:
tmsh show ltm pool pool_http

## Verificar Health Monitor HTTP aplicado:
tmsh list ltm pool pool_http monitor

## Verificar perfiles TCP y HTTP aplicados al Virtual Server:
tmsh list ltm virtual vs_http profiles
```

**Resultado esperado:** El Virtual Server `vs_http` debe mostrar estado `available` (verde), el pool `pool_http` debe tener 2 members activos (`10.20.20.11:80` y `10.20.20.12:80`) y los perfiles `tcp` y `http` deben estar asociados.

> **Nota:** Si los nombres de objetos difieren de los indicados (por ejemplo, `vs_web_http` en lugar de `vs_http`), sustituye consistentemente a lo largo del laboratorio. Los nombres utilizados aquí son representativos.

### Instalación de Apache Benchmark en el cliente

Si `ab` no está instalado en la máquina cliente (10.10.10.10):

```bash
sudo apt update && sudo apt install -y apache2-utils
ab -V
## Salida esperada: This is ApacheBench, Version 2.3 <$Revision: ...>
```

## Instrucciones Paso a Paso

---

### Paso 1: Recopilación de línea base de recursos del sistema (Fase 1 — Baseline)

**Objective:** Capturar métricas de CPU, memoria, disco e interfaces en condiciones normales de operación para establecer una referencia de comparación durante el diagnóstico posterior.

**Instructions:**

1. Conéctate por SSH al BIG-IP desde el cliente de administración:

```bash
ssh admin@192.0.2.245
```

2. Ejecuta los siguientes comandos en `tmsh` y registra la salida de cada uno en tu plantilla de documentación. Copia cada salida completa:

```bash
## 2a. Rendimiento general del sistema
tmsh show sys performance all-stats

## 2b. Utilización de CPU — detalle por núcleo
tmsh show sys cpu

## 2c. Estado de memoria del sistema
tmsh show sys memory

## 2d. Utilización de disco por partición
tmsh show sys disk

## 2e. Estadísticas de todas las interfaces de red
tmsh show net interface all
```

3. Desde la GUI (TMUI), navega a **Statistics > Performance > System** y captura una imagen de la gráfica de CPU y memoria de los últimos 30 minutos. Registra la hora exacta de la captura.

4. Navega a **Statistics > Module Statistics > Local Traffic** y selecciona **Virtual Servers** en el menú desplegable. Registra las estadísticas actuales del Virtual Server `vs_http`: conexiones actuales, total de conexiones, bytes in/out.

5. Accede a la shell de Bash para obtener estadísticas detalladas de TMM:

```bash
## Desde tmsh, cambiar a bash:
bash

## Estadísticas de TMM
tmctl -d blade tmm/global_host_cpu_usage

## Estadísticas de interfaces a nivel de kernel
netstat -i

## Regresar a tmsh
exit
```

6. Genera una prueba de conectividad básica desde el cliente de pruebas (10.10.10.10) para confirmar el funcionamiento normal:

```bash
## Desde el cliente (10.10.10.10):
curl -s -o /dev/null -w "HTTP Code: %{http_code} | Time Total: %{time_total}s\n" http://10.10.10.100
```

7. Ejecuta una carga moderada con `ab` para observar el comportamiento base:

```bash
## Desde el cliente (10.10.10.10):
ab -n 500 -c 10 http://10.10.10.100/
```

8. Inmediatamente después de que `ab` finalice, vuelve a ejecutar en BIG-IP:

```bash
tmsh show sys cpu
tmsh show ltm virtual vs_http
tmsh show ltm pool pool_http members
```

**Expected output:**

- `tmsh show sys cpu`: Porcentajes de utilización por núcleo. En condiciones de línea base con carga moderada, los valores típicos de TMM en un laboratorio estarán por debajo del 30%.
- `tmsh show sys memory`: Memoria total asignada (TMM Memory Used, Other Memory Used, Memory Available). No debe haber indicadores de swap activo significativo.
- `tmsh show sys disk`: Particiones `/`, `/var`, `/shared` con porcentaje de uso. Ninguna partición debe superar el 85%.
- `tmsh show net interface all`: Interfaces `mgmt`, `1.1` y `1.2` con estado `up`, sin errores ni drops.
- `curl`: HTTP Code 200, tiempo total inferior a 1 segundo.
- `ab`: Requests per second, tiempo medio por solicitud y porcentaje de solicitudes exitosas (debe ser 100%).

**Verification:**

Confirma que dispones de los siguientes datos documentados para tu línea base:

```bash
## Verificar que no hay errores ni drops en las interfaces:
tmsh show net interface 1.1 | grep -i -E "error|drop"
tmsh show net interface 1.2 | grep -i -E "error|drop"
## Resultado esperado: Counters en 0 o valores estables bajos
```

> **Documentación requerida:** Registra en tu plantilla: (1) Fecha y hora, (2) CPU por núcleo, (3) Memoria total/usada/disponible, (4) Disco por partición, (5) Estado y contadores de interfaces, (6) Estadísticas del Virtual Server, (7) Resultado de `ab`.

---

### Paso 2: Análisis de interfaces y detección de drops (Fase 2 — Interfaces)

**Objective:** Examinar estadísticas detalladas de interfaces para identificar errores, drops y validar la configuración de speed/duplex. Introducir una discrepancia controlada para observar su impacto.

**Instructions:**

1. Desde `tmsh`, obtén estadísticas detalladas de cada interfaz:

```bash
tmsh show net interface 1.1 detail
tmsh show net interface 1.2 detail
```

2. Documenta los siguientes campos para cada interfaz:

| Campo | Interfaz 1.1 | Interfaz 1.2 |
|---|---|---|
| Status | | |
| Speed (actual) | | |
| Duplex (actual) | | |
| Packets In | | |
| Packets Out | | |
| Bytes In | | |
| Bytes Out | | |
| Errors In | | |
| Errors Out | | |
| Drops In | | |
| Drops Out | | |
| Collisions | | |

3. Verifica la configuración actual de media (speed/duplex) de la interfaz 1.1:

```bash
tmsh list net interface 1.1
```

4. **Introducción de discrepancia controlada:** Modifica la configuración de la interfaz 1.1 para forzar un media type fijo que difiera de la negociación automática. Este cambio provocará una condición observable:

```bash
## Guardar configuración actual para restauración:
tmsh list net interface 1.1 media-fixed > /var/tmp/if11_backup.txt

## Forzar half-duplex en la interfaz 1.1:
tmsh modify net interface 1.1 media-fixed 100baseTX-HD
```

> **Advertencia:** Este cambio puede causar degradación temporal de conectividad en la VLAN External. Es un cambio deliberado para el ejercicio.

5. Espera 10 segundos y verifica el estado resultante:

```bash
tmsh show net interface 1.1 detail
```

6. Genera tráfico desde el cliente para observar el impacto:

```bash
## Desde el cliente (10.10.10.10):
ab -n 200 -c 5 http://10.10.10.100/
```

7. Inmediatamente después, revisa las estadísticas de la interfaz:

```bash
tmsh show net interface 1.1 detail | grep -i -E "status|speed|duplex|error|drop|collision"
```

8. **Restaura la configuración original** de la interfaz:

```bash
tmsh modify net interface 1.1 media-fixed auto
```

9. Confirma la restauración:

```bash
tmsh show net interface 1.1 detail | grep -i -E "status|speed|duplex"
```

**Expected output:**

- Tras forzar half-duplex, es posible observar: incremento de colisiones, errores de salida, drops o reducción de throughput en la salida de `ab` (mayor tiempo por solicitud, posibles solicitudes fallidas).
- En entornos virtualizados (VE), el efecto puede variar: algunos hypervisores no aplican la restricción de duplex a nivel virtual. Si no se observan errores, documenta que el entorno virtual no reproduce fielmente la discrepancia de duplex y explica por qué en hardware físico el impacto sería significativo (colisiones late, CRC errors, throughput degradado).
- Tras restaurar `auto`, el estado debe volver a `up` con speed y duplex negociados correctamente.

**Verification:**

```bash
## Confirmar que la interfaz 1.1 está operativa y sin errores acumulados nuevos:
tmsh show net interface 1.1 detail
## Verificar conectividad restaurada:
## Desde el cliente (10.10.10.10):
curl -s -o /dev/null -w "HTTP Code: %{http_code}\n" http://10.10.10.100
## Resultado esperado: HTTP Code: 200
```

> **Documentación requerida:** Registra la tabla comparativa de contadores antes y después de la discrepancia de duplex. Indica si se observaron errores, drops o colisiones y correlaciona con el resultado de `ab`.

---

### Paso 3: Captura de paquetes con tcpdump y análisis en Wireshark (Fase 3 — Packet Capture)

**Objective:** Ejecutar capturas de paquetes filtradas con `tcpdump` en BIG-IP, generar tráfico controlado, transferir la captura y analizar en Wireshark el three-way handshake, retransmisiones y resets TCP.

**Instructions:**

1. Desde la sesión SSH al BIG-IP, cambia a la shell de Bash:

```bash
bash
```

2. Inicia una captura filtrada por la IP del Virtual Server HTTP, guardando en archivo `.pcap`:

```bash
tcpdump -ni 0.0 -s0 -c 500 -w /var/tmp/lab8_vs_http.pcap host 10.10.10.100 &
```

> **Nota:** El flag `-ni 0.0` captura en todas las interfaces lógicas de TMM (client-side y server-side). `-s0` captura el paquete completo. `-c 500` limita a 500 paquetes. El `&` ejecuta en segundo plano.

3. Anota el PID del proceso `tcpdump`:

```bash
echo "PID de tcpdump: $!"
```

4. Desde el **cliente de pruebas** (10.10.10.10), genera tráfico HTTP:

```bash
## Solicitudes individuales para observar handshakes claros:
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code} in %{time_total}s\n" http://10.10.10.100/
  sleep 0.5
done

## Carga concurrente para generar volumen:
ab -n 200 -c 20 http://10.10.10.100/
```

5. De vuelta en BIG-IP, espera a que `tcpdump` finalice (500 paquetes) o detenlo manualmente:

```bash
## Si aún está ejecutándose:
kill %1 2>/dev/null
## Verificar que el archivo existe y tiene contenido:
ls -lh /var/tmp/lab8_vs_http.pcap
```

6. **Captura adicional para provocar resets TCP:** Inicia una segunda captura y genera solicitudes hacia un puerto cerrado o un pool member que no responde:

```bash
tcpdump -ni 0.0 -s0 -c 100 -w /var/tmp/lab8_resets.pcap host 10.10.10.100 &
```

7. Desde el cliente, intenta una conexión que genere un reset (por ejemplo, si temporalmente deshabilitamos un member):

```bash
## En BIG-IP tmsh (abrir otra sesión SSH):
tmsh modify ltm node 10.20.20.12 session user-disabled
```

8. Genera tráfico desde el cliente:

```bash
## Desde el cliente (10.10.10.10):
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" http://10.10.10.100/
done
```

9. Detén la captura y rehabilita el nodo:

```bash
## En BIG-IP bash:
kill %1 2>/dev/null

## Rehabilitar el nodo:
exit  # volver a tmsh
tmsh modify ltm node 10.20.20.12 session user-enabled
```

10. Verifica los archivos de captura:

```bash
bash
ls -lh /var/tmp/lab8_*.pcap
```

11. **Transfiere las capturas** al cliente de administración usando SCP. Desde la estación de trabajo (192.0.2.10):

```bash
scp admin@192.0.2.245:/var/tmp/lab8_vs_http.pcap ./
scp admin@192.0.2.245:/var/tmp/lab8_resets.pcap ./
```

12. **Abre `lab8_vs_http.pcap` en Wireshark 4.2.3** y realiza los siguientes análisis:

    a. **Three-way handshake:** Aplica el filtro `tcp.flags.syn==1` y localiza pares SYN / SYN-ACK. Verifica que cada conexión completa el handshake (SYN → SYN-ACK → ACK).

    b. **Retransmisiones TCP:** Aplica el filtro `tcp.analysis.retransmission`. Documenta cuántas retransmisiones se observan y entre qué endpoints ocurren.

    c. **Flujo client-side vs. server-side:** En la captura de interfaz `0.0`, observa que aparecen dos flujos por cada conexión HTTP: uno entre el cliente (10.10.10.10) y el VIP (10.10.10.100), y otro entre el Self IP interno (10.20.20.245) y el pool member (10.20.20.11 o 10.20.20.12).

13. **Abre `lab8_resets.pcap` en Wireshark** y aplica el filtro `tcp.flags.reset==1`. Documenta:
    - ¿Qué endpoint envía el RST?
    - ¿En qué momento del flujo TCP ocurre (después del SYN, después de datos, etc.)?
    - ¿Correlaciona con el member deshabilitado?

**Expected output:**

- `lab8_vs_http.pcap`: Archivo de 50 KB–2 MB con ~500 paquetes. En Wireshark se observan handshakes completos, solicitudes HTTP GET y respuestas 200 OK.
- `lab8_resets.pcap`: Archivo con paquetes que incluyen RST flags. Cuando un pool member está deshabilitado y solo queda uno activo, todo el tráfico se dirige al member restante. Si ambos estuvieran down, se observarían resets del BIG-IP al cliente.
- Las retransmisiones en condiciones normales de laboratorio deben ser mínimas (0-2).

**Verification:**

```bash
## En BIG-IP, confirmar que ambos members están activos nuevamente:
tmsh show ltm pool pool_http members

## Desde el cliente, confirmar servicio operativo:
curl -s -o /dev/null -w "HTTP Code: %{http_code}\n" http://10.10.10.100
## Resultado esperado: HTTP Code: 200
```

> **Documentación requerida:** Registra: (1) Capturas de pantalla de Wireshark mostrando el three-way handshake, (2) Número de retransmisiones encontradas, (3) Análisis de RST flags con endpoint origen y momento del flujo, (4) Observación de los dos flujos (client-side y server-side) en la captura de interfaz `0.0`.

---

### Paso 4: Troubleshooting de Virtual Servers y Pools — escenarios de falla (Fase 4)

**Objective:** Diagnosticar tres escenarios de falla simulados en Virtual Servers y Pools utilizando `tmsh` y la GUI, correlacionando configured state, availability state, Health Monitors, connection limits y perfiles TCP.

**Instructions:**

#### Escenario A: Health Monitor con configuración incorrecta

1. Modifica el Health Monitor HTTP del pool para que espere una cadena de respuesta que el servidor no devuelve:

```bash
tmsh modify ltm monitor http http_monitor send "GET /nonexistent_health_page HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n" recv "EXPECTED_STRING_THAT_DOES_NOT_EXIST"
```

> **Nota:** Si tu monitor tiene un nombre diferente, sustitúyelo. Verifica el nombre actual con `tmsh list ltm pool pool_http monitor`.

2. Espera 30 segundos (o el intervalo + timeout del monitor) y verifica el estado:

```bash
tmsh show ltm pool pool_http
tmsh show ltm pool pool_http members
tmsh show ltm virtual vs_http
```

3. Documenta:
   - ¿Cuál es el `Availability` del pool y de cada member?
   - ¿Cuál es el estado del Virtual Server?
   - ¿Qué indica el campo `Monitor` de cada member?

4. Prueba desde el cliente:

```bash
## Desde el cliente (10.10.10.10):
curl -v http://10.10.10.100/
```

5. Documenta el comportamiento observado (timeout, reset, error específico).

6. **Corrección:** Restaura el monitor a una configuración funcional:

```bash
tmsh modify ltm monitor http http_monitor send "GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n" recv ""
```

7. Espera 30 segundos y verifica la recuperación:

```bash
tmsh show ltm pool pool_http members
```

#### Escenario B: Connection limit alcanzado en un pool member

8. Configura un límite de conexiones extremadamente bajo en un member:

```bash
tmsh modify ltm pool pool_http members modify { 10.20.20.11:80 { connection-limit 1 } }
```

9. Genera carga concurrente desde el cliente:

```bash
## Desde el cliente (10.10.10.10):
ab -n 100 -c 20 http://10.10.10.100/
```

10. Durante o inmediatamente después de la carga, verifica las estadísticas:

```bash
tmsh show ltm pool pool_http members
tmsh show ltm pool pool_http members field-fmt | grep -A5 "10.20.20.11"
```

11. Documenta:
    - ¿Cuántas conexiones actuales tiene cada member?
    - ¿El member con `connection-limit 1` muestra estado diferente?
    - ¿Se observa distribución asimétrica del tráfico?

12. **Corrección:** Elimina el límite de conexiones:

```bash
tmsh modify ltm pool pool_http members modify { 10.20.20.11:80 { connection-limit 0 } }
```

#### Escenario C: Virtual Server con perfil TCP incompatible

13. Crea un perfil TCP personalizado con un valor de `idle-timeout` extremadamente bajo para simular desconexiones prematuras:

```bash
tmsh create ltm profile tcp tcp_lab8_broken idle-timeout 1 defaults-from tcp
```

14. Aplica el perfil al Virtual Server:

```bash
tmsh modify ltm virtual vs_http profiles delete { tcp }
tmsh modify ltm virtual vs_http profiles add { tcp_lab8_broken { context all } }
```

15. Genera solicitudes desde el cliente con una pausa deliberada:

```bash
## Desde el cliente (10.10.10.10):
## Solicitud con tiempo de respuesta esperado normal:
curl -v --max-time 10 http://10.10.10.100/

## Solicitud que simula un cliente lento (espera 3 segundos antes de enviar):
(sleep 3 && echo -e "GET / HTTP/1.1\r\nHost: 10.10.10.100\r\nConnection: close\r\n\r\n") | nc -w 5 10.10.10.100 80
```

16. Verifica las estadísticas del Virtual Server:

```bash
tmsh show ltm virtual vs_http
```

17. Documenta si se observan resets o conexiones cerradas prematuramente.

18. **Corrección:** Restaura el perfil TCP original:

```bash
tmsh modify ltm virtual vs_http profiles delete { tcp_lab8_broken }
tmsh modify ltm virtual vs_http profiles add { tcp { context all } }
tmsh delete ltm profile tcp tcp_lab8_broken
```

**Expected output:**

- **Escenario A:** Pool members marcados como `offline` (círculo rojo) o `unknown`. El Virtual Server pasa a estado `offline` o `unavailable`. El cliente recibe un reset de conexión o un error de timeout.
- **Escenario B:** El member con `connection-limit 1` alcanza su límite rápidamente; el tráfico se redistribuye al otro member. Las estadísticas muestran distribución desigual.
- **Escenario C:** Con `idle-timeout 1`, conexiones que tarden más de 1 segundo en completarse son terminadas por BIG-IP. Se observan resets o cierres de conexión en solicitudes lentas.

**Verification:**

```bash
## Confirmar que todos los escenarios están corregidos:
tmsh show ltm virtual vs_http
## Availability: available (verde)

tmsh show ltm pool pool_http members
## Ambos members: available, connection-limit 0

tmsh list ltm virtual vs_http profiles
## Debe mostrar perfil tcp (no tcp_lab8_broken)

## Prueba funcional final:
## Desde el cliente (10.10.10.10):
ab -n 50 -c 5 http://10.10.10.100/
## 100% de solicitudes exitosas
```

> **Documentación requerida:** Para cada escenario registra: (1) Comando de inyección de falla, (2) Salida de `tmsh show` mostrando el estado degradado, (3) Comportamiento observado desde el cliente, (4) Comando de corrección, (5) Confirmación de recuperación.

---

### Paso 5: Estadísticas de tráfico y análisis de flujo (Fase 5)

**Objective:** Revisar estadísticas de tráfico por objeto (Virtual Server, Pool, Node) en la GUI y CLI, y verificar la configuración de SNMP para monitoreo externo.

**Instructions:**

1. Genera una carga sostenida de 60 segundos desde el cliente:

```bash
## Desde el cliente (10.10.10.10):
ab -n 2000 -c 30 -t 60 http://10.10.10.100/
```

2. **Durante la ejecución de `ab`**, accede a la GUI de BIG-IP:
   - Navega a **Statistics > Module Statistics > Local Traffic**.
   - Selecciona **Virtual Servers** en el menú desplegable `Statistics Type`.
   - Observa y documenta para `vs_http`: Bits In, Bits Out, Packets In, Packets Out, Current Connections, Total Connections.

3. Cambia la vista a **Pools** y documenta las mismas métricas para `pool_http`.

4. Cambia a **Nodes** y verifica la distribución de tráfico entre `10.20.20.11` y `10.20.20.12`.

5. Navega a **Statistics > Performance > System** y observa las gráficas de CPU y throughput durante el periodo de carga. Captura una imagen.

6. Desde `tmsh`, obtén estadísticas en tiempo real:

```bash
## Estadísticas del Virtual Server:
tmsh show ltm virtual vs_http raw

## Estadísticas del Pool:
tmsh show ltm pool pool_http all-properties

## Estadísticas por Node:
tmsh show ltm node 10.20.20.11
tmsh show ltm node 10.20.20.12
```

7. **Verificación de SNMP:** Consulta la configuración actual de SNMP:

```bash
tmsh list sys snmp
tmsh list sys snmp communities
tmsh list sys snmp allowed-addresses
```

8. Si no existe una comunidad configurada, crea una de solo lectura para verificación:

```bash
tmsh modify sys snmp communities add { lab8_community { community-name lab8ro access ro } }
tmsh modify sys snmp allowed-addresses add { 192.0.2.0/24 }
```

9. Verifica la configuración SNMP resultante:

```bash
tmsh list sys snmp communities
```

10. Elimina la comunidad de prueba al finalizar:

```bash
tmsh modify sys snmp communities delete { lab8_community }
```

**Expected output:**

- Las estadísticas del Virtual Server muestran incremento proporcional de conexiones y bytes durante la carga de `ab`.
- La distribución entre los dos pool members debe ser aproximadamente equitativa (Round Robin o el método de balanceo configurado).
- Las gráficas de Performance muestran un pico de CPU y throughput coincidente con el periodo de `ab`.
- La configuración SNMP muestra las comunidades y direcciones permitidas.

**Verification:**

```bash
## Confirmar que las estadísticas del Virtual Server reflejan tráfico procesado:
tmsh show ltm virtual vs_http | grep -i "total"
## Debe mostrar Total Requests > 0

## Confirmar distribución entre members:
tmsh show ltm pool pool_http members | grep -A2 "10.20.20"
## Ambos members deben mostrar Total Requests > 0
```

> **Documentación requerida:** Registra: (1) Tabla de estadísticas por objeto, (2) Captura de pantalla de gráfica de Performance durante la carga, (3) Distribución porcentual de tráfico entre members, (4) Configuración SNMP verificada.

---

### Paso 6: Generación de QKView y análisis en iHealth (Fase 6)

**Objective:** Generar un QKView del sistema, descargarlo, cargarlo en F5 iHealth y revisar los diagnósticos automatizados para documentar la evidencia consolidada.

**Instructions:**

1. Desde la shell de Bash en BIG-IP, genera el QKView:

```bash
bash
qkview -f /var/tmp/lab8_qkview.tar.gz
```

> **Nota:** La generación puede tardar 2-5 minutos dependiendo de los recursos del laboratorio. No interrumpas el proceso.

2. Verifica que el archivo se generó correctamente:

```bash
ls -lh /var/tmp/lab8_qkview.tar.gz
## Tamaño esperado: 10-50 MB típicamente
```

3. Transfiere el QKView al cliente de administración:

```bash
## Desde la estación de trabajo (192.0.2.10):
scp admin@192.0.2.245:/var/tmp/lab8_qkview.tar.gz ./
```

4. Accede a [F5 iHealth](https://ihealth.f5.com/) con tu cuenta MyF5 (o las credenciales de demostración provistas por el instructor).

5. Sube el archivo `lab8_qkview.tar.gz`:
   - Haz clic en **Upload** o **New QKView**.
   - Selecciona el archivo y espera a que se procese (1-5 minutos).

6. Una vez procesado, revisa las siguientes secciones del informe de iHealth:

   a. **Diagnostics:** Identifica advertencias y errores. Documenta cada hallazgo con su severidad (Critical, High, Medium, Low, Informational).

   b. **Software:** Verifica la versión detectada (debe ser 17.1.2) y si iHealth reporta vulnerabilidades conocidas o actualizaciones recomendadas.

   c. **Hardware/Platform:** Confirma que iHealth identifica correctamente la plataforma como Virtual Edition.

   d. **Configuration:** Revisa si hay recomendaciones de configuración (por ejemplo, NTP no configurado, DNS faltante, etc.).

7. Documenta al menos **tres hallazgos** del informe de iHealth en tu plantilla con el siguiente formato:

| # | Severidad | Categoría | Descripción | Recomendación de iHealth | Acción propuesta |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |

8. Si no dispones de acceso a iHealth, como alternativa documenta el contenido del QKView localmente:

```bash
## En BIG-IP bash, listar el contenido del QKView:
tar tzf /var/tmp/lab8_qkview.tar.gz | head -30

## Revisar el archivo de diagnóstico incluido:
tar xzf /var/tmp/lab8_qkview.tar.gz -C /var/tmp/qkview_extract/ 2>/dev/null
ls /var/tmp/qkview_extract/
```

**Expected output:**

- QKView generado exitosamente con tamaño entre 10-50 MB.
- iHealth procesa el archivo y muestra un dashboard con diagnósticos categorizados.
- En un entorno de laboratorio, es común encontrar advertencias de severidad baja o media relacionadas con: NTP no configurado, licencia de evaluación, configuraciones por defecto no optimizadas.

**Verification:**

```bash
## Confirmar que el QKView existe y es transferible:
ls -lh /var/tmp/lab8_qkview.tar.gz
## Tamaño > 5 MB indica generación exitosa

## Verificar integridad del archivo:
file /var/tmp/lab8_qkview.tar.gz
## Resultado esperado: gzip compressed data
```

> **Documentación requerida:** Registra: (1) Tamaño y fecha del QKView, (2) Captura de pantalla del dashboard de iHealth (o listado de contenido si no hay acceso), (3) Tabla con mínimo 3 hallazgos documentados, (4) Versión de BIG-IP confirmada por iHealth.

---

## Validación y Pruebas

Completa las siguientes verificaciones para confirmar que el laboratorio se ha ejecutado correctamente y que la evidencia es suficiente:

### Verificación funcional completa

```bash
## 1. Confirmar que el entorno está restaurado a su estado operativo:
tmsh show ltm virtual vs_http
## Availability: available, State: enabled

tmsh show ltm pool pool_http members
## Ambos members: available, State: enabled, connection-limit: 0

## 2. Verificar que no quedan artefactos de los escenarios de falla:
tmsh list ltm profile tcp tcp_lab8_broken 2>&1
## Resultado esperado: "was not found" (el perfil fue eliminado)

## 3. Confirmar conectividad end-to-end:
## Desde el cliente (10.10.10.10):
curl -s -o /dev/null -w "HTTP: %{http_code} | Total: %{time_total}s\n" http://10.10.10.100
## HTTP: 200 | Total: < 1s

## 4. Verificar que las interfaces están limpias:
tmsh show net interface 1.1 | grep -i "media-active"
## Debe mostrar auto-negotiated, no half-duplex

## 5. Confirmar que los archivos de captura existen:
bash
ls -lh /var/tmp/lab8_vs_http.pcap /var/tmp/lab8_resets.pcap /var/tmp/lab8_qkview.tar.gz
exit
```

### Verificación de evidencia documentada

Confirma que tu plantilla de troubleshooting contiene:

| # | Evidencia | Fase | ¿Completa? |
|---|---|---|---|
| 1 | Línea base de CPU, memoria, disco, interfaces | Fase 1 | ☐ |
| 2 | Tabla comparativa de contadores de interfaz (antes/después de discrepancia duplex) | Fase 2 | ☐ |
| 3 | Capturas de pantalla de Wireshark: three-way handshake, retransmisiones, resets | Fase 3 | ☐ |
| 4 | Diagnóstico documentado de 3 escenarios de falla (A, B, C) con causa y corrección | Fase 4 | ☐ |
| 5 | Estadísticas de tráfico por objeto y distribución entre members | Fase 5 | ☐ |
| 6 | Hallazgos de iHealth (mínimo 3) con severidad y acción propuesta | Fase 6 | ☐ |

### Caso adversarial: validación de información contradictoria

Para verificar el rigor del diagnóstico, responde la siguiente pregunta en tu documentación:

> **Pregunta adversarial:** Un compañero afirma que el Virtual Server `vs_http` tiene un problema de rendimiento porque la CPU de TMM está al 95%. Sin embargo, al revisar `tmsh show sys cpu`, observas que la CPU promedio es del 12% y ningún núcleo individual supera el 20%. El compañero insiste porque vio el valor "95" en una pantalla de la GUI. ¿Qué explicación darías y qué evidencia adicional solicitarías antes de aceptar ese diagnóstico?

**Respuesta esperada (criterios de evaluación):**
- Identificar que el valor "95" podría corresponder a una métrica diferente (por ejemplo, porcentaje de utilización de un recurso específico, un pico histórico, o una lectura de un periodo de muestreo diferente).
- Solicitar la captura de pantalla exacta con la ruta de navegación en la GUI.
- Correlacionar con `tmsh show sys cpu` y `tmsh show sys performance` para obtener datos objetivos.
- No aceptar un diagnóstico basado en una sola métrica sin contexto temporal ni correlación con síntomas observables.

## Solución de Problemas

### Problema 1: tcpdump no captura paquetes del Virtual Server

**Síntomas:** Se ejecuta `tcpdump -ni 0.0 host 10.10.10.100` pero no aparecen paquetes, a pesar de que el cliente está generando tráfico y recibiendo respuestas HTTP 200.

**Causa:** El comando `tcpdump` se ejecutó desde `tmsh` en lugar de desde la shell de `bash`. En `tmsh`, `tcpdump` no está disponible como comando directo, o se ejecutó con la interfaz incorrecta (por ejemplo, `eth0` en lugar de `0.0`). La interfaz especial `0.0` en BIG-IP captura tráfico en todas las interfaces lógicas de TMM y solo está disponible desde la shell de Bash. Si se usa `eth0` o `mgmt`, solo se captura tráfico de la red de gestión.

**Solución:**

```bash
## 1. Confirmar que estás en bash (no en tmsh):
whoami
## Debe mostrar: root (después de sudo bash) o admin

## 2. Si estás en tmsh, cambiar a bash:
bash

## 3. Ejecutar tcpdump con la interfaz correcta:
tcpdump -ni 0.0 -s0 -c 10 host 10.10.10.100

## 4. Si necesitas ejecutar como root:
sudo tcpdump -ni 0.0 -s0 -c 10 host 10.10.10.100

## 5. Verificar que hay tráfico activo generándose desde el cliente:
## Desde otra terminal en el cliente (10.10.10.10):
curl http://10.10.10.100/
```

### Problema 2: Pool members marcados como down después de restaurar el Health Monitor

**Síntomas:** Después de ejecutar el Escenario A (Paso 4) y restaurar el monitor HTTP con `recv ""`, los pool members permanecen en estado `offline` (rojo) durante más de 60 segundos. El Virtual Server sigue no disponible.

**Causa:** El nombre del monitor modificado no coincide con el monitor realmente asignado al pool. Es posible que el pool utilice un monitor con nombre diferente (por ejemplo, `http` predeterminado del sistema vs. `http_monitor` personalizado). Al modificar un monitor que no es el asignado, el monitor real del pool puede haber quedado con una configuración incompatible, o el pool puede tener un monitor adicional que está fallando.

**Solución:**

```bash
## 1. Identificar el monitor exacto asignado al pool:
tmsh list ltm pool pool_http monitor
## Ejemplo de salida: monitor http

## 2. Si el monitor asignado es "http" (predeterminado del sistema), NO se puede
## modificar directamente. Verificar si fue modificado por error:
tmsh list ltm monitor http http

## 3. Si el monitor predeterminado fue alterado, restaurar los valores por defecto:
tmsh modify ltm monitor http http send "GET /\r\n" recv ""

## 4. Si el pool usa un monitor personalizado, verificar su configuración:
tmsh list ltm monitor http http_monitor

## 5. Forzar una reevaluación reiniciando el monitor en el pool:
tmsh modify ltm pool pool_http monitor none
tmsh modify ltm pool pool_http monitor http

## 6. Esperar el intervalo del monitor (por defecto 5 segundos) + timeout (16 segundos)
## y verificar:
sleep 20
tmsh show ltm pool pool_http members
## Ambos members deben mostrar: State: up, Monitor: up
```

## Limpieza

Ejecuta los siguientes comandos para restaurar el entorno a su estado previo al laboratorio:

```bash
## 1. Eliminar archivos de captura y QKView generados:
bash
rm -f /var/tmp/lab8_vs_http.pcap
rm -f /var/tmp/lab8_resets.pcap
rm -f /var/tmp/lab8_qkview.tar.gz
rm -rf /var/tmp/qkview_extract/
rm -f /var/tmp/if11_backup.txt
exit

## 2. Confirmar que no quedan perfiles de prueba:
tmsh list ltm profile tcp tcp_lab8_broken 2>&1
## Debe indicar "not found"

## 3. Confirmar que la interfaz 1.1 está en auto-negotiate:
tmsh list net interface 1.1 media-fixed
## Debe mostrar: media-fixed auto

## 4. Confirmar que connection-limit está en 0 para ambos members:
tmsh show ltm pool pool_http members | grep "connection-limit"
## Ambos deben mostrar: 0

## 5. Eliminar la comunidad SNMP de prueba si no fue eliminada:
tmsh modify sys snmp communities delete { lab8_community } 2>/dev/null

## 6. Confirmar que el Health Monitor está funcional:
tmsh show ltm pool pool_http members
## Ambos members: available

## 7. Guardar la configuración limpia:
tmsh save sys config

## 8. Verificación final end-to-end:
## Desde el cliente (10.10.10.10):
curl -s -o /dev/null -w "HTTP: %{http_code}\n" http://10.10.10.100
## HTTP: 200
```

## Resumen

En este laboratorio se aplicó una metodología de diagnóstico de rendimiento y red en BIG-IP VE 17.1.2 a través de seis fases progresivas:

1. **Línea base de recursos:** Se recopilaron métricas de CPU, memoria, disco e interfaces como referencia para comparación durante incidentes, diferenciando indicadores del Control Plane (procesos administrativos) y Data Plane (TMM).

2. **Análisis de interfaces:** Se examinaron contadores de errores, drops, speed y duplex, comprobando que una discrepancia de duplex produce degradación observable en las estadísticas.

3. **Captura de paquetes:** Se ejecutaron capturas filtradas con `tcpdump` en la interfaz `0.0`, se identificaron los flujos client-side y server-side, y se analizaron en Wireshark handshakes TCP, retransmisiones y resets.

4. **Escenarios de falla:** Se diagnosticaron tres condiciones controladas — monitor con recv incorrecto, connection limit alcanzado y perfil TCP con idle-timeout agresivo — correlacionando configured state vs. availability state.

5. **Estadísticas de tráfico:** Se verificó la distribución de carga entre pool members y se revisó la configuración de SNMP para monitoreo externo.

6. **QKView e iHealth:** Se generó un snapshot de diagnóstico del sistema y se analizaron los hallazgos automatizados para documentar evidencia consolidada.

**Principio clave:** Una métrica aislada no constituye un diagnóstico. La correlación entre utilización de recursos, estadísticas de interfaces, capturas de paquetes y estados de objetos es lo que permite identificar y sustentar la causa raíz de un problema de rendimiento.

### Recursos adicionales

| Recurso | URL |
|---|---|
| F5 BIG-IP TMOS Operations Guide | [https://techdocs.f5.com/en-us/bigip-17-1-0.html](https://techdocs.f5.com/en-us/bigip-17-1-0.html) |
| Referencia de comandos TMSH | [https://clouddocs.f5.com/cli/tmsh-reference/latest/](https://clouddocs.f5.com/cli/tmsh-reference/latest/) |
| K12878: Generating a QKView diagnostic file | [https://my.f5.com/manage/s/article/K12878](https://my.f5.com/manage/s/article/K12878) |
| K411: Overview of packet tracing with tcpdump | [https://my.f5.com/manage/s/article/K411](https://my.f5.com/manage/s/article/K411) |
| F5 iHealth | [https://ihealth.f5.com/](https://ihealth.f5.com/) |
| Wireshark 4.2.3 User Guide | [https://www.wireshark.org/docs/wsug_html/](https://www.wireshark.org/docs/wsug_html/) |

---

# Laboratorio 9 — Troubleshooting integral de una aplicación: Virtual Server, Pool, Health Monitor, Persistence, Profiles, Priority Groups, límites y análisis del tráfico de extremo a extremo

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (Apply) |
| **Versión BIG-IP** | F5 BIG-IP VE 17.1.2 |
| **Alineación curricular** | F5CAB5.03–F5CAB5.07; integra F5CAB1–F5CAB4 |

## Descripción General

Este laboratorio es el ejercicio de integración final del curso. El estudiante recibe un entorno BIG-IP preconfigurado con una aplicación web HTTPS publicada a través de un Virtual Server, un Pool con Priority Groups, Cookie Persistence, Health Monitor HTTP y Connection Limits. El entorno ha sido deliberadamente saboteado con **seis fallas inyectadas simultáneamente** que el estudiante debe descubrir, diagnosticar y corregir aplicando la metodología sistemática de cinco fases: **Identificar → Aislar → Diagnosticar → Corregir → Verificar**. Al finalizar, el estudiante genera un QKView, lo sube a iHealth y completa una hoja de registro de troubleshooting con evidencias medibles para cada falla.

## Objetivos de Aprendizaje

- [ ] Aplicar la metodología de troubleshooting de cinco fases (Identificar → Aislar → Diagnosticar → Corregir → Verificar) para resolver seis fallas inyectadas simultáneamente en un BIG-IP, documentando síntomas, causas raíz y acciones correctivas con evidencia medible.
- [ ] Diagnosticar y corregir fallas en Virtual Server, Pool, Health Monitor, Persistence y Profiles utilizando `tmsh`, TMUI, estadísticas de objetos y logs del sistema (`/var/log/ltm`).
- [ ] Ejecutar y analizar capturas de paquetes (`tcpdump`) en BIG-IP para inspeccionar el flujo cliente→VIP→Pool Member, identificando anomalías TCP/HTTP y correlacionándolas con las fallas detectadas.
- [ ] Evaluar la utilización de recursos del sistema (CPU, memoria, disco, interfaces) antes y después de las correcciones para confirmar el impacto de las fallas en el rendimiento, comparando contra una línea base operativa.
- [ ] Generar un QKView, subirlo a F5 iHealth e interpretar al menos dos diagnósticos automatizados relacionados con las fallas corregidas.

## Prerrequisitos

### Conocimiento requerido

| Área | Detalle |
|---|---|
| Laboratorios previos | Haber completado los laboratorios 1–8 (módulos F5CAB1–F5CAB4) o poseer conocimiento equivalente |
| BIG-IP LTM | Creación y administración de Virtual Servers, Pools, Health Monitors, Persistence Profiles, SSL Profiles |
| Comandos `tmsh` | `show ltm virtual`, `show ltm pool`, `show ltm monitor`, `show ltm persistence persist-records`, `show sys performance`, `show sys memory`, `show sys disk`, `show net interface` |
| TCP/IP | Three-way handshake, HTTP/HTTPS, cookies HTTP, direccionamiento IP, VLANs |
| BIG-IP conceptos | Diferencia entre *configured state* y *availability state*, Priority Group Activation threshold, Connection Limits |
| Herramientas | `curl` 8.5.0, `tcpdump`, Wireshark 4.2.5, Apache `ab` 2.4.58 |

### Acceso requerido

| Recurso | Credenciales / Método |
|---|---|
| TMUI BIG-IP | `https://192.0.2.245` — usuario `admin` |
| SSH BIG-IP | `ssh root@192.0.2.245` |
| Cliente de pruebas | `10.10.10.10` — acceso SSH y navegador Firefox ESR 128.0 |
| Cliente de administración | `192.0.2.10` — conectividad a red Management y VLAN External |
| Cuenta F5 iHealth | Cuenta registrada en `https://ihealth.f5.com` ([enlace oficial](https://ihealth.f5.com)) |
| Servidores web | `web01` (10.20.20.11) y `web02` (10.20.20.12) — acceso SSH para validación |

## Entorno de Laboratorio

### Topología de red

```
┌──────────────┐         VLAN External          ┌──────────────────┐         VLAN Internal         ┌─────────────┐
│   Cliente     │        10.10.10.0/24           │    BIG-IP VE     │        10.20.20.0/24          │   web01      │
│ 10.10.10.10   │◄──────────────────────────────►│ Ext: 10.10.10.245│◄────────────────────────────►│ 10.20.20.11  │
│               │                                │ Int: 10.20.20.245│                               └─────────────┘
│               │                                │ Mgmt:192.0.2.245 │                               ┌─────────────┐
└──────────────┘                                 └──────────────────┘◄────────────────────────────►│   web02      │
                                                                                                    │ 10.20.20.12  │
  Admin: 192.0.2.10 ──── Red Management 192.0.2.0/24 ──── BIG-IP Mgmt: 192.0.2.245                └─────────────┘
```

### Configuración objetivo (estado correcto antes de la inyección de fallas)

| Objeto | Configuración esperada |
|---|---|
| **Virtual Server** `vs_app_https` | Dirección: `10.10.10.100:443`, tipo Standard, protocolo TCP, Client SSL Profile: `clientssl_app`, HTTP Profile: `http_app`, Default Pool: `pool_app`, estado Enabled |
| **Pool** `pool_app` | Método LB: Round Robin, Priority Group Activation: 1, Monitor: `mon_http_app` |
| **Pool Member** `10.20.20.11:80` | Priority Group: 10, Connection Limit: 0 (sin límite), estado Enabled |
| **Pool Member** `10.20.20.12:80` | Priority Group: 10, Connection Limit: 0 (sin límite), estado Enabled |
| **Health Monitor** `mon_http_app` | Tipo HTTP, Interval: 5, Timeout: 16, Send: `GET / HTTP/1.1\r\nHost: localhost\r\n\r\n`, Receive: `200 OK` |
| **Persistence Profile** `persist_cookie_app` | Tipo Cookie, método Insert, Cookie Name: `BIGipServer_pool_app`, Expiration: 0 (sesión) |
| **Client SSL Profile** `clientssl_app` | Certificado y clave autofirmados, protocolo TLS 1.2+, asociado al VS |
| **HTTP Profile** `http_app` | Valores por defecto, asociado al VS |

### Las seis fallas inyectadas (referencia para el instructor)

> **NOTA PARA EL INSTRUCTOR:** Las siguientes fallas deben inyectarse **antes** de que el estudiante acceda al entorno. El estudiante **no** conoce esta lista; debe descubrir cada falla mediante la metodología de troubleshooting.

| # | Componente | Falla inyectada | Efecto observable |
|---|---|---|---|
| F1 | Virtual Server | Estado administrativo cambiado a **Disabled** | VS no acepta conexiones; Network Map muestra icono negro |
| F2 | Virtual Server | HTTP Profile **removido** del VS (incompatibilidad: VS HTTPS sin HTTP Profile impide procesamiento L7) | Sin inspección HTTP, cookie persistence no puede operar |
| F3 | Health Monitor | Receive String cambiada a `"404 Not Found"` (no coincide con respuesta real del servidor que devuelve `200 OK`) | Todos los Pool Members marcados como *Offline* (rojo) |
| F4 | Pool | Priority Group Activation threshold cambiado a **99** (inalcanzable con 2 members en PG-10) | Incluso con members disponibles, el pool no activa tráfico correctamente |
| F5 | Pool Members | Connection Limit de `10.20.20.11:80` establecido en **1** | Después de 1 conexión concurrente, el member rechaza nuevas conexiones; distribución de carga desequilibrada |
| F6 | Persistence | Cookie Name cambiado a una cadena vacía `""` | BIG-IP no puede insertar cookie de persistence; las sesiones no se mantienen |

### Comandos de inyección de fallas (solo instructor)

```bash
## F1: Deshabilitar Virtual Server
tmsh modify ltm virtual vs_app_https disabled

## F2: Eliminar HTTP Profile del Virtual Server
tmsh modify ltm virtual vs_app_https profiles delete { http_app }

## F3: Cambiar Receive String del Health Monitor
tmsh modify ltm monitor http mon_http_app recv "404 Not Found"

## F4: Cambiar Priority Group Activation threshold
tmsh modify ltm pool pool_app min-active-members 99

## F5: Establecer Connection Limit en Pool Member
tmsh modify ltm pool pool_app members { 10.20.20.11:80 { connection-limit 1 } }

## F6: Vaciar Cookie Name en Persistence Profile
tmsh modify ltm persistence cookie persist_cookie_app cookie-name ""

## Guardar configuración
tmsh save sys config
```

### Verificación previa del entorno (instructor)

```bash
## Confirmar que los servidores web responden
curl -s -o /dev/null -w "%{http_code}" http://10.20.20.11/
## Esperado: 200

curl -s -o /dev/null -w "%{http_code}" http://10.20.20.12/
## Esperado: 200

## Confirmar que BIG-IP es accesible
tmsh show sys version
## Esperado: BIG-IP 17.1.2
```

## Instrucciones Paso a Paso

---

### Paso 1: Registrar línea base de recursos del sistema

**Objetivo:** Capturar el estado de utilización de CPU, memoria, disco e interfaces del BIG-IP **antes** de iniciar las correcciones, para poder comparar después del restablecimiento del servicio.

**Instrucciones:**

1. Desde el cliente de administración (`192.0.2.10`), inicie sesión SSH en el BIG-IP:

   ```bash
   ssh root@192.0.2.245
   ```

2. Capture el estado de rendimiento general del sistema:

   ```bash
   tmsh show sys performance all-stats
   ```

3. Registre el uso de memoria:

   ```bash
   tmsh show sys memory
   ```

4. Registre la ocupación de disco:

   ```bash
   tmsh show sys disk logical-disk
   ```

5. Registre el estado y estadísticas de las interfaces de datos:

   ```bash
   tmsh show net interface 1.1
   tmsh show net interface 1.2
   ```

6. Registre las estadísticas actuales del Virtual Server (incluso si está inoperante):

   ```bash
   tmsh show ltm virtual vs_app_https
   tmsh show ltm virtual vs_app_https stats
   ```

7. Guarde toda la salida en un archivo de referencia:

   ```bash
   tmsh show sys performance all-stats > /var/tmp/baseline_pre.txt
   tmsh show sys memory >> /var/tmp/baseline_pre.txt
   tmsh show sys disk logical-disk >> /var/tmp/baseline_pre.txt
   tmsh show net interface all-properties >> /var/tmp/baseline_pre.txt
   tmsh show ltm virtual vs_app_https >> /var/tmp/baseline_pre.txt
   echo "--- Baseline capturada: $(date) ---" >> /var/tmp/baseline_pre.txt
   ```

**Salida esperada:**

- `show sys performance all-stats`: muestra porcentajes de CPU para System, TMM y otros procesos. En estado de reposo con fallas inyectadas, TMM CPU debería ser bajo (<5%) ya que no hay tráfico procesándose.
- `show sys memory`: muestra TMM Memory Used y Other Memory Used. Valores típicos de laboratorio: <40% utilización total.
- `show sys disk logical-disk`: particiones con al menos 20% de espacio libre.
- `show net interface`: interfaces 1.1 y 1.2 con estado `up`, sin errores ni drops significativos.
- `show ltm virtual`: el VS aparecerá con `Availability: offline` y `State: disabled`.

**Verificación:**

- [ ] El archivo `/var/tmp/baseline_pre.txt` existe y contiene datos de los cinco comandos.
- [ ] Se identifican los valores de CPU TMM, memoria utilizada, espacio en disco disponible y estado de interfaces.

---

### Paso 2: Identificar síntomas — prueba de conectividad y Network Map (FASE 1)

**Objetivo:** Documentar el estado de disponibilidad de cada objeto BIG-IP y registrar el error que percibe el cliente al intentar acceder a la aplicación.

**Instrucciones:**

1. Desde el cliente de pruebas (`10.10.10.10`), intente acceder a la aplicación:

   ```bash
   curl -vk https://10.10.10.100/ 2>&1 | tee /tmp/curl_initial.txt
   ```

2. Registre el error obtenido. Con el VS deshabilitado, espere un rechazo de conexión o timeout.

3. Acceda al TMUI desde el cliente de administración:

   ```
   https://192.0.2.245
   ```

4. Navegue a **Local Traffic → Network Map** y documente el estado de cada objeto:

   | Objeto | Estado observado (color/icono) | Disponibilidad |
   |---|---|---|
   | `vs_app_https` | | |
   | `pool_app` | | |
   | `10.20.20.11:80` | | |
   | `10.20.20.12:80` | | |

5. Desde SSH al BIG-IP, confirme los estados con `tmsh`:

   ```bash
   tmsh show ltm virtual vs_app_https
   tmsh show ltm pool pool_app
   tmsh show ltm pool pool_app members
   ```

6. Revise los logs recientes para eventos relevantes:

   ```bash
   tail -100 /var/log/ltm | grep -iE "monitor|pool|virtual|error|down"
   ```

**Salida esperada:**

- `curl`: error de tipo `Connection refused` o `Connection reset by peer` (el VS está deshabilitado).
- Network Map: `vs_app_https` con icono negro (Disabled), `pool_app` con icono rojo (Offline — todos los members marcados como no disponibles por el monitor).
- `show ltm pool pool_app members`: ambos members con `State: enabled`, `Availability: offline` (monitor falla).
- Logs en `/var/log/ltm`: mensajes indicando que el monitor `mon_http_app` marca members como down.

**Verificación:**

- [ ] Se documenta el error exacto del `curl` (código de error o mensaje).
- [ ] Se registra el estado de cada objeto en Network Map.
- [ ] Se identifican al menos dos síntomas distintos: VS deshabilitado y Pool Members offline.

---

### Paso 3: Aislar y diagnosticar fallas — inspección jerárquica de componentes (FASE 2)

**Objetivo:** Recorrer la jerarquía de troubleshooting (Virtual Server → Pool → Pool Members → Health Monitor → Persistence → Profiles) para identificar las seis fallas inyectadas.

**Instrucciones:**

#### 3a. Inspección del Virtual Server

1. Liste la configuración completa del Virtual Server:

   ```bash
   tmsh list ltm virtual vs_app_https
   ```

2. Verifique los siguientes elementos y anote discrepancias:

   | Atributo | Valor esperado | Valor observado | ¿Discrepancia? |
   |---|---|---|---|
   | Destination | `10.10.10.100:443` | | |
   | State | `enabled` | | |
   | Pool | `pool_app` | | |
   | Profiles: `clientssl_app` | Presente | | |
   | Profiles: `http_app` | Presente | | |
   | IP Protocol | `tcp` | | |

3. Identifique **Falla F1**: el estado administrativo es `disabled`.

4. Identifique **Falla F2**: el HTTP Profile `http_app` no aparece en la lista de profiles. Sin HTTP Profile, el VS no puede procesar tráfico a nivel L7, lo que impide que Cookie Persistence funcione.

   ```bash
   tmsh list ltm virtual vs_app_https profiles
   ```

#### 3b. Inspección del Pool y Priority Groups

5. Liste la configuración del Pool:

   ```bash
   tmsh list ltm pool pool_app
   ```

6. Verifique:

   | Atributo | Valor esperado | Valor observado | ¿Discrepancia? |
   |---|---|---|---|
   | Load Balancing Mode | `round-robin` | | |
   | Min Active Members | `1` | | |
   | Monitor | `mon_http_app` | | |

7. Identifique **Falla F4**: `min-active-members` es `99` en lugar de `1`. Con solo 2 members en PG-10, el threshold de 99 es inalcanzable, lo que provoca un comportamiento anómalo en la activación del Priority Group.

#### 3c. Inspección de Pool Members y Connection Limits

8. Liste los members con detalle:

   ```bash
   tmsh list ltm pool pool_app members
   tmsh show ltm pool pool_app members
   ```

9. Verifique Connection Limits:

   | Member | Connection Limit esperado | Connection Limit observado | ¿Discrepancia? |
   |---|---|---|---|
   | `10.20.20.11:80` | `0` (sin límite) | | |
   | `10.20.20.12:80` | `0` (sin límite) | | |

10. Identifique **Falla F5**: `10.20.20.11:80` tiene `connection-limit 1`.

#### 3d. Inspección del Health Monitor

11. Liste la configuración del monitor:

    ```bash
    tmsh list ltm monitor http mon_http_app
    ```

12. Verifique:

    | Atributo | Valor esperado | Valor observado | ¿Discrepancia? |
    |---|---|---|---|
    | Interval | `5` | | |
    | Timeout | `16` | | |
    | Send String | `GET / HTTP/1.1\r\nHost: localhost\r\n\r\n` | | |
    | Receive String | `200 OK` | | |

13. Confirme qué responde realmente el backend:

    ```bash
    curl -s -o /dev/null -w "%{http_code}" http://10.20.20.11/
    curl -s -o /dev/null -w "%{http_code}" http://10.20.20.12/
    ```

    Ambos deben devolver `200`.

14. Identifique **Falla F3**: la Receive String es `"404 Not Found"` pero el servidor responde `200 OK`. El monitor nunca encuentra coincidencia y marca los members como offline.

#### 3e. Inspección de Persistence

15. Liste la configuración del Persistence Profile:

    ```bash
    tmsh list ltm persistence cookie persist_cookie_app
    ```

16. Verifique:

    | Atributo | Valor esperado | Valor observado | ¿Discrepancia? |
    |---|---|---|---|
    | Method | `insert` | | |
    | Cookie Name | `BIGipServer_pool_app` | | |
    | Expiration | `0` | | |

17. Identifique **Falla F6**: el `cookie-name` está vacío (`""`), lo que impide que BIG-IP inserte correctamente la cookie de persistence.

#### 3f. Resumen de fallas identificadas

18. Complete la tabla de hallazgos:

    | # | Componente | Falla identificada | Impacto |
    |---|---|---|---|
    | F1 | Virtual Server | Estado `disabled` | VS no acepta tráfico |
    | F2 | Virtual Server / Profiles | HTTP Profile ausente | Sin procesamiento L7; persistence inoperante |
    | F3 | Health Monitor | Receive String incorrecta (`404 Not Found`) | Todos los members marcados offline |
    | F4 | Pool | `min-active-members` = 99 | Priority Group Activation inalcanzable |
    | F5 | Pool Member | Connection Limit = 1 en `10.20.20.11:80` | Distribución de carga desequilibrada |
    | F6 | Persistence | Cookie Name vacío | Sessions no se mantienen |

**Salida esperada:**

Seis fallas documentadas con su componente, descripción y efecto observable.

**Verificación:**

- [ ] Se identificaron las 6 fallas antes de proceder a las correcciones.
- [ ] Para cada falla se registró: el comando usado para detectarla, el valor incorrecto y el valor esperado.

---

### Paso 4: Capturar tráfico para evidencia pre-corrección (FASE 3)

**Objetivo:** Ejecutar una captura de paquetes en BIG-IP para documentar el comportamiento anómalo del tráfico antes de aplicar correcciones.

**Instrucciones:**

1. Desde SSH al BIG-IP, inicie una captura en la interfaz virtual `0.0` (captura todo el tráfico procesado por TMM):

   ```bash
   tcpdump -ni 0.0 -s 0 -w /var/tmp/capture_pre_fix.pcap host 10.10.10.100 &
   TCPDUMP_PID=$!
   echo "tcpdump PID: $TCPDUMP_PID"
   ```

2. Desde el cliente de pruebas, genere tráfico durante 30 segundos:

   ```bash
   # Intentar conexión HTTPS (fallará porque VS está disabled)
   curl -vk --connect-timeout 5 https://10.10.10.100/ 2>&1 | tee /tmp/curl_pre_fix.txt

   # Intentar múltiples conexiones
   for i in $(seq 1 5); do
     curl -sk --connect-timeout 3 https://10.10.10.100/ 2>&1 | tail -1
   done
   ```

3. Detenga la captura en BIG-IP:

   ```bash
   kill $TCPDUMP_PID
   ls -la /var/tmp/capture_pre_fix.pcap
   ```

4. Descargue el archivo desde el cliente de administración:

   ```bash
   scp root@192.0.2.245:/var/tmp/capture_pre_fix.pcap /tmp/
   ```

5. Abra el archivo en Wireshark 4.2.5 y aplique los siguientes filtros de display:

   ```
   tcp.flags.reset == 1
   tcp.analysis.retransmission
   ```

6. Documente las observaciones:
   - ¿Se observan paquetes RST?
   - ¿Hay intentos de handshake TCP que no se completan?
   - ¿Hay evidencia de tráfico hacia los Pool Members?

**Salida esperada:**

- Con el VS deshabilitado, se esperan RST inmediatos o ausencia total de respuesta en la VIP `10.10.10.100:443`.
- No debería haber tráfico hacia `10.20.20.11` ni `10.20.20.12` en la captura.
- El archivo `.pcap` tendrá pocos paquetes, mayormente SYN del cliente y RST del BIG-IP (o ningún paquete si el VS no responde en absoluto).

**Verificación:**

- [ ] El archivo `capture_pre_fix.pcap` existe y tiene tamaño > 0 bytes.
- [ ] Se documentaron las anomalías observadas en Wireshark (RST, retransmisiones, ausencia de respuesta).

---

### Paso 5: Aplicar correcciones en orden de dependencia (FASE 4)

**Objetivo:** Corregir las seis fallas en el orden correcto para evitar dependencias rotas: primero el Health Monitor (para que los members vuelvan a estar disponibles), luego el Pool (Priority Groups y Connection Limits), después los Profiles del Virtual Server, luego Persistence, y finalmente habilitar el Virtual Server.

**Instrucciones:**

#### 5a. Corregir Falla F3 — Health Monitor Receive String

1. Corrija la Receive String:

   ```bash
   tmsh modify ltm monitor http mon_http_app recv "200 OK"
   ```

2. Espere 16 segundos (el timeout del monitor) y verifique que los members vuelvan a estar disponibles:

   ```bash
   sleep 20
   tmsh show ltm pool pool_app members
   ```

3. **Resultado esperado:** Ambos members deben mostrar `Availability: available` (icono verde en TMUI).

#### 5b. Corregir Falla F4 — Priority Group Activation threshold

4. Corrija el threshold:

   ```bash
   tmsh modify ltm pool pool_app min-active-members 1
   ```

5. Verifique:

   ```bash
   tmsh list ltm pool pool_app min-active-members
   ```

6. **Resultado esperado:** `min-active-members 1`.

#### 5c. Corregir Falla F5 — Connection Limit del Pool Member

7. Elimine el límite de conexiones:

   ```bash
   tmsh modify ltm pool pool_app members { 10.20.20.11:80 { connection-limit 0 } }
   ```

8. Verifique:

   ```bash
   tmsh list ltm pool pool_app members { 10.20.20.11:80 }
   ```

9. **Resultado esperado:** `connection-limit 0`.

#### 5d. Corregir Falla F2 — Restaurar HTTP Profile en el Virtual Server

10. Agregue el HTTP Profile al Virtual Server:

    ```bash
    tmsh modify ltm virtual vs_app_https profiles add { http_app { context all } }
    ```

11. Verifique que ambos profiles (clientssl y http) estén presentes:

    ```bash
    tmsh list ltm virtual vs_app_https profiles
    ```

12. **Resultado esperado:** Se listan `clientssl_app` y `http_app`.

#### 5e. Corregir Falla F6 — Cookie Name de Persistence

13. Restaure el Cookie Name:

    ```bash
    tmsh modify ltm persistence cookie persist_cookie_app cookie-name "BIGipServer_pool_app"
    ```

14. Verifique:

    ```bash
    tmsh list ltm persistence cookie persist_cookie_app cookie-name
    ```

15. **Resultado esperado:** `cookie-name BIGipServer_pool_app`.

#### 5f. Corregir Falla F1 — Habilitar el Virtual Server

16. Habilite el Virtual Server (esta corrección se aplica al final para que todas las dependencias estén resueltas):

    ```bash
    tmsh modify ltm virtual vs_app_https enabled
    ```

17. Verifique el estado completo:

    ```bash
    tmsh show ltm virtual vs_app_https
    ```

18. **Resultado esperado:** `Availability: available`, `State: enabled`.

#### 5g. Guardar la configuración

19. Persista los cambios:

    ```bash
    tmsh save sys config
    ```

**Salida esperada general después de todas las correcciones:**

```
Ltm::Virtual Server: vs_app_https
  Availability     : available
  State            : enabled
  Reason           : The virtual server is available
```

```
Ltm::Pool: pool_app
  Availability     : available
  State            : enabled
  Current Members  : 2
  Active Members   : 2
```

**Verificación:**

- [ ] Cada corrección fue verificada individualmente con el comando `tmsh` correspondiente antes de pasar a la siguiente.
- [ ] El Virtual Server muestra `available` y `enabled`.
- [ ] Ambos Pool Members muestran `available` y `enabled`.

---

### Paso 6: Verificar el restablecimiento del servicio de extremo a extremo

**Objetivo:** Confirmar que la aplicación es accesible desde el cliente, que la cookie de persistence se inserta correctamente y que la distribución de carga funciona entre ambos Pool Members.

**Instrucciones:**

1. Desde el cliente de pruebas, ejecute una solicitud HTTPS y examine los headers:

   ```bash
   curl -vk https://10.10.10.100/ 2>&1 | tee /tmp/curl_post_fix.txt
   ```

2. Verifique que la respuesta sea HTTP 200 OK:

   ```bash
   curl -sk -o /dev/null -w "%{http_code}\n" https://10.10.10.100/
   ```

   **Resultado esperado:** `200`

3. Verifique que la cookie de persistence esté presente en la respuesta:

   ```bash
   curl -sk -D - https://10.10.10.100/ | grep -i "Set-Cookie"
   ```

   **Resultado esperado:** Una línea que contenga `Set-Cookie: BIGipServer_pool_app=...`

4. Verifique la tabla de persistence:

   ```bash
   tmsh show ltm persistence persist-records
   ```

   **Resultado esperado:** Al menos un registro de persistence asociado al Pool `pool_app`.

5. Genere carga con `ab` para verificar distribución de tráfico entre ambos members:

   ```bash
   ab -n 100 -c 10 -k https://10.10.10.100/
   ```

   > **Nota:** Si `ab` no soporta HTTPS directamente en su versión, use como alternativa:
   > ```bash
   > for i in $(seq 1 100); do curl -sk https://10.10.10.100/ > /dev/null; done
   > ```

6. Verifique las estadísticas de distribución en el Pool:

   ```bash
   tmsh show ltm pool pool_app members stats
   ```

7. Confirme que ambos members recibieron conexiones. Con Round Robin y sin persistence en las peticiones de `ab` (cada petición es una nueva conexión sin cookie), la distribución debería ser aproximadamente equitativa.

8. Revise las estadísticas del Virtual Server post-corrección:

   ```bash
   tmsh show ltm virtual vs_app_https stats
   ```

   **Resultado esperado:** Contadores de `Total Requests`, `Bits In`, `Bits Out` y `Current Connections` con valores > 0.

**Verificación:**

- [ ] `curl` devuelve HTTP 200 OK.
- [ ] La cookie `BIGipServer_pool_app` aparece en el header `Set-Cookie`.
- [ ] La tabla de persistence contiene al menos un registro.
- [ ] Ambos Pool Members muestran conexiones en sus estadísticas.
- [ ] Las estadísticas del Virtual Server muestran tráfico procesado.

---

### Paso 7: Capturar tráfico post-corrección y analizar en Wireshark

**Objetivo:** Ejecutar una nueva captura de paquetes para contrastar con la captura pre-corrección y confirmar que el flujo cliente→VIP→Pool Member es correcto.

**Instrucciones:**

1. Inicie la captura en BIG-IP:

   ```bash
   tcpdump -ni 0.0 -s 0 -w /var/tmp/capture_post_fix.pcap host 10.10.10.100 &
   TCPDUMP_PID=$!
   ```

2. Genere tráfico desde el cliente:

   ```bash
   curl -vk https://10.10.10.100/ 2>&1 > /dev/null
   curl -vk https://10.10.10.100/ 2>&1 > /dev/null
   curl -vk https://10.10.10.100/ 2>&1 > /dev/null
   ```

3. Detenga la captura:

   ```bash
   kill $TCPDUMP_PID
   ```

4. Descargue y abra en Wireshark 4.2.5:

   ```bash
   scp root@192.0.2.245:/var/tmp/capture_post_fix.pcap /tmp/
   ```

5. En Wireshark, aplique los siguientes filtros y documente los hallazgos:

   | Filtro de display | Qué buscar | Resultado esperado |
   |---|---|---|
   | `tcp.flags.reset == 1` | RST inesperados | Ninguno o mínimos (solo cierres normales) |
   | `ssl.handshake` | TLS handshake completo | Client Hello → Server Hello → Certificate → ... → Finished |
   | `http.cookie contains "BIGipServer"` | Cookie de persistence | Presente en requests del cliente |
   | `ip.dst == 10.20.20.11 or ip.dst == 10.20.20.12` | Tráfico hacia backends | Paquetes hacia ambos servers |

6. Compare con la captura pre-corrección:
   - Pre-corrección: RST o ausencia de respuesta.
   - Post-corrección: handshake TLS completo, HTTP 200, cookies de persistence.

**Salida esperada:**

En la captura post-corrección se observa:
- TCP three-way handshake exitoso entre cliente y VIP.
- TLS handshake completo.
- Tráfico HTTP entre BIG-IP y los Pool Members en la VLAN interna.
- Cookie `BIGipServer_pool_app` en los headers HTTP.

**Verificación:**

- [ ] La captura post-corrección muestra handshakes TLS completos (sin RST prematuros).
- [ ] Se identifica tráfico hacia ambos Pool Members (`10.20.20.11` y `10.20.20.12`).
- [ ] La cookie de persistence es visible en el análisis HTTP de Wireshark.

---

### Paso 8: Registrar línea base post-corrección y comparar recursos

**Objetivo:** Capturar métricas de utilización de recursos después de las correcciones y compararlas con la línea base del Paso 1 para confirmar que las fallas no dejaron impacto residual.

**Instrucciones:**

1. Capture las métricas post-corrección:

   ```bash
   tmsh show sys performance all-stats > /var/tmp/baseline_post.txt
   tmsh show sys memory >> /var/tmp/baseline_post.txt
   tmsh show sys disk logical-disk >> /var/tmp/baseline_post.txt
   tmsh show net interface all-properties >> /var/tmp/baseline_post.txt
   tmsh show ltm virtual vs_app_https stats >> /var/tmp/baseline_post.txt
   echo "--- Post-corrección: $(date) ---" >> /var/tmp/baseline_post.txt
   ```

2. Compare las métricas clave:

   ```bash
   echo "=== CPU PRE ===" && grep -i "tmm" /var/tmp/baseline_pre.txt | head -5
   echo "=== CPU POST ===" && grep -i "tmm" /var/tmp/baseline_post.txt | head -5
   ```

3. Verifique el estado de las interfaces:

   ```bash
   tmsh show net interface 1.1
   tmsh show net interface 1.2
   ```

4. Confirme que no hay errores ni drops nuevos en las interfaces:

   | Interfaz | Errors In (pre) | Errors In (post) | Drops (pre) | Drops (post) |
   |---|---|---|---|---|
   | 1.1 | | | | |
   | 1.2 | | | | |

5. Documente la comparación:
   - TMM CPU: ¿incremento esperado después de procesar tráfico? (Sí, es normal un leve incremento.)
   - Memoria: ¿cambio significativo? (No debería haber cambio relevante en un entorno de laboratorio.)
   - Disco: ¿se generaron archivos de captura que ocupen espacio? (Sí, los `.pcap` y la futura QKView.)

**Salida esperada:**

Las métricas post-corrección deben mostrar:
- TMM CPU con un leve incremento respecto al estado pre-corrección (ahora procesa tráfico activamente).
- Memoria sin cambios significativos.
- Disco con los archivos de captura adicionales pero sin presión de espacio.
- Interfaces sin errores nuevos.

**Verificación:**

- [ ] Los archivos `baseline_pre.txt` y `baseline_post.txt` existen en `/var/tmp/`.
- [ ] Se documentó la comparación de al menos CPU TMM, memoria y estado de interfaces.
- [ ] No se detectaron errores ni drops anómalos en las interfaces.

---

### Paso 9: Generar QKView y subir a iHealth (FASE 5)

**Objetivo:** Generar un archivo QKView del estado actual del BIG-IP y subirlo a F5 iHealth para obtener diagnósticos automatizados.

**Instrucciones:**

1. Genere el QKView desde SSH:

   ```bash
   qkview -f /var/tmp/lab9.qkview
   ```

   > **Nota:** La generación puede tomar 2–5 minutos. No interrumpa el proceso.

2. Verifique que el archivo se generó correctamente:

   ```bash
   ls -lh /var/tmp/lab9.qkview
   ```

   **Resultado esperado:** Un archivo de entre 30 MB y 200 MB.

3. Descargue el archivo al cliente de administración:

   ```bash
   scp root@192.0.2.245:/var/tmp/lab9.qkview /tmp/
   ```

4. Acceda a F5 iHealth desde el navegador:

   ```
   https://ihealth.f5.com
   ```

5. Inicie sesión con su cuenta F5 registrada.

6. Suba el archivo `lab9.qkview` mediante la opción **Upload QKView**.

7. Una vez procesado (puede tomar 2–10 minutos), revise la sección **Diagnostics** y busque:

   | Diagnóstico esperado | Relación con falla del laboratorio |
   |---|---|
   | Advertencias sobre configuración de monitors | Relacionado con F3 (si el QKView se generó antes de la corrección) o ausencia de advertencia (si se generó después) |
   | Advertencias sobre Connection Limits muy bajos | Relacionado con F5 |
   | Advertencias sobre Persistence Profile | Relacionado con F6 |
   | Estado general del sistema (CPU, memoria) | Línea base de recursos |

   > **Nota:** Dado que el QKView se genera **después** de las correcciones, iHealth puede no mostrar las fallas originales. Sin embargo, debería mostrar diagnósticos generales sobre la configuración, versión de software y mejores prácticas. Documente al menos **dos diagnósticos** que iHealth presente, aunque sean recomendaciones generales.

8. Si iHealth muestra heurísticas (por ejemplo, recomendaciones de versión de software, configuración de NTP, o parámetros de monitor), documente:

   | # | Diagnóstico iHealth | Severidad | Descripción | Acción recomendada |
   |---|---|---|---|---|
   | 1 | | | | |
   | 2 | | | | |

**Salida esperada:**

- Archivo `lab9.qkview` generado exitosamente.
- Al menos dos diagnósticos documentados de iHealth.

**Verificación:**

- [ ] El archivo QKView existe y tiene tamaño razonable (>30 MB).
- [ ] El QKView fue subido exitosamente a iHealth.
- [ ] Se documentaron al menos dos diagnósticos de iHealth con su severidad y descripción.

---

### Paso 10: Completar la hoja de registro de troubleshooting

**Objetivo:** Consolidar toda la documentación del laboratorio en una hoja de registro estructurada que sirva como evidencia de la resolución completa del escenario.

**Instrucciones:**

1. Complete la siguiente tabla para cada una de las seis fallas:

   | # | Síntoma observado | Componente afectado | Comando de detección | Causa raíz | Acción correctiva (comando) | Verificación (comando y resultado) |
   |---|---|---|---|---|---|---|
   | F1 | VS no acepta conexiones; icono negro en Network Map | Virtual Server `vs_app_https` | `tmsh show ltm virtual vs_app_https` → State: disabled | Estado administrativo cambiado a Disabled | `tmsh modify ltm virtual vs_app_https enabled` | `tmsh show ltm virtual vs_app_https` → State: enabled, Availability: available |
   | F2 | Sin procesamiento L7; cookie persistence no se inserta | Virtual Server Profiles | `tmsh list ltm virtual vs_app_https profiles` → `http_app` ausente | HTTP Profile eliminado del VS | `tmsh modify ltm virtual vs_app_https profiles add { http_app { context all } }` | `tmsh list ltm virtual vs_app_https profiles` → `http_app` presente |
   | F3 | Todos los Pool Members marcados Offline (rojo) | Health Monitor `mon_http_app` | `tmsh list ltm monitor http mon_http_app` → recv `"404 Not Found"` | Receive String no coincide con respuesta real del backend (200 OK) | `tmsh modify ltm monitor http mon_http_app recv "200 OK"` | `tmsh show ltm pool pool_app members` → Availability: available |
   | F4 | Priority Group no activa correctamente | Pool `pool_app` | `tmsh list ltm pool pool_app min-active-members` → 99 | Threshold inalcanzable con 2 members | `tmsh modify ltm pool pool_app min-active-members 1` | `tmsh list ltm pool pool_app min-active-members` → 1 |
   | F5 | Distribución de carga desequilibrada; member rechaza conexiones | Pool Member `10.20.20.11:80` | `tmsh list ltm pool pool_app members` → connection-limit 1 | Connection Limit demasiado restrictivo | `tmsh modify ltm pool pool_app members { 10.20.20.11:80 { connection-limit 0 } }` | `tmsh list ltm pool pool_app members { 10.20.20.11:80 }` → connection-limit 0 |
   | F6 | Cookie de persistence no se inserta en respuestas | Persistence Profile `persist_cookie_app` | `tmsh list ltm persistence cookie persist_cookie_app cookie-name` → `""` | Cookie Name vacío | `tmsh modify ltm persistence cookie persist_cookie_app cookie-name "BIGipServer_pool_app"` | `curl -sk -D - https://10.10.10.100/` → `Set-Cookie: BIGipServer_pool_app=...` |

2. Revise los logs finales para confirmar que no hay errores residuales:

   ```bash
   tail -50 /var/log/ltm | grep -iE "error|fail|down"
   ```

   **Resultado esperado:** Sin mensajes de error recientes (o solo mensajes históricos del periodo de fallas).

3. Realice una verificación final completa:

   ```bash
   # Estado del Virtual Server
   tmsh show ltm virtual vs_app_https

   # Estado del Pool y members
   tmsh show ltm pool pool_app
   tmsh show ltm pool pool_app members

   # Tabla de persistence (después de generar tráfico)
   curl -sk https://10.10.10.100/ > /dev/null
   tmsh show ltm persistence persist-records

   # Estadísticas finales
   tmsh show ltm virtual vs_app_https stats
   tmsh show ltm pool pool_app stats
   ```

**Salida esperada:**

Todos los objetos en estado `available` y `enabled`. Estadísticas con tráfico procesado. Tabla de persistence con registros activos. Sin errores en logs.

**Verificación:**

- [ ] La hoja de registro de las 6 fallas está completa con todos los campos.
- [ ] La verificación final muestra todos los objetos en estado operativo.
- [ ] Los logs no muestran errores residuales.

---

## Validación y Pruebas

### Criterios de éxito medibles

| # | Criterio | Evidencia requerida | Resultado |
|---|---|---|---|
| V1 | Las 6 fallas fueron identificadas correctamente | Tabla de hallazgos del Paso 3 completada con los 6 componentes y valores incorrectos | ☐ |
| V2 | `curl -sk https://10.10.10.100/` devuelve HTTP 200 | Salida del comando mostrando código 200 | ☐ |
| V3 | Cookie de persistence presente en respuesta | Header `Set-Cookie: BIGipServer_pool_app=...` en salida de `curl -D -` | ☐ |
| V4 | Ambos Pool Members reciben tráfico | `tmsh show ltm pool pool_app members stats` muestra conexiones > 0 en ambos members | ☐ |
| V5 | Captura post-corrección muestra TLS handshake completo | Archivo `.pcap` analizado en Wireshark con filtro `ssl.handshake` mostrando secuencia completa | ☐ |
| V6 | QKView generado y subido a iHealth | Archivo `lab9.qkview` en `/var/tmp/` y captura de pantalla de iHealth con diagnósticos | ☐ |
| V7 | Línea base pre y post documentada | Archivos `baseline_pre.txt` y `baseline_post.txt` con datos de CPU, memoria, disco e interfaces | ☐ |
| V8 | Hoja de registro completa con las 6 fallas | Tabla con síntoma, componente, causa raíz, corrección y verificación para cada falla | ☐ |

### Caso adverso: información contradictoria en logs

Durante el proceso de troubleshooting, es posible que los logs en `/var/log/ltm` muestren mensajes que aparenten señalar una causa raíz diferente a la real. Por ejemplo:

- Un mensaje de `pool member 10.20.20.11:80 monitor status down` podría llevar a sospechar un problema en el servidor web, cuando la causa real es la Receive String incorrecta del monitor (F3).
- Un mensaje de conexión rechazada podría atribuirse a un firewall, cuando la causa es el VS deshabilitado (F1).

**Instrucción de validación adversa:** Antes de dar por cerrada cada falla, el estudiante debe **confirmar la causa raíz con al menos dos fuentes independientes** (por ejemplo, el comando `tmsh list` del objeto + una prueba directa contra el backend). No se debe aceptar una corrección basada únicamente en la desaparición del síntoma sin verificar que la causa raíz fue eliminada.

**Ejemplo de validación cruzada para F3:**

```bash
## Fuente 1: Configuración del monitor
tmsh list ltm monitor http mon_http_app recv
## Debe mostrar "200 OK"

## Fuente 2: Respuesta real del backend
curl -s -o /dev/null -w "%{http_code}" http://10.20.20.11/
## Debe mostrar 200

## Fuente 3: Estado del member después de la corrección
tmsh show ltm pool pool_app members { 10.20.20.11:80 }
## Debe mostrar Availability: available
```

## Solución de Problemas

### Problema 1: Después de corregir F1 (habilitar VS), el Virtual Server muestra `available` pero `curl` sigue fallando con "Connection refused"

**Síntomas:**
- `tmsh show ltm virtual vs_app_https` muestra `State: enabled`, `Availability: available`.
- `curl -vk https://10.10.10.100/` devuelve `curl: (7) Failed to connect to 10.10.10.100 port 443: Connection refused`.
- El Pool muestra members disponibles.

**Causa:**
El HTTP Profile (F2) no fue restaurado antes de habilitar el VS, o se agregó con un contexto incorrecto. Cuando un VS tipo HTTPS tiene un Client SSL Profile pero carece de HTTP Profile, puede aceptar la conexión TLS pero fallar al procesar la petición HTTP, resultando en un cierre abrupto que el cliente percibe como rechazo. Otra causa posible es que el orden de corrección no fue respetado y el VS fue habilitado mientras el Pool aún no tenía members disponibles (F3 sin corregir), lo que puede hacer que el VS entre en estado `available` brevemente y luego caiga.

**Solución:**

1. Verifique que el HTTP Profile está asignado:

   ```bash
   tmsh list ltm virtual vs_app_https profiles
   ```

2. Si no aparece `http_app`, agréguelo:

   ```bash
   tmsh modify ltm virtual vs_app_https profiles add { http_app { context all } }
   ```

3. Verifique que el Pool tiene members disponibles:

   ```bash
   tmsh show ltm pool pool_app members
   ```

4. Si los members están offline, corrija primero el monitor (F3) y espere el ciclo de verificación.

5. Repita la prueba:

   ```bash
   curl -vk https://10.10.10.100/
   ```

---

### Problema 2: Los Pool Members aparecen `available` después de corregir F3, pero solo uno recibe tráfico

**Síntomas:**
- `tmsh show ltm pool pool_app members` muestra ambos members como `available`.
- Después de 100 requests con `curl`, `tmsh show ltm pool pool_app members stats` muestra que `10.20.20.11:80` tiene 1 conexión total y `10.20.20.12:80` tiene 99 conexiones.
- El método de LB es Round Robin.

**Causa:**
La falla F5 (Connection Limit = 1 en `10.20.20.11:80`) no fue corregida. Con un límite de 1 conexión concurrente, el member acepta la primera conexión pero rechaza todas las siguientes mientras esa conexión esté activa. Con conexiones HTTP keep-alive o si las conexiones se solapan, el efecto es que casi todo el tráfico se dirige al otro member. Adicionalmente, si la falla F4 (Priority Group Activation threshold = 99) tampoco fue corregida, el comportamiento de selección de members puede ser impredecible.

**Solución:**

1. Verifique el Connection Limit:

   ```bash
   tmsh list ltm pool pool_app members { 10.20.20.11:80 } connection-limit
   ```

2. Si el valor es diferente de `0`, corríjalo:

   ```bash
   tmsh modify ltm pool pool_app members { 10.20.20.11:80 { connection-limit 0 } }
   ```

3. Verifique también el Priority Group Activation threshold:

   ```bash
   tmsh list ltm pool pool_app min-active-members
   ```

4. Si el valor es diferente de `1`, corríjalo:

   ```bash
   tmsh modify ltm pool pool_app min-active-members 1
   ```

5. Reinicie las estadísticas para una prueba limpia:

   ```bash
   tmsh reset-stats ltm pool pool_app
   ```

6. Repita la prueba de carga y verifique distribución equitativa:

   ```bash
   for i in $(seq 1 50); do curl -sk https://10.10.10.100/ > /dev/null; done
   tmsh show ltm pool pool_app members stats
   ```

## Limpieza

> **IMPORTANTE:** Este es el último laboratorio del curso. La limpieza es opcional y depende de si el entorno será reutilizado.

1. **Si el entorno será reutilizado para otra cohorte**, restaure las fallas para el siguiente grupo de estudiantes ejecutando los comandos de inyección de la sección "Comandos de inyección de fallas" del instructor.

2. **Elimine los archivos temporales** generados durante el laboratorio:

   ```bash
   rm -f /var/tmp/capture_pre_fix.pcap
   rm -f /var/tmp/capture_post_fix.pcap
   rm -f /var/tmp/baseline_pre.txt
   rm -f /var/tmp/baseline_post.txt
   rm -f /var/tmp/lab9.qkview
   ```

3. **Verifique el espacio de disco** después de la limpieza:

   ```bash
   tmsh show sys disk logical-disk
   ```

4. **Si el entorno será destruido**, no se requiere acción adicional. Documente el estado final del laboratorio para referencia del instructor.

## Resumen

En este laboratorio se aplicó la metodología sistemática de troubleshooting de cinco fases para resolver un escenario con seis fallas inyectadas simultáneamente en un entorno BIG-IP. Los puntos clave consolidados son:

- **Identificación estructurada:** El Network Map de TMUI y las pruebas de conectividad con `curl` proporcionan una primera vista rápida del estado del servicio, pero no son suficientes para determinar causas raíz.
- **Inspección jerárquica:** Recorrer la cadena Virtual Server → Pool → Pool Members → Health Monitor → Persistence → Profiles permite aislar fallas sin saltar a conclusiones. Cada nivel puede enmascarar o amplificar problemas de niveles inferiores.
- **Orden de corrección:** Las dependencias entre objetos BIG-IP exigen un orden de corrección deliberado. Habilitar un VS antes de que su Pool tenga members disponibles puede generar síntomas confusos adicionales.
- **Evidencia cruzada:** Cada falla debe confirmarse con al menos dos fuentes independientes (`tmsh list`, `tmsh show`, prueba directa al backend, logs) para evitar diagnósticos incorrectos basados en correlaciones espurias.
- **Utilización de recursos:** La comparación de líneas base pre y post-corrección permite cuantificar el impacto de las fallas en CPU, memoria e interfaces, y confirmar que no quedan efectos residuales.
- **Documentación operativa:** El QKView y la hoja de registro de troubleshooting son entregables profesionales que permiten auditar el proceso de resolución y transferir conocimiento al equipo.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación F5 BIG-IP LTM 17.1 | [https://techdocs.f5.com/en-us/bigip-17-1-0.html](https://techdocs.f5.com/en-us/bigip-17-1-0.html) |
| Referencia de comandos TMSH | [https://clouddocs.f5.com/cli/tmsh-reference/latest/](https://clouddocs.f5.com/cli/tmsh-reference/latest/) |
| F5 iHealth | [https://ihealth.f5.com](https://ihealth.f5.com) |
| K12878: Generación de QKView | [https://my.f5.com/manage/s/article/K12878](https://my.f5.com/manage/s/article/K12878) |
| K13637: Troubleshooting de Health Monitors | [https://my.f5.com/manage/s/article/K13637](https://my.f5.com/manage/s/article/K13637) |
| K411: Troubleshooting de Pool Members | [https://my.f5.com/manage/s/article/K411](https://my.f5.com/manage/s/article/K411) |
