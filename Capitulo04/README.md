# Laboratorio 6 — Alta disponibilidad, Device Trust y ConfigSync

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Alineación** | F5CAB4.01, F5CAB4.08 |
| **Versión BIG-IP** | F5 BIG-IP VE 17.1.1.3 |

## Descripción General

En este laboratorio operarás y diagnosticarás una implementación de alta disponibilidad Active/Standby compuesta por dos instancias BIG-IP VE (BIG-IP-A y BIG-IP-B) ya configuradas con Device Trust, Device Group de tipo Sync-Failover, ConfigSync y un Virtual Server operativo. Ejecutarás acciones de Force to Standby y analizarás Force Offline, inducirás un failover controlado, generarás y resolverás estados de sincronización incorrectos (Changes Pending y Sync Failed) y documentarás las evidencias en cada fase. Todas las actividades se realizan sobre una infraestructura pre-existente; no se construye la topología desde cero.

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Verificar e interpretar el estado Active/Standby, Device Trust, Device Groups y ConfigSync mediante TMUI y TMSH en ambos dispositivos del par HA.
- [ ] Ejecutar Force to Standby sobre el dispositivo Active, observar la transición de roles y diferenciar operativamente esta acción de Force Offline.
- [ ] Simular un failover inducido, analizar los logs de `/var/log/ltm` y ejecutar el procedimiento de recuperación para restaurar la operación normal del par.
- [ ] Diagnosticar estados de sincronización incorrectos (Changes Pending y Sync Failed) e identificar y ejecutar las acciones correctivas apropiadas.
- [ ] Documentar evidencias medibles de cada fase del laboratorio para demostrar comprensión operativa de la alta disponibilidad en BIG-IP.

## Prerrequisitos

### Conocimientos Requeridos

| Área | Detalle |
|---|---|
| TCP/IP y redes | Direccionamiento IP, subredes, VLANs, routing básico |
| HTTP/HTTPS | Puertos, métodos, códigos de respuesta |
| CLI Linux/Unix | Navegación de directorios, `cat`, `tail`, `grep`, `ssh` |
| BIG-IP (Módulos 1–3) | Arquitectura BIG-IP, Virtual Servers, pools, Self IPs, TMUI, TMSH |

### Acceso y Credenciales

| Recurso | Dirección / Dato | Credenciales |
|---|---|---|
| BIG-IP-A (Management) | `192.0.2.245` | `admin` / `F5l@bP@ss2024!` |
| BIG-IP-B (Management) | `192.0.2.246` | `admin` / `F5l@bP@ss2024!` |
| BIG-IP-A TMUI | `https://192.0.2.245` | `admin` / `F5l@bP@ss2024!` |
| BIG-IP-B TMUI | `https://192.0.2.246` | `admin` / `F5l@bP@ss2024!` |
| Cliente de administración | `192.0.2.10` | Acceso SSH y navegador |
| Cliente de pruebas | `10.10.10.10` | Acceso curl y navegador |

> **Nota importante sobre la topología del curso:** La topología persistente del curso incluye un único BIG-IP VE. Para este laboratorio específico, el entorno se ha ampliado temporalmente con una segunda instancia (BIG-IP-B) pre-configurada. Al finalizar, se regresará a la topología de un solo BIG-IP. Los estudiantes **no** necesitan crear la segunda instancia; ya está disponible al iniciar el laboratorio.

## Entorno de Laboratorio

### Topología de Red del Laboratorio

```
                    Red Management: 192.0.2.0/24
  ┌──────────┐      ┌──────────────┐      ┌──────────────┐
  │ Cliente   │      │  BIG-IP-A    │      │  BIG-IP-B    │
  │ Admin     │──────│  Mgmt:       │──────│  Mgmt:       │
  │ .2.10     │      │  .2.245      │      │  .2.246      │
  └──────────┘      └──────┬───────┘      └──────┬───────┘
                           │                      │
                    Red External: 10.10.10.0/24
  ┌──────────┐      ┌──────┴───────┐      ┌──────┴───────┐
  │ Cliente   │      │ Self IP:     │      │ Self IP:     │
  │ Pruebas   │──────│ .10.245      │──────│ .10.246      │
  │ .10.10    │      │ Float:       │      │              │
  └──────────┘      │ .10.100 (VS) │      └──────┬───────┘
                    └──────┬───────┘             │
                           │                      │
                    Red Internal: 10.20.20.0/24
                    ┌──────┴───────┐      ┌──────┴───────┐
                    │ Self IP:     │      │ Self IP:     │
                    │ .20.245      │      │ .20.246      │
                    └──────┬───────┘      └──────┬───────┘
                           │                      │
                    ┌──────┴──┐          ┌────────┴┐
                    │ web01   │          │ web02   │
                    │ .20.11  │          │ .20.12  │
                    └─────────┘          └─────────┘
```

### Direccionamiento Detallado

| Dispositivo | Management IP | External Self IP | Internal Self IP | Floating IP |
|---|---|---|---|---|
| BIG-IP-A | 192.0.2.245/24 | 10.10.10.245/24 | 10.20.20.245/24 | 10.10.10.100 (VS) |
| BIG-IP-B | 192.0.2.246/24 | 10.10.10.246/24 | 10.20.20.246/24 | — |
| Floating Self IP External | — | 10.10.10.240/24 | — | — |
| Floating Self IP Internal | — | — | 10.20.20.240/24 | — |

### Configuración Pre-existente

El entorno ya incluye los siguientes objetos configurados en ambos dispositivos:

| Objeto | Configuración |
|---|---|
| **Device Trust** | Establecido entre BIG-IP-A y BIG-IP-B |
| **Device Group** | `dg-lab-failover` (tipo Sync-Failover, ambos miembros) |
| **Traffic Group** | `traffic-group-1` (activo en BIG-IP-A) |
| **Virtual Server** | `vs_http` en `10.10.10.100:80` (HTTP) |
| **Pool** | `pool_web` con miembros `10.20.20.11:80` y `10.20.20.12:80` |
| **Monitor** | `http` aplicado a `pool_web` |
| **ConfigSync** | Estado inicial: In Sync |
| **Failover** | BIG-IP-A = Active, BIG-IP-B = Standby |

### Software del Entorno

| Componente | Versión Exacta | Fuente Oficial |
|---|---|---|
| F5 BIG-IP VE | 17.1.1.3 | [https://my.f5.com/manage/s/downloads](https://my.f5.com/manage/s/downloads) |
| Ubuntu Server (servidores web) | 22.04.4 LTS | [https://releases.ubuntu.com/22.04/](https://releases.ubuntu.com/22.04/) |
| Apache HTTP Server | 2.4.58 | [https://httpd.apache.org/download.cgi](https://httpd.apache.org/download.cgi) |
| curl (cliente de pruebas) | 7.81.0 | Incluido en Ubuntu 22.04 LTS |
| OpenSSH (acceso CLI) | 9.6p1 | Incluido en BIG-IP 17.1.1.3 |

### Verificación Inicial del Entorno

Desde el cliente de administración (`192.0.2.10`), ejecuta las siguientes comprobaciones antes de iniciar el laboratorio:

```bash
## Verificar conectividad a ambos BIG-IP por Management
ping -c 3 192.0.2.245
ping -c 3 192.0.2.246

## Verificar acceso SSH a ambos dispositivos
ssh admin@192.0.2.245 'tmsh show cm failover-status'
ssh admin@192.0.2.246 'tmsh show cm failover-status'
```

**Resultado esperado:** BIG-IP-A responde `ACTIVE` y BIG-IP-B responde `STANDBY`. Si los resultados difieren, notifica al instructor antes de continuar.

Desde el cliente de pruebas (`10.10.10.10`):

```bash
## Verificar que el Virtual Server responde
curl -s -o /dev/null -w "%{http_code}" http://10.10.10.100
```

**Resultado esperado:** Código HTTP `200`.

## Instrucciones Paso a Paso

### Paso 1: Exploración del estado Active/Standby mediante TMUI y TMSH

**Objetivo:** Verificar e interpretar el estado actual de alta disponibilidad, Device Trust, Device Groups y ConfigSync en ambos dispositivos utilizando la interfaz gráfica y la línea de comandos.

**Instrucciones:**

1. Abre dos pestañas del navegador en el cliente de administración. Accede a la TMUI de BIG-IP-A (`https://192.0.2.245`) en una pestaña y a BIG-IP-B (`https://192.0.2.246`) en la otra. Inicia sesión con las credenciales `admin` / `F5l@bP@ss2024!`.

2. En BIG-IP-A, observa la barra superior de TMUI. Identifica el indicador de estado de HA que aparece junto al nombre del dispositivo. Registra en tu hoja de trabajo si muestra **ACTIVE** (normalmente en verde) o **STANDBY** (normalmente en azul/gris).

3. Navega a **Device Management > Devices** en BIG-IP-A. Haz clic en el nombre del dispositivo local (`bigip-a.f5lab.local`). Documenta los siguientes campos:
   - **Name**
   - **Management IP**
   - **Failover State**
   - **ConfigSync Address**

4. En la misma vista, haz clic en el dispositivo peer (`bigip-b.f5lab.local`). Documenta los mismos campos.

5. Navega a **Device Management > Device Trust > Peer Authority Devices**. Confirma que BIG-IP-B aparece listado como dispositivo de confianza. Registra el estado del certificado de confianza.

6. Navega a **Device Management > Device Groups**. Haz clic en `dg-lab-failover`. Documenta:
   - **Type** (debe ser `Sync-Failover`)
   - **Members** (deben aparecer ambos dispositivos)

7. Navega a **Device Management > Overview**. Identifica el indicador de sincronización. Registra si muestra **In Sync**, **Changes Pending** o **Sync Failed**.

8. Abre dos terminales SSH desde el cliente de administración, una a cada BIG-IP:

   ```bash
   # Terminal 1 - BIG-IP-A
   ssh admin@192.0.2.245
   ```

   ```bash
   # Terminal 2 - BIG-IP-B
   ssh admin@192.0.2.246
   ```

9. En **BIG-IP-A**, ejecuta los siguientes comandos y registra las salidas:

   ```bash
   tmsh show cm failover-status
   ```

   ```bash
   tmsh show cm sync-status
   ```

   ```bash
   tmsh show cm trust-domain all
   ```

   ```bash
   tmsh show cm device-group
   ```

   ```bash
   tmsh show cm traffic-group
   ```

10. Repite los mismos cinco comandos en **BIG-IP-B** y registra las salidas.

11. En **BIG-IP-A**, ejecuta el siguiente comando para ver los detalles del dispositivo local:

    ```bash
    tmsh show cm device bigip-a.f5lab.local
    ```

12. Completa la siguiente tabla en tu hoja de trabajo:

    | Atributo | BIG-IP-A | BIG-IP-B |
    |---|---|---|
    | Failover Status | | |
    | Sync Status | | |
    | Trust Domain Status | | |
    | Device Group Membership | | |
    | Traffic Group Owner | | |

**Resultado esperado:**

En BIG-IP-A:
```text
Failover status
  ACTIVE

Sync Status:
  Status: In Sync

Trust Domain:
  Status: Valid
```

En BIG-IP-B:
```text
Failover status
  STANDBY

Sync Status:
  Status: In Sync

Trust Domain:
  Status: Valid
```

El traffic group `traffic-group-1` debe mostrar a BIG-IP-A como propietario actual.

**Verificación:**

- [ ] BIG-IP-A muestra estado `ACTIVE` tanto en TMUI como en TMSH.
- [ ] BIG-IP-B muestra estado `STANDBY` tanto en TMUI como en TMSH.
- [ ] Device Trust muestra estado `Valid` en ambos dispositivos.
- [ ] El Device Group `dg-lab-failover` es de tipo `Sync-Failover` y contiene ambos miembros.
- [ ] ConfigSync muestra `In Sync` en ambos dispositivos.
- [ ] La tabla de tu hoja de trabajo está completa con todos los campos documentados.

---

### Paso 2: Verificación de tráfico de producción a través del dispositivo Active

**Objetivo:** Confirmar que el Virtual Server procesa tráfico correctamente a través del dispositivo Active antes de ejecutar cualquier operación de failover.

**Instrucciones:**

1. Desde el **cliente de pruebas** (`10.10.10.10`), ejecuta cinco solicitudes HTTP al Virtual Server y registra las respuestas:

   ```bash
   for i in $(seq 1 5); do
     echo "--- Solicitud $i ---"
     curl -s -w "\nHTTP Code: %{http_code}\nTime: %{time_total}s\n" http://10.10.10.100
     echo ""
   done
   ```

2. Observa en las respuestas si el contenido proviene de `web01` o `web02` (los servidores están configurados para identificarse en la página de respuesta).

3. En **BIG-IP-A** (dispositivo Active), verifica las estadísticas del Virtual Server:

   ```bash
   tmsh show ltm virtual vs_http
   ```

4. Verifica las estadísticas del pool:

   ```bash
   tmsh show ltm pool pool_web
   ```

5. Verifica el estado de los miembros del pool:

   ```bash
   tmsh show ltm pool pool_web members
   ```

6. En **BIG-IP-B** (dispositivo Standby), ejecuta los mismos tres comandos. Compara las estadísticas de tráfico.

7. Registra en tu hoja de trabajo:
   - Número de conexiones totales en el VS (BIG-IP-A vs BIG-IP-B)
   - Estado de cada miembro del pool en ambos dispositivos
   - Código HTTP recibido en cada solicitud curl

**Resultado esperado:**

Las cinco solicitudes curl devuelven código HTTP `200`. BIG-IP-A muestra contadores de conexiones incrementados en `vs_http` y `pool_web`. BIG-IP-B muestra contadores en cero o muy bajos (no procesa tráfico de producción en estado Standby). Ambos miembros del pool aparecen como `available` (verde) en ambos dispositivos.

**Verificación:**

- [ ] Todas las solicitudes HTTP devuelven código `200`.
- [ ] BIG-IP-A muestra estadísticas de tráfico activas en el Virtual Server.
- [ ] BIG-IP-B muestra estadísticas mínimas o nulas (confirma que no procesa tráfico).
- [ ] Ambos miembros del pool están en estado `available` en ambos dispositivos.

---

### Paso 3: Ejecución de Force to Standby y observación de cambio de roles

**Objetivo:** Ejecutar Force to Standby sobre BIG-IP-A, observar la transición de roles hacia BIG-IP-B y verificar la continuidad del tráfico de producción.

**Instrucciones:**

1. **Antes de ejecutar la conmutación**, marca un punto de referencia en los logs de ambos dispositivos. En cada BIG-IP, ejecuta:

   ```bash
   # En BIG-IP-A
   tail -1 /var/log/ltm | cat
   ```

   ```bash
   # En BIG-IP-B
   tail -1 /var/log/ltm | cat
   ```

   Registra la marca de tiempo de la última línea para poder filtrar los eventos posteriores.

2. En el **cliente de pruebas**, inicia un ciclo continuo de solicitudes HTTP para observar la continuidad del servicio durante la conmutación:

   ```bash
   while true; do
     RESP=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 2 http://10.10.10.100)
     echo "$(date '+%H:%M:%S') - HTTP $RESP"
     sleep 1
   done
   ```

   Deja este comando ejecutándose en una terminal separada.

3. En **BIG-IP-A**, ejecuta Force to Standby mediante TMSH:

   ```bash
   tmsh run sys failover standby
   ```

4. Observa inmediatamente la salida del ciclo de curl en el cliente de pruebas. Registra:
   - ¿Hubo alguna solicitud con código diferente a `200`?
   - ¿Cuántos segundos (aproximadamente) transcurrieron con respuestas fallidas, si las hubo?

5. Verifica el nuevo estado de failover en **ambos** dispositivos:

   ```bash
   # En BIG-IP-A
   tmsh show cm failover-status
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm failover-status
   ```

6. Verifica el propietario del traffic group:

   ```bash
   # En cualquiera de los dos BIG-IP
   tmsh show cm traffic-group traffic-group-1
   ```

7. En **BIG-IP-A**, revisa los eventos de failover en el log:

   ```bash
   grep -i "failover\|standby\|active" /var/log/ltm | tail -20
   ```

8. En **BIG-IP-B**, revisa los mismos eventos:

   ```bash
   grep -i "failover\|standby\|active" /var/log/ltm | tail -20
   ```

9. Registra las líneas de log relevantes en tu hoja de trabajo. Identifica los mensajes que indican:
   - BIG-IP-A pasó a Standby
   - BIG-IP-B asumió el rol Active
   - El traffic group cambió de propietario

10. En la TMUI de ambos dispositivos, actualiza la página y confirma visualmente el cambio de roles en la barra superior.

11. Verifica las estadísticas del Virtual Server en **BIG-IP-B** (ahora Active):

    ```bash
    tmsh show ltm virtual vs_http
    ```

12. Detén el ciclo de curl en el cliente de pruebas (`Ctrl+C`).

**Resultado esperado:**

```text
## BIG-IP-A
Failover status
  STANDBY

## BIG-IP-B
Failover status
  ACTIVE
```

El traffic group `traffic-group-1` muestra a BIG-IP-B como propietario actual. El ciclo de curl puede mostrar 0–2 solicitudes fallidas durante la transición (esto es normal en un failover sin mirroring de conexiones). Los logs de `/var/log/ltm` contienen entradas indicando el cambio de estado.

**Verificación:**

- [ ] BIG-IP-A muestra estado `STANDBY`.
- [ ] BIG-IP-B muestra estado `ACTIVE`.
- [ ] El traffic group `traffic-group-1` tiene como propietario a BIG-IP-B.
- [ ] El tráfico HTTP se restauró en un máximo de 2–3 segundos.
- [ ] Los logs de `/var/log/ltm` contienen eventos de failover documentados en tu hoja de trabajo.
- [ ] La TMUI de ambos dispositivos refleja los nuevos roles.

---

### Paso 4: Análisis de Force Offline vs Force to Standby y restauración del estado original

**Objetivo:** Comprender la diferencia operativa entre Force to Standby y Force Offline, y restaurar el par al estado original (BIG-IP-A = Active).

**Instrucciones:**

1. Con BIG-IP-B actualmente Active, analiza conceptualmente qué ocurriría si ejecutaras `Force Offline` en BIG-IP-B. **No ejecutes este comando todavía.** Responde en tu hoja de trabajo:

   - ¿Qué sucede con el tráfico si el dispositivo Active se pone Force Offline?
   - ¿Puede el dispositivo volver a ser Active automáticamente después de Force Offline?
   - ¿En qué escenarios operativos es apropiado usar Force Offline en lugar de Force to Standby?

2. Para observar el comportamiento real de Force Offline de forma segura, primero restaura BIG-IP-A como Active. Ejecuta en **BIG-IP-B**:

   ```bash
   tmsh run sys failover standby
   ```

3. Verifica que BIG-IP-A volvió a ser Active:

   ```bash
   # En BIG-IP-A
   tmsh show cm failover-status
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm failover-status
   ```

4. Ahora, con BIG-IP-A Active y BIG-IP-B Standby, ejecuta Force Offline en **BIG-IP-B** (el dispositivo Standby, para no interrumpir tráfico):

   ```bash
   # En BIG-IP-B
   tmsh run sys failover offline
   ```

5. Verifica el estado en ambos dispositivos:

   ```bash
   # En BIG-IP-A
   tmsh show cm failover-status
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm failover-status
   ```

6. Observa en la TMUI de BIG-IP-B el indicador de estado. Debe mostrar `FORCED OFFLINE`.

7. Analiza: ¿Qué pasaría ahora si BIG-IP-A fallara? ¿BIG-IP-B asumiría el tráfico automáticamente? Registra tu respuesta.

8. Restaura BIG-IP-B a estado operativo. En **BIG-IP-B**, ejecuta:

   ```bash
   tmsh run sys failover online
   ```

9. Verifica que BIG-IP-B volvió a estado `STANDBY`:

   ```bash
   # En BIG-IP-B
   tmsh show cm failover-status
   ```

10. Completa la siguiente tabla comparativa en tu hoja de trabajo:

    | Característica | Force to Standby | Force Offline |
    |---|---|---|
    | ¿Transfiere tráfico al peer? | | |
    | ¿El dispositivo puede volver a ser Active automáticamente? | | |
    | ¿El dispositivo participa en la elección de failover? | | |
    | ¿Caso de uso típico? | | |

**Resultado esperado:**

Después de Force Offline en BIG-IP-B:
```text
## BIG-IP-A
Failover status
  ACTIVE

## BIG-IP-B
Failover status
  FORCED OFFLINE
```

Después de restaurar con `tmsh run sys failover online`:
```text
## BIG-IP-B
Failover status
  STANDBY
```

**Tabla comparativa esperada:**

| Característica | Force to Standby | Force Offline |
|---|---|---|
| ¿Transfiere tráfico al peer? | Sí, el peer asume el traffic group | Sí, si se aplica al Active; si se aplica al Standby, no hay transferencia |
| ¿El dispositivo puede volver a ser Active automáticamente? | Sí, si ocurre un failover posterior | No, requiere intervención administrativa (`failover online`) |
| ¿El dispositivo participa en la elección de failover? | Sí, sigue siendo elegible | No, queda excluido |
| ¿Caso de uso típico? | Mantenimiento planificado con retorno rápido | Reparación, investigación o apagado prolongado |

**Verificación:**

- [ ] Force Offline en BIG-IP-B produjo estado `FORCED OFFLINE`.
- [ ] BIG-IP-B no podría asumir tráfico automáticamente mientras está en Forced Offline.
- [ ] `tmsh run sys failover online` restauró BIG-IP-B a `STANDBY`.
- [ ] La tabla comparativa en tu hoja de trabajo está completa y es correcta.
- [ ] El estado final es: BIG-IP-A = Active, BIG-IP-B = Standby.

---

### Paso 5: Simulación de failover inducido y análisis de logs

**Objetivo:** Inducir un failover controlado deshabilitando la interfaz de red del dispositivo Active, observar la transición automática, analizar los logs y ejecutar la recuperación.

**Instrucciones:**

1. Confirma el estado inicial: BIG-IP-A = Active, BIG-IP-B = Standby.

   ```bash
   # En BIG-IP-A
   tmsh show cm failover-status
   ```

2. Inicia nuevamente el ciclo de monitoreo de tráfico desde el **cliente de pruebas**:

   ```bash
   while true; do
     RESP=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 2 http://10.10.10.100)
     echo "$(date '+%H:%M:%S') - HTTP $RESP"
     sleep 1
   done
   ```

3. Registra la hora actual antes de inducir el failover:

   ```bash
   # En BIG-IP-A
   date '+%Y-%m-%d %H:%M:%S'
   ```

4. En **BIG-IP-A**, deshabilita la interfaz externa (1.1) para simular una pérdida de conectividad:

   ```bash
   tmsh modify net interface 1.1 disabled
   ```

   > **Advertencia:** Esta acción interrumpirá la conectividad de tráfico de BIG-IP-A. La sesión SSH por Management IP seguirá activa porque usa una interfaz diferente.

5. Observa la salida del ciclo de curl. Registra:
   - Hora del primer error (si lo hay)
   - Hora de la primera respuesta exitosa después del failover
   - Duración aproximada de la interrupción

6. Después de 15–20 segundos, verifica el estado de failover en ambos dispositivos:

   ```bash
   # En BIG-IP-A (vía Management SSH)
   tmsh show cm failover-status
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm failover-status
   ```

7. Analiza los logs de failover en **BIG-IP-B** (que debió asumir el rol Active):

   ```bash
   grep -i "failover\|active\|standby\|ha " /var/log/ltm | tail -30
   ```

8. Analiza los logs en **BIG-IP-A**:

   ```bash
   grep -i "failover\|active\|standby\|ha \|interface" /var/log/ltm | tail -30
   ```

9. Registra en tu hoja de trabajo al menos tres líneas de log significativas de cada dispositivo que evidencien:
   - Detección de falla en BIG-IP-A
   - Asunción del rol Active por BIG-IP-B
   - Cambio de propietario del traffic group

10. **Recuperación:** Rehabilita la interfaz en BIG-IP-A:

    ```bash
    # En BIG-IP-A
    tmsh modify net interface 1.1 enabled
    ```

11. Espera 10 segundos y verifica el estado:

    ```bash
    # En BIG-IP-A
    tmsh show cm failover-status
    ```

    ```bash
    # En BIG-IP-B
    tmsh show cm failover-status
    ```

    > **Nota:** Dependiendo de la configuración de auto-failback, BIG-IP-A puede permanecer en Standby incluso después de recuperar la interfaz. Este es el comportamiento esperado por defecto: el dispositivo que asumió el rol Active lo conserva hasta que se ejecute una conmutación manual o ocurra otro evento de failover.

12. Si BIG-IP-A permanece en Standby (comportamiento esperado), restaura el estado original ejecutando Force to Standby en **BIG-IP-B**:

    ```bash
    # En BIG-IP-B
    tmsh run sys failover standby
    ```

13. Verifica el estado final:

    ```bash
    # En BIG-IP-A
    tmsh show cm failover-status
    ```

    ```bash
    # En BIG-IP-B
    tmsh show cm failover-status
    ```

14. Detén el ciclo de curl (`Ctrl+C`).

15. Verifica que el tráfico fluye correctamente:

    ```bash
    # Desde el cliente de pruebas
    curl -s -w "\nHTTP Code: %{http_code}\n" http://10.10.10.100
    ```

**Resultado esperado:**

Después de deshabilitar la interfaz 1.1 en BIG-IP-A:
- BIG-IP-B asume el rol Active en un plazo de 5–15 segundos.
- El ciclo de curl muestra una interrupción breve seguida de respuestas `200`.

Después de rehabilitar la interfaz y restaurar manualmente:
```text
## BIG-IP-A
Failover status
  ACTIVE

## BIG-IP-B
Failover status
  STANDBY
```

Ejemplo de líneas de log esperadas en BIG-IP-B:
```text
notice mcpd[xxxx]: 01071234:5: Device bigip-b.f5lab.local, now Active for traffic group traffic-group-1
```

**Verificación:**

- [ ] BIG-IP-B asumió el rol Active automáticamente tras la falla simulada.
- [ ] La interrupción de tráfico fue inferior a 15 segundos.
- [ ] Los logs de `/var/log/ltm` contienen eventos de failover documentados.
- [ ] La interfaz 1.1 de BIG-IP-A fue rehabilitada exitosamente.
- [ ] El estado final es: BIG-IP-A = Active, BIG-IP-B = Standby.
- [ ] El tráfico HTTP responde con código `200`.

---

### Paso 6: Diagnóstico y resolución del estado Changes Pending en ConfigSync

**Objetivo:** Generar deliberadamente un estado de ConfigSync "Changes Pending", interpretarlo correctamente y ejecutar la sincronización manual para restaurar el estado "In Sync".

**Instrucciones:**

1. Confirma el estado de sincronización actual:

   ```bash
   # En BIG-IP-A
   tmsh show cm sync-status
   ```

   Debe mostrar `In Sync`.

2. En **BIG-IP-A** (dispositivo Active), crea un pool de prueba **sin sincronizar**:

   ```bash
   tmsh create ltm pool pool_test_sync members add { 10.20.20.11:8080 10.20.20.12:8080 }
   ```

3. Verifica inmediatamente el estado de sincronización en **BIG-IP-A**:

   ```bash
   tmsh show cm sync-status
   ```

4. Verifica el estado de sincronización en **BIG-IP-B**:

   ```bash
   tmsh show cm sync-status
   ```

5. En la TMUI de BIG-IP-A, navega a **Device Management > Overview**. Observa el indicador de sincronización. Debe mostrar **Changes Pending** con una indicación de qué dispositivo tiene los cambios más recientes.

6. Registra en tu hoja de trabajo:
   - ¿Qué dispositivo muestra "Changes Pending"?
   - ¿En qué dirección debe ejecutarse la sincronización? (¿Desde A hacia B, o desde B hacia A?)

7. Verifica que el pool de prueba **no existe** en BIG-IP-B:

   ```bash
   # En BIG-IP-B
   tmsh list ltm pool pool_test_sync
   ```

   Debe devolver un error o resultado vacío.

8. Ejecuta la sincronización desde **BIG-IP-A** hacia el grupo:

   **Opción TMSH:**
   ```bash
   # En BIG-IP-A
   tmsh run cm config-sync to-group dg-lab-failover
   ```

   **Opción TMUI:** En BIG-IP-A, navega a **Device Management > Overview**, haz clic en el indicador de Changes Pending y selecciona **Sync Device to Group**.

9. Espera 10–15 segundos y verifica el estado de sincronización en ambos dispositivos:

   ```bash
   # En BIG-IP-A
   tmsh show cm sync-status
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm sync-status
   ```

10. Verifica que el pool de prueba ahora existe en **BIG-IP-B**:

    ```bash
    # En BIG-IP-B
    tmsh list ltm pool pool_test_sync
    ```

11. Registra la evidencia de sincronización exitosa en tu hoja de trabajo.

**Resultado esperado:**

Antes de sincronizar:
```text
## BIG-IP-A
Status: Changes Pending
  Recommended Action: to-group dg-lab-failover

## BIG-IP-B
Status: Changes Pending (awaiting sync from peer)
```

Después de sincronizar:
```text
## Ambos dispositivos
Status: In Sync
```

El pool `pool_test_sync` aparece en BIG-IP-B después de la sincronización.

**Verificación:**

- [ ] El estado `Changes Pending` fue generado correctamente al crear el pool sin sincronizar.
- [ ] Identificaste correctamente la dirección de sincronización (de BIG-IP-A hacia el grupo).
- [ ] La sincronización manual restauró el estado `In Sync`.
- [ ] El pool `pool_test_sync` existe en ambos dispositivos después de la sincronización.

---

### Paso 7: Simulación y recuperación de estado Sync Failed

**Objetivo:** Simular un estado de sincronización fallida, diagnosticar la causa y aplicar el procedimiento de recuperación correcto.

**Instrucciones:**

1. Confirma que el estado actual es `In Sync`:

   ```bash
   # En BIG-IP-A
   tmsh show cm sync-status
   ```

2. Para simular un estado de Sync Failed, vas a crear una discrepancia directa. En **BIG-IP-B**, crea un objeto local que entre en conflicto con la configuración sincronizada:

   ```bash
   # En BIG-IP-B
   tmsh create ltm pool pool_conflict members add { 10.20.20.11:9090 }
   ```

3. Simultáneamente, en **BIG-IP-A**, crea un pool con el mismo nombre pero diferente configuración:

   ```bash
   # En BIG-IP-A
   tmsh create ltm pool pool_conflict members add { 10.20.20.12:9090 }
   ```

4. Intenta sincronizar desde BIG-IP-A:

   ```bash
   # En BIG-IP-A
   tmsh run cm config-sync to-group dg-lab-failover
   ```

5. Verifica el estado de sincronización en ambos dispositivos:

   ```bash
   # En BIG-IP-A
   tmsh show cm sync-status
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm sync-status
   ```

   > **Nota:** Dependiendo de la versión y configuración exacta, el estado puede mostrar `Changes Pending` en lugar de `Sync Failed`. En ambos casos, la discrepancia existe y debe resolverse.

6. Revisa los logs para identificar errores de sincronización:

   ```bash
   # En BIG-IP-A
   grep -i "config.sync\|sync.*fail\|mcpd.*sync" /var/log/ltm | tail -15
   ```

   ```bash
   # En BIG-IP-B
   grep -i "config.sync\|sync.*fail\|mcpd.*sync" /var/log/ltm | tail -15
   ```

7. Registra los mensajes de error relevantes en tu hoja de trabajo.

8. **Procedimiento de recuperación:** Determina cuál dispositivo tiene la configuración "correcta" o autoritativa. En este escenario de laboratorio, BIG-IP-A (Active) es la fuente autoritativa.

9. Primero, elimina el pool conflictivo en **BIG-IP-B** para limpiar la discrepancia local:

   ```bash
   # En BIG-IP-B
   tmsh delete ltm pool pool_conflict
   ```

10. Ahora ejecuta una sincronización forzada desde BIG-IP-A (la fuente autoritativa) hacia el grupo. Usa la opción `force-full-load-push` si el estado persiste como problemático:

    ```bash
    # En BIG-IP-A
    tmsh run cm config-sync to-group dg-lab-failover
    ```

    Si el estado no se resuelve con el comando anterior:

    ```bash
    # En BIG-IP-A
    tmsh run cm config-sync force-full-load-push to-group dg-lab-failover
    ```

11. Verifica el estado de sincronización:

    ```bash
    # En BIG-IP-A
    tmsh show cm sync-status
    ```

    ```bash
    # En BIG-IP-B
    tmsh show cm sync-status
    ```

12. Verifica que `pool_conflict` existe en BIG-IP-B con la configuración de BIG-IP-A:

    ```bash
    # En BIG-IP-B
    tmsh list ltm pool pool_conflict
    ```

13. Registra en tu hoja de trabajo:
    - Causa raíz del problema de sincronización
    - Procedimiento de recuperación ejecutado
    - Estado final de ConfigSync

**Resultado esperado:**

Después de la recuperación:
```text
## Ambos dispositivos
Status: In Sync
```

El pool `pool_conflict` en BIG-IP-B contiene el miembro `10.20.20.12:9090` (la configuración de BIG-IP-A, que fue la fuente autoritativa).

**Verificación:**

- [ ] Se generó exitosamente una discrepancia de configuración entre ambos dispositivos.
- [ ] Los logs de `/var/log/ltm` contienen evidencia del problema de sincronización.
- [ ] Se identificó correctamente a BIG-IP-A como fuente autoritativa de configuración.
- [ ] La sincronización forzada restauró el estado `In Sync`.
- [ ] El pool `pool_conflict` tiene configuración consistente en ambos dispositivos.

---

### Paso 8: Revisión consolidada de logs de HA y documentación final

**Objetivo:** Realizar una revisión consolidada de los eventos de alta disponibilidad registrados durante todo el laboratorio y completar la documentación de evidencias.

**Instrucciones:**

1. En **BIG-IP-A**, genera un extracto de todos los eventos de HA del período del laboratorio:

   ```bash
   grep -iE "failover|standby|active|sync|ha |trust|device-group|traffic-group" /var/log/ltm | tail -50
   ```

2. En **BIG-IP-B**, ejecuta el mismo comando:

   ```bash
   grep -iE "failover|standby|active|sync|ha |trust|device-group|traffic-group" /var/log/ltm | tail -50
   ```

3. Identifica y clasifica los eventos en las siguientes categorías. Registra al menos un ejemplo de cada una en tu hoja de trabajo:

   | Categoría de Evento | Ejemplo de Línea de Log | Dispositivo |
   |---|---|---|
   | Cambio de estado a Active | | |
   | Cambio de estado a Standby | | |
   | Force Offline / Online | | |
   | Failover automático (por falla) | | |
   | Sincronización exitosa | | |
   | Error o advertencia de sincronización | | |

4. Verifica el estado final completo del par HA:

   ```bash
   # En BIG-IP-A
   tmsh show cm failover-status
   tmsh show cm sync-status
   tmsh show cm device
   ```

   ```bash
   # En BIG-IP-B
   tmsh show cm failover-status
   tmsh show cm sync-status
   tmsh show cm device
   ```

5. Desde el **cliente de pruebas**, ejecuta una verificación final de tráfico:

   ```bash
   for i in $(seq 1 3); do
     curl -s -w "HTTP %{http_code} - " http://10.10.10.100
     echo ""
   done
   ```

6. Completa la hoja de trabajo final del laboratorio con todas las evidencias recopiladas.

**Resultado esperado:**

Estado final del par:
- BIG-IP-A: `ACTIVE`, `In Sync`
- BIG-IP-B: `STANDBY`, `In Sync`
- Tráfico HTTP: código `200` en todas las solicitudes

La tabla de eventos contiene al menos un ejemplo de cada categoría.

**Verificación:**

- [ ] Se identificaron eventos de al menos 5 de las 6 categorías listadas.
- [ ] El estado final del par es BIG-IP-A = Active, BIG-IP-B = Standby, In Sync.
- [ ] El tráfico de producción funciona correctamente.
- [ ] La hoja de trabajo del laboratorio está completa con todas las evidencias.

## Validación y Pruebas

### Criterios de Éxito Medibles

Para considerar el laboratorio completado exitosamente, debes cumplir **todos** los siguientes criterios:

| # | Criterio | Evidencia Requerida |
|---|---|---|
| 1 | Estado Active/Standby verificado en ambos dispositivos | Salida de `tmsh show cm failover-status` de ambos BIG-IP (Paso 1) |
| 2 | Device Trust validado | Salida de `tmsh show cm trust-domain all` mostrando estado `Valid` (Paso 1) |
| 3 | Force to Standby ejecutado con continuidad de tráfico | Log del ciclo de curl mostrando recuperación en ≤3 segundos (Paso 3) |
| 4 | Force Offline diferenciado operativamente | Tabla comparativa completada correctamente (Paso 4) |
| 5 | Failover inducido y recuperado | Logs de `/var/log/ltm` con eventos de failover + restauración del estado original (Paso 5) |
| 6 | Changes Pending diagnosticado y resuelto | Salida de `tmsh show cm sync-status` antes y después de sincronización (Paso 6) |
| 7 | Sync Failed/discrepancia recuperado | Evidencia de sincronización forzada exitosa (Paso 7) |
| 8 | Logs de HA clasificados | Tabla de eventos con al menos 5 categorías documentadas (Paso 8) |

### Prueba de Validación Final

Ejecuta la siguiente secuencia de validación completa desde el **cliente de administración**:

```bash
echo "=== Validación Final del Laboratorio 6 ==="
echo ""
echo "--- Estado BIG-IP-A ---"
ssh admin@192.0.2.245 'tmsh show cm failover-status; echo "---"; tmsh show cm sync-status'
echo ""
echo "--- Estado BIG-IP-B ---"
ssh admin@192.0.2.246 'tmsh show cm failover-status; echo "---"; tmsh show cm sync-status'
echo ""
echo "--- Prueba de Tráfico ---"
curl -s -w "HTTP Code: %{http_code}\n" http://10.10.10.100
echo ""
echo "=== Fin de Validación ==="
```

**Salida esperada:**
```text
=== Validación Final del Laboratorio 6 ===

--- Estado BIG-IP-A ---
Failover status
  ACTIVE
---
Status: In Sync

--- Estado BIG-IP-B ---
Failover status
  STANDBY
---
Status: In Sync

--- Prueba de Tráfico ---
HTTP Code: 200

=== Fin de Validación ===
```

### Caso Adversarial: Validación de Integridad

Para verificar que comprendes los estados y no simplemente repites comandos, responde las siguientes preguntas en tu hoja de trabajo. Estas preguntas presentan escenarios que **no** se pueden resolver con los procedimientos estándar del laboratorio:

1. **Escenario contradictorio:** Un compañero afirma que ambos BIG-IP muestran estado `ACTIVE` simultáneamente después de un failover. ¿Es esto posible? Si sí, ¿cómo se llama esta condición y qué la causa? ¿Qué procedimiento seguirías para resolverla?

2. **Información inexistente:** Se te pide verificar el estado de Device Trust ejecutando `tmsh show cm trust-domain all`, pero el comando devuelve `Trust Domain: (empty)`. ¿Qué significa esto? ¿Puede el par de HA funcionar correctamente sin Device Trust? ¿Qué acción tomarías?

3. **Escenario de ambigüedad:** ConfigSync muestra `In Sync` pero un Virtual Server que existe en BIG-IP-A no aparece en BIG-IP-B. ¿Cómo es posible esto si el estado dice "In Sync"? Identifica al menos dos causas posibles.

> **Respuestas orientativas (para el instructor):**
> 1. Sí, es posible — se denomina *split brain*. Puede ocurrir por pérdida total de comunicación de failover. Se resuelve poniendo uno de los dispositivos en Force Offline, verificando la configuración y restaurando la comunicación de failover antes de reintegrar el dispositivo.
> 2. Un trust domain vacío indica que Device Trust no se ha establecido o se ha roto. El par HA **no** puede funcionar correctamente sin Device Trust. Se debe restablecer la relación de confianza desde **Device Management > Device Trust > Peer Authority Devices** en TMUI.
> 3. Posibles causas: (a) el Virtual Server pertenece a una partición diferente que no está incluida en la sincronización del Device Group; (b) el objeto fue creado con `tmsh` directamente en la configuración local sin guardarlo en la partición sincronizada; (c) un bug de sincronización que requiere `force-full-load-push`.

## Solución de Problemas

### Problema 1: Force to Standby no produce cambio de rol

**Síntomas:**
- Se ejecuta `tmsh run sys failover standby` en BIG-IP-A pero el estado permanece `ACTIVE`.
- BIG-IP-B no asume el rol Active.
- El comando no devuelve error explícito.

**Causa:**
BIG-IP-B no es elegible para asumir el traffic group. Las causas más comunes son:
- BIG-IP-B está en estado `FORCED OFFLINE`.
- La comunicación de failover entre los dispositivos está interrumpida (Self IPs de HA no alcanzables).
- BIG-IP-B tiene un estado de sincronización `Sync Failed` que impide la transición.

**Solución:**

1. Verifica el estado de BIG-IP-B:
   ```bash
   ssh admin@192.0.2.246 'tmsh show cm failover-status'
   ```

2. Si muestra `FORCED OFFLINE`, restaura su participación:
   ```bash
   # En BIG-IP-B
   tmsh run sys failover online
   ```

3. Verifica la conectividad de failover:
   ```bash
   # En BIG-IP-A, hacer ping al Self IP de failover de BIG-IP-B
   ping 10.10.10.246
   ```

4. Si la sincronización está fallida, resuelve primero el estado de ConfigSync (ver Paso 7 del laboratorio) antes de intentar la conmutación.

5. Reintenta la operación de Force to Standby.

---

### Problema 2: ConfigSync permanece en "Changes Pending" después de ejecutar sincronización

**Síntomas:**
- Se ejecuta `tmsh run cm config-sync to-group dg-lab-failover` pero el estado sigue mostrando `Changes Pending`.
- La TMUI muestra un indicador amarillo/naranja persistente en Device Management > Overview.
- No hay mensajes de error evidentes en la salida del comando.

**Causa:**
Posibles causas:
- Existen cambios no guardados en la configuración en ejecución (running config) que no se han persistido. El comando `config-sync` sincroniza la configuración guardada.
- Hay cambios pendientes en **ambos** dispositivos, creando un conflicto bidireccional.
- El Device Group no incluye correctamente a ambos dispositivos.

**Solución:**

1. Guarda la configuración en ejecución antes de sincronizar:
   ```bash
   # En BIG-IP-A
   tmsh save sys config
   ```

2. Verifica si hay cambios pendientes en ambos dispositivos:
   ```bash
   # En BIG-IP-A
   tmsh show cm sync-status
   # En BIG-IP-B
   tmsh show cm sync-status
   ```

3. Si ambos dispositivos muestran cambios, decide cuál es la fuente autoritativa y ejecuta la sincronización desde ese dispositivo:
   ```bash
   # Desde el dispositivo autoritativo
   tmsh run cm config-sync to-group dg-lab-failover
   ```

4. Si el problema persiste, ejecuta una sincronización completa forzada:
   ```bash
   tmsh run cm config-sync force-full-load-push to-group dg-lab-failover
   ```

5. Verifica el resultado:
   ```bash
   tmsh show cm sync-status
   ```

6. Si aún persiste, revisa los logs para identificar objetos específicos que causan el conflicto:
   ```bash
   grep -i "sync\|config-sync\|mcpd" /var/log/ltm | tail -20
   ```

## Limpieza

Ejecuta los siguientes pasos para restaurar el entorno a su estado original y eliminar los objetos de prueba creados durante el laboratorio.

**En BIG-IP-A (debe estar Active):**

```bash
## Verificar que BIG-IP-A es Active
tmsh show cm failover-status

## Eliminar los pools de prueba creados durante el laboratorio
tmsh delete ltm pool pool_test_sync
tmsh delete ltm pool pool_conflict

## Guardar la configuración
tmsh save sys config

## Sincronizar la eliminación al peer
tmsh run cm config-sync to-group dg-lab-failover
```

**Verificación de limpieza:**

```bash
## En BIG-IP-A
tmsh list ltm pool pool_test_sync 2>&1
tmsh list ltm pool pool_conflict 2>&1
## Ambos comandos deben devolver error (pools no encontrados)

## Verificar sincronización
tmsh show cm sync-status
## Debe mostrar: In Sync
```

**En BIG-IP-B, verificar que los pools fueron eliminados:**

```bash
tmsh list ltm pool pool_test_sync 2>&1
tmsh list ltm pool pool_conflict 2>&1
## Ambos deben devolver error

tmsh show cm sync-status
## Debe mostrar: In Sync
```

**Verificación final del estado del entorno:**

```bash
## En BIG-IP-A
tmsh show cm failover-status
## Esperado: ACTIVE

## En BIG-IP-B
tmsh show cm failover-status
## Esperado: STANDBY

## Tráfico de producción
curl -s -o /dev/null -w "%{http_code}" http://10.10.10.100
## Esperado: 200
```

> **Nota:** La segunda instancia BIG-IP-B será retirada del entorno de laboratorio al finalizar este módulo. Los laboratorios posteriores regresan a la topología persistente de un solo BIG-IP VE.

## Resumen

En este laboratorio aplicaste los procedimientos operativos fundamentales para administrar un par BIG-IP en alta disponibilidad Active/Standby:

- **Exploración de estado:** Verificaste el estado de failover, Device Trust, Device Groups y ConfigSync utilizando tanto TMUI como TMSH, confirmando la coherencia entre ambas interfaces.
- **Force to Standby vs Force Offline:** Ejecutaste una conmutación controlada con Force to Standby y observaste que el dispositivo permanece elegible para failover, a diferencia de Force Offline que lo excluye completamente hasta intervención administrativa.
- **Failover inducido:** Simulaste una falla de red deshabilitando una interfaz, observaste la transición automática de roles y ejecutaste la recuperación completa, incluyendo la restauración manual del estado original.
- **ConfigSync — Changes Pending:** Generaste una discrepancia de configuración deliberada, interpretaste el indicador de sincronización y ejecutaste la sincronización manual en la dirección correcta.
- **ConfigSync — Sync Failed / Discrepancia:** Creaste un conflicto de configuración entre dispositivos, diagnosticaste la causa mediante logs y aplicaste sincronización forzada desde la fuente autoritativa.
- **Análisis de logs:** Revisaste y clasificaste eventos de `/var/log/ltm` para documentar cada fase de la operación de HA.

### Conceptos Clave Reforzados

| Concepto | Aplicación en el Laboratorio |
|---|---|
| Active/Standby | Verificación de roles y transiciones controladas |
| Device Trust | Validación de relación criptográfica entre peers |
| ConfigSync | Diagnóstico y resolución de estados In Sync, Changes Pending y Sync Failed |
| Force to Standby | Conmutación planificada con continuidad de servicio |
| Force Offline | Exclusión administrativa de la elección de failover |
| Traffic Group | Identificación de propietario y migración durante failover |

### Recursos Adicionales

| Recurso | URL |
|---|---|
| F5 Docs: Device Service Clustering Administration | [https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/big-ip-device-service-clustering-admin.html](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/big-ip-device-service-clustering-admin.html) |
| F5 Docs: ConfigSync Overview | [https://my.f5.com/manage/s/article/K13946](https://my.f5.com/manage/s/article/K13946) |
| F5 Docs: Troubleshooting ConfigSync | [https://my.f5.com/manage/s/article/K13920](https://my.f5.com/manage/s/article/K13920) |
| F5 Docs: Force to Standby and Force Offline | [https://my.f5.com/manage/s/article/K14051](https://my.f5.com/manage/s/article/K14051) |
| F5 BIG-IP VE 17.1.1.3 Release Notes | [https://my.f5.com/manage/s/article/K000137353](https://my.f5.com/manage/s/article/K000137353) |

---

# Laboratorio 7 — Administración y recuperación del Control Plane: UCS, usuarios, servicios, logs, estado del sistema y validación de servicios

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Módulo** | 4 — Administración del Control Plane |
| **Alineación** | F5CAB4.03, F5CAB4.04, F5CAB4.05, F5CAB4.06, F5CAB4.07, F5CAB4.10 |
| **Plataforma** | F5 BIG-IP Virtual Edition 17.1.2 |

## Descripción General

Este laboratorio consolida las competencias de administración del Control Plane de BIG-IP en una secuencia integrada de siete actividades. El estudiante revisará el estado del sistema y la alta disponibilidad, creará y validará respaldos UCS, gestionará usuarios locales con distintos niveles de privilegio, configurará servicios del sistema (NTP, DNS, Syslog, SNMP), analizará logs para identificar eventos y niveles de severidad, validará puertos y servicios en escucha del Control Plane y ejecutará un procedimiento de restauración UCS con verificación posterior. Todas las actividades se realizan sobre la topología persistente del curso: 1 BIG-IP VE, 1 cliente y 2 servidores web.

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Crear un archivo UCS completo desde GUI y TMSH, verificar su contenido y transferirlo de forma segura a la estación del estudiante.
- [ ] Gestionar usuarios locales en BIG-IP con diferentes roles, verificar su acceso y restringir privilegios según las necesidades operativas.
- [ ] Configurar y validar los servicios del sistema DNS, NTP, Syslog y SNMP, confirmando su operatividad mediante TMSH y GUI.
- [ ] Analizar los archivos de log `/var/log/ltm`, `/var/log/secure` y `/var/log/audit` para identificar eventos del sistema e interpretar niveles de severidad.
- [ ] Validar los servicios activos y puertos en escucha del Control Plane mediante `netstat` y TMSH, mapeando procesos a puertos asociados.

## Prerrequisitos

### Conocimientos requeridos

- Fundamentos de TCP/IP, direccionamiento IP y subredes.
- Operación básica de línea de comandos Linux (navegación de directorios, lectura de archivos, `grep`, `tail`).
- Familiaridad con TMSH: sintaxis de comandos `show`, `list`, `modify`, `create`, `save`, `load`.
- Comprensión de la arquitectura Active/Standby de BIG-IP, Device Trust y ConfigSync (Lección 4.1 del curso).
- Haber completado los Laboratorios 1–6 del Módulo 4, o contar con el entorno en estado equivalente preconfigurado.

### Acceso requerido

| Recurso | Dirección / Credenciales |
|---|---|
| BIG-IP VE Management IP | `192.0.2.245` — usuario: `admin`, contraseña: proporcionada por el instructor |
| Cliente de administración (SSH/SCP) | `192.0.2.10` — acceso SSH al BIG-IP y capacidad SCP |
| Cliente de pruebas (navegador/curl) | `10.10.10.10` — conectividad a VLANs External e Internal |
| Servidor NTP interno | `192.0.2.10` (el propio cliente de administración actúa como fuente NTP del laboratorio) |
| Servidor DNS interno | `192.0.2.10` |
| Servidor Syslog remoto | `192.0.2.10` puerto UDP/514 (syslog-ng 4.7.1 preconfigurado en el cliente de administración) |

> **Nota sobre la topología**: La topología persistente del curso dispone de **un único BIG-IP VE**. Los contenidos de alta disponibilidad (Device Trust, failover, ConfigSync) se trabajan como **análisis técnico y verificación de estado**, no como configuración real de un segundo dispositivo. Los comandos HA se ejecutan para interpretar la salida de un dispositivo standalone y comprender qué indicaría cada estado en un par real.

## Entorno de Laboratorio

### Topología de red

```
┌──────────────┐    Management: 192.0.2.0/24    ┌──────────────────┐
│   Cliente     │◄──────────────────────────────►│   BIG-IP VE      │
│ 192.0.2.10   │    External: 10.10.10.0/24     │ Mgmt: 192.0.2.245│
│ 10.10.10.10  │◄──────────────────────────────►│ Ext:  10.10.10.245│
└──────────────┘                                 │ Int:  10.20.20.245│
                                                 └────────┬─────────┘
                              Internal: 10.20.20.0/24     │
                         ┌─────────────────────────────────┤
                         │                                 │
                  ┌──────┴──────┐                  ┌───────┴─────┐
                  │   web01     │                  │   web02     │
                  │10.20.20.11  │                  │10.20.20.12  │
                  └─────────────┘                  └─────────────┘
```

### Software del laboratorio

| Componente | Versión exacta | Fuente oficial |
|---|---|---|
| F5 BIG-IP VE | 17.1.2 | [https://my.f5.com/manage/s/downloads](https://my.f5.com/manage/s/downloads) |
| Ubuntu Server (cliente y servidores web) | 22.04.4 LTS | [https://releases.ubuntu.com/22.04/](https://releases.ubuntu.com/22.04/) |
| Apache HTTP Server (web01, web02) | 2.4.58 | [https://httpd.apache.org/download.cgi](https://httpd.apache.org/download.cgi) |
| OpenSSH (cliente) | 9.6p1 | [https://www.openssh.com/releasenotes.html](https://www.openssh.com/releasenotes.html) |
| curl (cliente) | 8.5.0 | [https://curl.se/download.html](https://curl.se/download.html) |
| syslog-ng (cliente, servidor de logs) | 4.7.1 | [https://github.com/syslog-ng/syslog-ng/releases](https://github.com/syslog-ng/syslog-ng/releases) |

### Verificación previa del entorno

Desde el **cliente de administración** (`192.0.2.10`), ejecutar antes de iniciar:

```bash
## Verificar conectividad SSH al BIG-IP
ssh admin@192.0.2.245 'tmsh show sys version' 

## Verificar conectividad HTTPS (debe devolver HTTP 200 o 302)
curl -sk -o /dev/null -w "%{http_code}" https://192.0.2.245

## Verificar que los servidores web responden
curl -s -o /dev/null -w "%{http_code}" http://10.20.20.11
curl -s -o /dev/null -w "%{http_code}" http://10.20.20.12
```

**Resultados esperados**: el comando SSH debe mostrar la versión `17.1.2`; curl a HTTPS debe devolver `200` o `302`; curl a los servidores web debe devolver `200`.

## Instrucciones Paso a Paso

### Paso 1: Revisión del estado inicial del sistema

**Objetivo**: Verificar el estado del BIG-IP, interpretar la información de failover, Device Trust y sincronización, y revisar el estado de los objetos del Data Plane desde el Dashboard y TMSH.

**Instrucciones**:

1. Abrir una sesión SSH al BIG-IP desde el cliente de administración:

   ```bash
   ssh admin@192.0.2.245
   ```

2. Verificar el estado de failover del dispositivo:

   ```bash
   tmsh show sys failover
   ```

3. Consultar el estado de Device Trust:

   ```bash
   tmsh show cm trust-domain
   ```

4. Revisar el estado de sincronización de configuración:

   ```bash
   tmsh show cm sync-status
   ```

5. Verificar los dispositivos conocidos y su estado:

   ```bash
   tmsh show cm device
   ```

6. Consultar los traffic groups configurados y su estado de propiedad:

   ```bash
   tmsh list cm traffic-group
   tmsh show cm traffic-group
   ```

7. Revisar el estado de los objetos del Data Plane desde TMSH:

   ```bash
   tmsh show ltm virtual
   tmsh show ltm pool
   tmsh show ltm node
   ```

8. Abrir un navegador web en el cliente de administración y acceder a `https://192.0.2.245`. Iniciar sesión con las credenciales de administrador.

9. En TMUI, navegar a **System > Dashboard**. Observar y documentar:
   - Estado general del dispositivo (indicador de color en la esquina superior izquierda).
   - Información de HA (sección de failover).
   - Uso de CPU, memoria y disco.

10. Navegar a **Local Traffic > Network Map**. Verificar el estado visual de los Virtual Servers, Pools y Nodes (verde = disponible, rojo = no disponible, azul = desconocido).

11. Navegar a **Device Management > Devices**. Observar el dispositivo local, su estado de failover y la información de Device Trust.

**Salida esperada**:

Para `tmsh show sys failover`, en un dispositivo standalone la salida típica es:

```text
Failover Status
  ACTIVE
```

Para `tmsh show cm trust-domain`, en un dispositivo standalone:

```text
-----------------------------------------
Cm::Trust Domain
Name                    Root
Status                  standalone
...
```

Para `tmsh show cm sync-status`, en un dispositivo standalone:

```text
-----------------------------------------
Color   green
Mode    standalone
...
Status  Standalone
```

Para `tmsh show ltm virtual`, debe aparecer al menos un Virtual Server con estado `available` (indicado por un campo `Availability` con valor `available`).

**Verificación**:

- [ ] El estado de failover muestra `ACTIVE`.
- [ ] El trust-domain muestra `standalone` (confirmando la topología de un solo dispositivo).
- [ ] El sync-status muestra `Standalone` con color `green`.
- [ ] Al menos un Virtual Server, un Pool y un Node aparecen en estado operativo.
- [ ] El Dashboard de TMUI muestra el dispositivo sin alertas críticas.
- [ ] El Network Map muestra los objetos con indicadores visuales verdes.

> **Análisis HA**: En un par Active/Standby real, `tmsh show cm trust-domain` mostraría ambos dispositivos con estado `Peer Authority Device`. El `sync-status` indicaría `In Sync` o `Changes Pending`. Documenta en tu cuaderno qué valores esperarías ver en cada comando si existiera un segundo BIG-IP (`bigip-b`) en estado Standby.

---

### Paso 2: Creación y verificación de respaldo UCS

**Objetivo**: Crear un archivo UCS (User Configuration Set) completo desde GUI y TMSH, verificar su contenido y transferirlo de forma segura al cliente de administración.

**Instrucciones**:

1. **Crear UCS desde TMUI**: En el navegador, navegar a **System > Archives**. Hacer clic en **Create**. En el campo **Name**, escribir `lab7-backup-gui`. Dejar las opciones predeterminadas (incluir claves privadas). Hacer clic en **Finished**. Esperar a que el proceso complete.

2. **Crear UCS desde TMSH**: En la sesión SSH al BIG-IP, ejecutar:

   ```bash
   tmsh save sys ucs /var/local/ucs/lab7-backup.ucs
   ```

3. Verificar que ambos archivos UCS existen:

   ```bash
   ls -lh /var/local/ucs/lab7-backup*.ucs
   ```

4. Examinar el contenido del archivo UCS (es un archivo tar comprimido con gzip):

   ```bash
   tar tzf /var/local/ucs/lab7-backup.ucs | head -40
   ```

5. Verificar que el archivo contiene los componentes críticos de configuración:

   ```bash
   tar tzf /var/local/ucs/lab7-backup.ucs | grep -E "bigip.conf|bigip_base.conf|bigip_user.conf|ssl.key|ssl.crt"
   ```

6. Verificar el tamaño y la integridad básica del archivo:

   ```bash
   file /var/local/ucs/lab7-backup.ucs
   md5sum /var/local/ucs/lab7-backup.ucs
   ```

7. **Transferir el archivo UCS al cliente de administración**. Desde el **cliente de administración** (`192.0.2.10`), ejecutar:

   ```bash
   mkdir -p ~/ucs-backups
   scp admin@192.0.2.245:/var/local/ucs/lab7-backup.ucs ~/ucs-backups/
   ```

8. Verificar la transferencia en el cliente:

   ```bash
   ls -lh ~/ucs-backups/lab7-backup.ucs
   md5sum ~/ucs-backups/lab7-backup.ucs
   ```

**Salida esperada**:

Para el comando `tar tzf`, la salida debe incluir rutas como:

```text
var/local/ucs/
config/bigip.conf
config/bigip_base.conf
config/bigip_user.conf
config/ssl/ssl.key/
config/ssl/ssl.crt/
config/partitions/
...
```

Para `file`, la salida debe indicar:

```text
/var/local/ucs/lab7-backup.ucs: gzip compressed data, ...
```

Los valores `md5sum` deben coincidir entre el BIG-IP y el cliente tras la transferencia SCP.

**Verificación**:

- [ ] El archivo `lab7-backup-gui.ucs` aparece en **System > Archives** en TMUI.
- [ ] El archivo `lab7-backup.ucs` existe en `/var/local/ucs/` con tamaño mayor a 0 bytes.
- [ ] El contenido del UCS incluye `bigip.conf`, `bigip_base.conf` y directorios de certificados/claves.
- [ ] La suma MD5 coincide entre el BIG-IP y el cliente de administración.
- [ ] El archivo fue transferido exitosamente al directorio `~/ucs-backups/` del cliente.

> **Implicación de seguridad**: El archivo UCS contiene claves privadas SSL/TLS, credenciales de usuario cifradas y la master key del dispositivo. Debe tratarse con el mismo nivel de protección que las credenciales de administrador. En un entorno de producción, el almacenamiento del UCS debe estar cifrado y el acceso restringido. La opción `passphrase` de TMSH permite cifrar el UCS con una contraseña adicional: `tmsh save sys ucs /var/local/ucs/lab7-encrypted.ucs passphrase <frase>`.

---

### Paso 3: Gestión de usuarios y control de acceso

**Objetivo**: Crear usuarios locales con diferentes roles, verificar el acceso con cada nivel de privilegio y restringir el acceso SSH a usuarios operativos.

**Instrucciones**:

1. **Crear usuario con rol Operator desde TMSH**:

   ```bash
   tmsh create auth user lab7-operator password 'Op3r@t0r2024!' partition-access add { all-partitions { role operator } }
   ```

2. **Crear usuario con rol Administrator desde TMUI**: Navegar a **System > Users > User List**. Hacer clic en **Create**. Configurar:
   - **User Name**: `lab7-admin`
   - **Password**: `Adm1n2024!`
   - **Confirm Password**: `Adm1n2024!`
   - **Role**: `Administrator`
   - **Partition Access**: `All Partitions`
   
   Hacer clic en **Finished**.

3. Verificar que ambos usuarios existen:

   ```bash
   tmsh list auth user lab7-operator
   tmsh list auth user lab7-admin
   ```

4. **Probar acceso SSH con el usuario Operator** desde el cliente de administración:

   ```bash
   ssh lab7-operator@192.0.2.245
   ```

   Una vez conectado (si el shell lo permite), intentar ejecutar un comando TMSH privilegiado:

   ```bash
   tmsh show sys failover
   tmsh modify sys global-settings gui-setup disabled
   ```

   Observar qué comandos son permitidos y cuáles son denegados. Salir de la sesión:

   ```bash
   exit
   ```

5. **Probar acceso TMUI con el usuario Operator**: Abrir una ventana de navegador en modo privado/incógnito. Acceder a `https://192.0.2.245`. Iniciar sesión con `lab7-operator` / `Op3r@t0r2024!`. Observar las opciones de menú disponibles y las restricciones del rol Operator.

6. **Probar acceso SSH con el usuario Administrator**:

   ```bash
   ssh lab7-admin@192.0.2.245
   ```

   Ejecutar:

   ```bash
   tmsh show sys failover
   tmsh list auth user
   exit
   ```

7. **Restringir acceso SSH del usuario Operator** modificando su shell:

   ```bash
   tmsh modify auth user lab7-operator shell /sbin/nologin
   ```

8. Verificar la restricción intentando conectar por SSH:

   ```bash
   ssh lab7-operator@192.0.2.245
   ```

9. Confirmar que el acceso TMUI del usuario Operator sigue funcionando (repetir paso 5).

10. Revisar los proveedores de autenticación configurados:

    ```bash
    tmsh list auth source
    ```

**Salida esperada**:

Para `tmsh list auth user lab7-operator`:

```text
auth user lab7-operator {
    description "lab7-operator"
    encrypted-password $6$...
    partition-access {
        all-partitions {
            role operator
        }
    }
    shell /sbin/nologin
}
```

El intento de SSH con `lab7-operator` después de cambiar el shell debe producir:

```text
This account is currently not available.
Connection to 192.0.2.245 closed.
```

El usuario `lab7-admin` debe poder ejecutar todos los comandos TMSH sin restricción.

Para `tmsh list auth source`:

```text
auth source {
    type local
    ...
}
```

**Verificación**:

- [ ] El usuario `lab7-operator` existe con rol `operator` en todas las particiones.
- [ ] El usuario `lab7-admin` existe con rol `administrator` en todas las particiones.
- [ ] El usuario Operator tiene acceso limitado en TMSH (solo lectura de ciertos objetos, sin capacidad de modificación de configuración).
- [ ] El usuario Operator puede acceder a TMUI con menús restringidos.
- [ ] Tras el cambio de shell a `/sbin/nologin`, el usuario Operator no puede iniciar sesión SSH.
- [ ] El usuario Operator mantiene acceso funcional a TMUI tras la restricción de SSH.
- [ ] El proveedor de autenticación es `local`.

---

### Paso 4: Configuración y validación de servicios del sistema

**Objetivo**: Configurar y verificar los servicios NTP, DNS, Syslog remoto y SNMP del BIG-IP, confirmando su operatividad desde TMSH y GUI.

**Instrucciones**:

1. **Configurar el servidor NTP**:

   ```bash
   tmsh modify sys ntp servers add { 192.0.2.10 }
   ```

2. Verificar la configuración NTP:

   ```bash
   tmsh list sys ntp
   ```

3. Validar la sincronización NTP (puede tardar unos segundos en establecerse):

   ```bash
   tmsh show sys ntp
   ```

   Alternativamente, verificar con `ntpq`:

   ```bash
   ntpq -pn
   ```

4. **Configurar el servidor DNS del sistema**:

   ```bash
   tmsh modify sys dns name-servers add { 192.0.2.10 }
   ```

5. Verificar la configuración DNS:

   ```bash
   tmsh list sys dns
   ```

6. Probar la resolución DNS desde el BIG-IP:

   ```bash
   dig @192.0.2.10 f5.com +short
   nslookup f5.com 192.0.2.10
   ```

7. **Configurar el reenvío de Syslog a servidor remoto**:

   ```bash
   tmsh modify sys syslog remote-servers add { syslog-lab { host 192.0.2.10 remote-port 514 } }
   ```

8. Verificar la configuración de Syslog:

   ```bash
   tmsh list sys syslog remote-servers
   ```

9. Generar un evento de prueba para validar el reenvío de Syslog:

   ```bash
   logger -p local0.notice "BIG-IP Lab7 syslog test message from $(hostname)"
   ```

10. En el **cliente de administración** (`192.0.2.10`), verificar que el mensaje fue recibido por syslog-ng:

    ```bash
    sudo tail -5 /var/log/syslog | grep "Lab7"
    ```

11. **Verificar la configuración SNMP básica**:

    ```bash
    tmsh list sys snmp
    tmsh list sys snmp allowed-addresses
    tmsh list sys snmp communities
    ```

12. Guardar todos los cambios de configuración:

    ```bash
    tmsh save sys config
    ```

13. **Verificar desde TMUI**: Navegar a **System > Configuration > Device > NTP**. Confirmar que el servidor `192.0.2.10` aparece listado. Navegar a **System > Configuration > Device > DNS**. Confirmar el servidor DNS. Navegar a **System > Logs > Configuration > Remote Logging**. Confirmar el servidor Syslog remoto.

**Salida esperada**:

Para `tmsh list sys ntp`:

```text
sys ntp {
    servers { 192.0.2.10 }
    ...
}
```

Para `tmsh show sys ntp`, la salida debe mostrar el servidor con información de estrato y offset (valores numéricos indican comunicación activa):

```text
--------------------------------------------------
Sys::NTP
  Server   192.0.2.10
    Status   ...
    Stratum  ...
    ...
```

Para `tmsh list sys syslog remote-servers`:

```text
sys syslog {
    remote-servers {
        syslog-lab {
            host 192.0.2.10
            remote-port 514
        }
    }
}
```

> **Nota**: Si el servidor NTP del laboratorio no está activo, `ntpq -pn` puede mostrar el servidor con estado `.INIT.` o sin respuesta. Esto es aceptable en el contexto del laboratorio; lo importante es verificar que la configuración fue aplicada correctamente. Lo mismo aplica para DNS si no hay un resolvedor real en `192.0.2.10`.

**Verificación**:

- [ ] El servidor NTP `192.0.2.10` está configurado en `sys ntp`.
- [ ] `tmsh show sys ntp` muestra el servidor (el estado puede variar según la disponibilidad real del servicio NTP).
- [ ] El servidor DNS `192.0.2.10` está configurado en `sys dns`.
- [ ] El servidor Syslog remoto `syslog-lab` está configurado con host `192.0.2.10` y puerto `514`.
- [ ] La configuración SNMP muestra las comunidades y direcciones permitidas actuales.
- [ ] Los cambios fueron guardados con `tmsh save sys config`.
- [ ] Las configuraciones son visibles en las secciones correspondientes de TMUI.

---

### Paso 5: Análisis de logs del sistema

**Objetivo**: Analizar los archivos de log `/var/log/ltm`, `/var/log/secure` y `/var/log/audit` para identificar eventos del sistema, intentos de autenticación y cambios de configuración, interpretando los niveles de severidad.

**Instrucciones**:

1. **Analizar `/var/log/ltm`** — eventos del plano de datos y del sistema LTM:

   ```bash
   tail -50 /var/log/ltm
   ```

2. Buscar eventos específicos relacionados con pools, monitores y cambios de estado:

   ```bash
   grep -i "pool\|monitor\|node\|mcpd" /var/log/ltm | tail -20
   ```

3. Identificar los niveles de severidad presentes en los eventos. Los niveles estándar de syslog son:

   | Nivel | Código | Descripción |
   |---|---|---|
   | Emergency | 0 | Sistema inutilizable |
   | Alert | 1 | Acción inmediata requerida |
   | Critical | 2 | Condiciones críticas |
   | Error | 3 | Condiciones de error |
   | Warning | 4 | Condiciones de advertencia |
   | Notice | 5 | Condición normal pero significativa |
   | Informational | 6 | Mensajes informativos |
   | Debug | 7 | Mensajes de depuración |

   Buscar eventos con diferentes niveles:

   ```bash
   grep -iE "err|warning|notice|crit|alert" /var/log/ltm | tail -15
   ```

4. **Analizar `/var/log/secure`** — eventos de autenticación:

   ```bash
   tail -30 /var/log/secure
   ```

5. Buscar los intentos de autenticación de los usuarios creados en el Paso 3:

   ```bash
   grep -i "lab7" /var/log/secure
   ```

6. Identificar intentos exitosos y fallidos:

   ```bash
   grep -i "accepted\|failed\|invalid" /var/log/secure | tail -15
   ```

7. **Analizar `/var/log/audit`** — registro de cambios de configuración:

   ```bash
   tail -40 /var/log/audit
   ```

8. Buscar los cambios realizados durante este laboratorio (creación de usuarios, modificación de NTP, DNS, Syslog):

   ```bash
   grep -iE "lab7|ntp|dns|syslog|user" /var/log/audit | tail -20
   ```

9. Identificar qué usuario administrativo realizó cada cambio y la marca de tiempo:

   ```bash
   grep "admin" /var/log/audit | tail -10
   ```

10. **Consultar logs desde TMUI**: Navegar a **System > Logs > Local Traffic**. Revisar los eventos recientes. Navegar a **System > Logs > Audit**. Revisar los cambios de configuración registrados.

11. Documentar al menos **tres eventos** encontrados, indicando para cada uno:
    - Archivo de log donde se encontró.
    - Marca de tiempo.
    - Nivel de severidad.
    - Descripción del evento.
    - Relevancia operativa.

**Salida esperada**:

Ejemplo de entrada en `/var/log/ltm`:

```text
Jun 15 10:23:45 bigip notice mcpd[6789]: 01070727:5: Pool /Common/web_pool member /Common/web01:80 monitor status up. ...
```

Ejemplo de entrada en `/var/log/secure`:

```text
Jun 15 10:30:12 bigip sshd[12345]: Accepted password for lab7-admin from 192.0.2.10 port 54321 ssh2
Jun 15 10:31:05 bigip sshd[12346]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 ... user=lab7-operator
```

Ejemplo de entrada en `/var/log/audit`:

```text
Jun 15 10:28:00 bigip notice mcpd[6789]: 01420002:5: AUDIT - user admin - RAW: tmsh create auth user lab7-operator ...
```

**Verificación**:

- [ ] Se identificaron eventos en `/var/log/ltm` relacionados con objetos LTM (pools, nodes, monitores).
- [ ] Se encontraron registros de autenticación en `/var/log/secure` correspondientes a los usuarios `lab7-operator` y `lab7-admin`.
- [ ] Se identificó al menos un intento de acceso denegado (el SSH rechazado de `lab7-operator` tras el cambio de shell).
- [ ] Se encontraron entradas de auditoría en `/var/log/audit` correspondientes a los cambios de configuración realizados en pasos anteriores.
- [ ] Se documentaron al menos tres eventos con su archivo de origen, marca de tiempo, nivel de severidad y descripción.
- [ ] Se identificaron correctamente al menos dos niveles de severidad diferentes en los logs analizados.

---

### Paso 6: Validación de puertos y servicios del Control Plane

**Objetivo**: Identificar todos los puertos en escucha del Control Plane de BIG-IP, mapear cada puerto a su servicio correspondiente y verificar el estado de los servicios del sistema.

**Instrucciones**:

1. Listar todos los puertos TCP y UDP en escucha en el BIG-IP:

   ```bash
   netstat -tulnp
   ```

2. Filtrar únicamente los puertos TCP en estado LISTEN:

   ```bash
   netstat -tlnp
   ```

3. Filtrar los puertos UDP en escucha:

   ```bash
   netstat -ulnp
   ```

4. Identificar y documentar los servicios del Control Plane mapeando puertos a procesos. Completar la siguiente tabla con los resultados observados:

   | Puerto | Protocolo | Proceso | Servicio |
   |---|---|---|---|
   | 22 | TCP | sshd | SSH — acceso remoto administrativo |
   | 443 | TCP | httpd | TMUI — interfaz web de administración |
   | 4353 | TCP | named/iquery | iQuery — comunicación entre dispositivos HA |
   | 161 | UDP | snmpd | SNMP — monitoreo de red |
   | 123 | UDP | ntpd/chronyd | NTP — sincronización de tiempo |
   | 53 | TCP/UDP | named | DNS — resolución de nombres (si GTM/DNS provisionado) |
   | 514 | UDP | syslog-ng | Syslog — recepción de logs (si configurado) |

   > **Nota**: Los puertos exactos y procesos pueden variar según los módulos provisionados y la configuración específica. Documenta lo que observes en tu instancia.

5. Verificar el estado de los servicios del sistema mediante TMSH:

   ```bash
   tmsh show sys service
   ```

6. Verificar servicios específicos:

   ```bash
   tmsh show sys service sshd
   tmsh show sys service httpd
   tmsh show sys service snmpd
   tmsh show sys service ntpd
   tmsh show sys service named
   ```

7. Verificar el estado de los procesos TMM (Traffic Management Microkernel):

   ```bash
   tmsh show sys tmm-info
   ```

8. Verificar los puertos de escucha configurados administrativamente en el plano de gestión:

   ```bash
   tmsh list sys httpd allow
   tmsh list sys sshd allow
   ```

9. Confirmar que las ACL de SSHD y HTTPD restringen el acceso a `192.0.2.10`:

   ```bash
   tmsh list sys httpd allow
   tmsh list sys sshd allow
   ```

   La salida esperada debe mostrar:

   ```text
   sys httpd {
       allow { 192.0.2.10 }
   }
   ```

   Si no está restringido, documentar el estado actual. **No modificar** las ACL en este paso; solo verificar.

10. Identificar si hay servicios inactivos o detenidos y documentar su estado:

    ```bash
    tmsh show sys service all-properties
    ```

**Salida esperada**:

Para `netstat -tlnp` (extracto):

```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1234/sshd
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      5678/httpd
tcp        0      0 127.0.0.1:4353          0.0.0.0:*               LISTEN      9012/named
...
```

Para `tmsh show sys service sshd`:

```text
-----------------------------------------
Sys::Service
  Service Name  sshd
  Status        run
  ...
```

**Verificación**:

- [ ] Se ejecutó `netstat -tulnp` y se identificaron al menos 5 puertos en escucha del Control Plane.
- [ ] Se completó la tabla de mapeo puerto-proceso-servicio con los datos reales observados.
- [ ] Los servicios `sshd`, `httpd`, `snmpd` y `ntpd` aparecen con estado `run` en `tmsh show sys service`.
- [ ] Se verificaron las ACL de `sys httpd` y `sys sshd`.
- [ ] Se documentaron los servicios activos e inactivos del sistema.
- [ ] Se identificó el proceso TMM y su estado operativo.

---

### Paso 7: Restauración UCS y validación final

**Objetivo**: Ejecutar el procedimiento de restauración UCS, verificar la integridad del sistema post-restauración y confirmar que la configuración coincide con el estado previo al respaldo.

**Instrucciones**:

1. **Documentar el estado actual** antes de la restauración para tener un punto de comparación. Ejecutar y guardar la salida de:

   ```bash
   tmsh show sys failover
   tmsh list auth user lab7-operator
   tmsh list auth user lab7-admin
   tmsh list sys ntp
   tmsh list sys dns
   tmsh list sys syslog remote-servers
   tmsh show ltm virtual
   tmsh show ltm pool
   ```

   Redirigir la salida a un archivo de referencia:

   ```bash
   tmsh show sys failover > /var/tmp/pre-restore-state.txt
   tmsh list auth user >> /var/tmp/pre-restore-state.txt
   tmsh list sys ntp >> /var/tmp/pre-restore-state.txt
   tmsh list sys dns >> /var/tmp/pre-restore-state.txt
   tmsh show ltm virtual >> /var/tmp/pre-restore-state.txt
   ```

2. Verificar que el archivo UCS de respaldo está disponible:

   ```bash
   ls -lh /var/local/ucs/lab7-backup.ucs
   ```

3. **Ejecutar la restauración UCS**:

   > **ADVERTENCIA**: La restauración UCS reemplaza la configuración completa del dispositivo. En un entorno de producción, esto debe realizarse durante una ventana de mantenimiento aprobada. En este laboratorio, el procedimiento es seguro porque se restaura el respaldo creado en el Paso 2, que corresponde al estado previo a la creación de usuarios y servicios adicionales.

   ```bash
   tmsh load sys ucs /var/local/ucs/lab7-backup.ucs
   ```

4. Esperar a que el proceso de restauración complete. El sistema puede tardar entre 30 segundos y 2 minutos. Observar los mensajes de la consola.

5. **Verificar el estado post-restauración** ejecutando los mismos comandos del Paso 1:

   ```bash
   tmsh show sys failover
   tmsh show cm trust-domain
   tmsh show cm sync-status
   ```

6. Verificar el estado de los objetos del Data Plane:

   ```bash
   tmsh show ltm virtual
   tmsh show ltm pool
   tmsh show ltm node
   ```

7. Verificar si los usuarios creados en el Paso 3 **persisten o fueron eliminados** por la restauración (depende de si fueron creados antes o después del UCS):

   ```bash
   tmsh list auth user lab7-operator
   tmsh list auth user lab7-admin
   ```

   > **Análisis crítico**: El archivo UCS fue creado **antes** de crear los usuarios `lab7-operator` y `lab7-admin` y antes de configurar NTP, DNS y Syslog. Por lo tanto, la restauración debería **eliminar** estos objetos, devolviendo el sistema al estado exacto del momento del respaldo. Este es el comportamiento esperado y demuestra que UCS es una restauración completa, no incremental.

8. Verificar si los servicios del sistema (NTP, DNS, Syslog) fueron revertidos:

   ```bash
   tmsh list sys ntp
   tmsh list sys dns
   tmsh list sys syslog remote-servers
   ```

9. Verificar la conectividad de los servicios del Data Plane desde el cliente de pruebas:

   ```bash
   # Desde el cliente de pruebas (10.10.10.10)
   curl -s -o /dev/null -w "%{http_code}" http://10.10.10.100
   ```

10. Comparar el estado post-restauración con el archivo de referencia pre-restauración:

    ```bash
    tmsh show sys failover > /var/tmp/post-restore-state.txt
    tmsh list auth user >> /var/tmp/post-restore-state.txt
    tmsh list sys ntp >> /var/tmp/post-restore-state.txt
    tmsh list sys dns >> /var/tmp/post-restore-state.txt
    tmsh show ltm virtual >> /var/tmp/post-restore-state.txt
    ```

    Comparar ambos archivos:

    ```bash
    diff /var/tmp/pre-restore-state.txt /var/tmp/post-restore-state.txt
    ```

**Salida esperada**:

La restauración UCS debe producir un mensaje similar a:

```text
Saving active configuration...
/var/local/ucs/lab7-backup.ucs is restored.
```

Los usuarios `lab7-operator` y `lab7-admin` **no deberían existir** tras la restauración (fueron creados después del respaldo):

```text
The requested user (/Common/lab7-operator) was not found.
```

Los servicios NTP, DNS y Syslog deberían reflejar la configuración que existía al momento de crear el UCS (sin los servidores añadidos en el Paso 4, a menos que ya estuvieran configurados previamente).

El `diff` debe mostrar diferencias en las secciones de usuarios y servicios del sistema, confirmando que la restauración revirtió los cambios.

**Verificación**:

- [ ] La restauración UCS completó sin errores.
- [ ] El estado de failover post-restauración es `ACTIVE`.
- [ ] Los objetos del Data Plane (Virtual Servers, Pools, Nodes) están operativos.
- [ ] Los usuarios `lab7-operator` y `lab7-admin` **no existen** (confirmando la restauración completa).
- [ ] La configuración de NTP, DNS y Syslog refleja el estado del momento del respaldo.
- [ ] El Virtual Server responde correctamente desde el cliente de pruebas (`HTTP 200`).
- [ ] El `diff` entre estados pre y post-restauración muestra las diferencias esperadas.

## Validación y Pruebas

### Criterios de éxito del laboratorio

Para considerar el laboratorio completado exitosamente, el estudiante debe cumplir **todos** los siguientes criterios:

| # | Criterio | Evidencia requerida |
|---|---|---|
| 1 | Archivo UCS creado y transferido | Archivo `lab7-backup.ucs` presente en `~/ucs-backups/` del cliente con MD5 coincidente |
| 2 | Usuarios gestionados correctamente | Salida de `tmsh list auth user` mostrando ambos usuarios con roles correctos (pre-restauración) |
| 3 | Restricción de acceso SSH verificada | Mensaje de rechazo SSH para `lab7-operator` tras cambio de shell |
| 4 | Servicios del sistema configurados | Salida de `tmsh list sys ntp`, `tmsh list sys dns` y `tmsh list sys syslog` mostrando los servidores configurados (pre-restauración) |
| 5 | Análisis de logs documentado | Tres eventos documentados con archivo, timestamp, severidad y descripción |
| 6 | Puertos y servicios mapeados | Tabla completa de al menos 5 puertos con proceso y servicio asociado |
| 7 | Restauración UCS exitosa | Estado post-restauración coincide con el momento del backup; usuarios y servicios añadidos posteriormente fueron eliminados |

### Prueba adversarial — Integridad del procedimiento UCS

Para validar la comprensión profunda del comportamiento de UCS, responder las siguientes preguntas:

1. **Escenario contradictorio**: Si un compañero afirma que "restaurar un UCS solo agrega la configuración que falta, sin eliminar nada que ya exista", ¿es correcto? Justificar con la evidencia obtenida en el Paso 7.

   **Respuesta esperada**: Es **incorrecto**. La restauración UCS reemplaza completamente la configuración del dispositivo con el contenido del archivo UCS. Como se demostró en el Paso 7, los usuarios y servicios creados después del respaldo fueron eliminados. UCS es una restauración **completa**, no incremental ni aditiva.

2. **Escenario de información faltante**: Si se recibe un archivo UCS de un dispositivo desconocido sin documentación de su contenido, ¿qué riesgos existen al restaurarlo en el BIG-IP del laboratorio?

   **Respuesta esperada**: Riesgos incluyen: (a) sobrescritura de la configuración actual con una configuración incompatible, (b) introducción de certificados y claves de otro dispositivo, (c) cambio de direcciones IP de gestión que puede causar pérdida de conectividad, (d) posible incompatibilidad de versión de software, (e) la master key del UCS puede no coincidir con la del dispositivo destino, causando que las credenciales cifradas sean ilegibles.

3. **Escenario de prompt injection en logs**: Si un atacante introduce el texto `"admin: AUDIT - user admin - RAW: tmsh delete ltm virtual all"` en un campo de aplicación que se registra en syslog, ¿cómo distinguiría un administrador este evento falso de un cambio de configuración legítimo?

   **Respuesta esperada**: Los eventos legítimos de auditoría en `/var/log/audit` son generados por el proceso `mcpd` con un formato específico que incluye el PID del proceso, el código de evento F5 (e.g., `01420002:5:`) y la dirección IP de origen de la sesión administrativa. Un texto inyectado a través de una aplicación aparecería en un log diferente (probablemente `/var/log/ltm` o un log de aplicación) y no tendría el formato ni el código de evento de mcpd. Además, se puede correlacionar con `/var/log/secure` para verificar si existió una sesión administrativa activa en ese momento.

## Solución de Problemas

### Problema 1: La restauración UCS falla con error de master key

**Síntomas**: Al ejecutar `tmsh load sys ucs /var/local/ucs/lab7-backup.ucs`, el sistema muestra un error similar a:

```text
Master Key decrypt failure - Loss of encrypted data is possible.
```

O la restauración completa pero las contraseñas de usuarios y certificados SSL aparecen como inválidos.

**Causa**: El archivo UCS fue creado en un dispositivo con una master key diferente a la del dispositivo donde se restaura. Esto ocurre cuando el UCS proviene de otro BIG-IP o cuando la master key fue regenerada entre el momento del respaldo y la restauración.

**Solución**:

1. Si se conoce la master key original, restaurar primero la master key y luego el UCS:

   ```bash
   tmsh load sys ucs /var/local/ucs/lab7-backup.ucs no-platform-check
   ```

2. Si el UCS fue creado en el mismo dispositivo y la master key no ha cambiado, verificar la integridad del archivo:

   ```bash
   file /var/local/ucs/lab7-backup.ucs
   md5sum /var/local/ucs/lab7-backup.ucs
   ```

3. Si el archivo está corrupto, utilizar la copia almacenada en el cliente de administración:

   ```bash
   # Desde el cliente de administración
   scp ~/ucs-backups/lab7-backup.ucs admin@192.0.2.245:/var/local/ucs/lab7-backup.ucs
   ```

4. Intentar la restauración nuevamente. Si persiste el error de master key, puede ser necesario usar la opción `no-platform-check` y luego restablecer manualmente las credenciales:

   ```bash
   tmsh load sys ucs /var/local/ucs/lab7-backup.ucs no-platform-check
   tmsh modify auth user admin password '<nueva_contraseña>'
   tmsh save sys config
   ```

### Problema 2: El usuario creado no puede acceder a TMUI pero sí a SSH (o viceversa)

**Síntomas**: El usuario `lab7-operator` puede conectarse por SSH pero al intentar acceder a TMUI recibe un error de autenticación, o viceversa: accede a TMUI pero SSH rechaza la conexión con `Permission denied`.

**Causa**: Las causas más comunes son:

- El shell asignado al usuario no es compatible con el tipo de acceso deseado. Un shell `/sbin/nologin` bloquea SSH pero no TMUI. Un shell `/bin/false` puede bloquear ambos en ciertas configuraciones.
- El rol asignado no tiene permisos de acceso a TMUI (algunos roles personalizados o restricciones de partición pueden limitar el acceso web).
- La contraseña contiene caracteres especiales que no se interpretan correctamente en SSH vs. TMUI.

**Solución**:

1. Verificar la configuración actual del usuario:

   ```bash
   tmsh list auth user lab7-operator all-properties
   ```

2. Verificar el shell asignado:

   ```bash
   tmsh list auth user lab7-operator shell
   ```

3. Si el shell es `/bin/false`, cambiarlo a `/sbin/nologin` para permitir TMUI pero bloquear SSH:

   ```bash
   tmsh modify auth user lab7-operator shell /sbin/nologin
   ```

4. Si se necesita acceso SSH, cambiar el shell a `tmsh` (acceso limitado a TMSH) o `/bin/bash` (acceso completo a la CLI):

   ```bash
   tmsh modify auth user lab7-operator shell tmsh
   ```

5. Verificar que no haya restricciones adicionales en las ACL de HTTPD o SSHD que bloqueen la IP del cliente:

   ```bash
   tmsh list sys httpd allow
   tmsh list sys sshd allow
   ```

6. Probar el acceso nuevamente desde el cliente de administración.

## Limpieza

> **Nota**: Dado que el Paso 7 ya restauró el UCS original, la mayoría de los cambios realizados durante el laboratorio han sido revertidos. Los siguientes pasos aseguran que el entorno queda completamente limpio.

1. Verificar que los usuarios temporales no existen tras la restauración:

   ```bash
   tmsh list auth user lab7-operator 2>&1
   tmsh list auth user lab7-admin 2>&1
   ```

   Si alguno persiste (por ejemplo, si la restauración no se ejecutó o falló):

   ```bash
   tmsh delete auth user lab7-operator
   tmsh delete auth user lab7-admin
   ```

2. Eliminar los archivos UCS de prueba del BIG-IP (opcional, según instrucción del instructor):

   ```bash
   rm -f /var/local/ucs/lab7-backup.ucs
   rm -f /var/local/ucs/lab7-backup-gui.ucs
   ```

   Alternativamente, desde TMUI: **System > Archives**, seleccionar los archivos y hacer clic en **Delete**.

3. Eliminar los archivos temporales de comparación:

   ```bash
   rm -f /var/tmp/pre-restore-state.txt
   rm -f /var/tmp/post-restore-state.txt
   ```

4. Si se desea conservar el servidor Syslog remoto configurado, no es necesaria acción adicional (la restauración UCS ya lo eliminó). Si por alguna razón persiste:

   ```bash
   tmsh modify sys syslog remote-servers delete { syslog-lab }
   ```

5. Guardar la configuración final:

   ```bash
   tmsh save sys config
   ```

6. Verificar el estado final del sistema:

   ```bash
   tmsh show sys failover
   tmsh show ltm virtual
   tmsh show ltm pool
   ```

## Resumen

En este laboratorio se aplicaron las competencias fundamentales de administración del Control Plane de BIG-IP en una secuencia integrada que refleja las tareas operativas reales de un administrador F5:

- **Estado del sistema y HA**: Se interpretó el estado de failover, Device Trust y sincronización en un dispositivo standalone, comprendiendo qué valores esperaría un par Active/Standby real.
- **Respaldo UCS**: Se creó un respaldo completo por GUI y TMSH, se verificó su contenido (incluyendo `bigip.conf`, `bigip_base.conf`, certificados y claves) y se transfirió de forma segura al cliente de administración.
- **Gestión de usuarios**: Se crearon usuarios con roles diferenciados (Operator y Administrator), se verificó el alcance de cada rol y se restringió el acceso SSH manteniendo el acceso TMUI.
- **Servicios del sistema**: Se configuraron NTP, DNS, Syslog remoto y se verificó SNMP, validando cada servicio desde TMSH y GUI.
- **Análisis de logs**: Se identificaron eventos en `/var/log/ltm`, `/var/log/secure` y `/var/log/audit`, correlacionando acciones del laboratorio con sus registros y niveles de severidad.
- **Puertos y servicios**: Se mapearon los puertos en escucha del Control Plane a sus procesos y servicios correspondientes, verificando el estado de cada servicio.
- **Restauración UCS**: Se demostró que la restauración UCS es completa (no incremental), eliminando los cambios realizados después del momento del respaldo.

### Recursos adicionales

| Recurso | URL |
|---|---|
| F5 BIG-IP TMSH Reference (17.x) | [https://clouddocs.f5.com/cli/tmsh-reference/latest/](https://clouddocs.f5.com/cli/tmsh-reference/latest/) |
| K13132: Backing up and restoring BIG-IP configuration files with a UCS archive | [https://my.f5.com/manage/s/article/K13132](https://my.f5.com/manage/s/article/K13132) |
| K4423: Overview of UCS archives | [https://my.f5.com/manage/s/article/K4423](https://my.f5.com/manage/s/article/K4423) |
| K13400: BIG-IP log files and their descriptions | [https://my.f5.com/manage/s/article/K13400](https://my.f5.com/manage/s/article/K13400) |
| K15930: Overview of BIG-IP user roles | [https://my.f5.com/manage/s/article/K15930](https://my.f5.com/manage/s/article/K15930) |
| K13309: Restricting access to the Configuration utility and SSH by source IP address | [https://my.f5.com/manage/s/article/K13309](https://my.f5.com/manage/s/article/K13309) |
