# Laboratorio 4 — Publicación de una aplicación mediante BIG-IP: Pools, Pool Members, Health Monitors, load balancing y Virtual Servers

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio crearás y validarás todos los objetos LTM necesarios para publicar una aplicación web a través de F5 BIG-IP: Nodes, Pool Members, un Pool con método Round Robin, un Health Monitor HTTP personalizado y un Virtual Server de tipo Standard. Partiendo de un BIG-IP con conectividad de red pre-configurada pero sin objetos LTM, construirás la cadena completa de entrega de aplicación y verificarás el funcionamiento extremo a extremo, incluyendo la detección automática de fallos en backends y la recuperación del servicio.

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear Nodes y Pool Members en BIG-IP, comprendiendo la relación jerárquica Node → Pool Member → Pool y el papel del service port.
- [ ] Configurar un Health Monitor HTTP personalizado y asociarlo a un Pool para determinar automáticamente la disponibilidad de cada backend.
- [ ] Crear un Virtual Server de tipo Standard con perfiles TCP y HTTP que publique una aplicación web accesible desde la red externa mediante un Pool con balanceo Round Robin.
- [ ] Validar la distribución de tráfico entre backends y verificar que BIG-IP detecta la caída de un Pool Member, redirige el tráfico al miembro sano y restaura automáticamente el miembro cuando el servicio se recupera.

## Prerrequisitos

### Conocimientos requeridos

| Área | Detalle |
|---|---|
| TCP/IP | Direccionamiento IP, subredes, máscaras, gateways, puertos TCP |
| HTTP | Modelo cliente-servidor, funcionamiento básico del puerto 80 |
| Linux | Comandos básicos de terminal: `systemctl`, `curl`, `ssh` |
| BIG-IP | Navegación básica en TMUI (GUI web) y concepto de TMSH |
| VLANs | Comprensión de segmentación de red |

### Acceso y entorno requeridos

| Recurso | Requisito |
|---|---|
| BIG-IP VE | Acceso TMUI vía `https://192.0.2.245` desde la estación de administración. Credenciales: `admin` / contraseña proporcionada por el instructor. |
| Cliente de pruebas | Máquina con IP `10.10.10.10/24` en la VLAN external, con `curl` (8.5.0) y Firefox ESR (128.0 ESR) disponibles. |
| Servidores backend | `web01` (`10.20.20.11`) y `web02` (`10.20.20.12`) con Apache 2.4.58 activo en TCP/80 y páginas `index.html` personalizadas (cada una identifica el servidor). |
| Acceso SSH | Credenciales SSH a `web01` y `web02` para detener/iniciar Apache durante las pruebas de failover. |
| Conectividad de red | VLANs `external` (interfaz 1.1, red `10.10.10.0/24`) e `internal` (interfaz 1.2, red `10.20.20.0/24`) configuradas en BIG-IP con Self IPs `10.10.10.245` y `10.20.20.245` respectivamente. |

> **Importante:** Este laboratorio asume que las VLANs, Self IPs y rutas del BIG-IP fueron pre-configuradas por el instructor. No se requiere ningún laboratorio anterior completado por el estudiante.

## Entorno de Laboratorio

### Topología de red

```
┌──────────────┐       VLAN external        ┌──────────────────┐       VLAN internal        ┌──────────────┐
│   Cliente     │      10.10.10.0/24         │    BIG-IP VE     │      10.20.20.0/24         │   web01       │
│ 10.10.10.10   ├───────────────────────────►│ ext: 10.10.10.245│───────────────────────────►│ 10.20.20.11   │
│               │                            │ int: 10.20.20.245│                            │ Apache :80    │
└──────────────┘                             │ mgmt:192.0.2.245 │                            └──────────────┘
                                             │ VIP: 10.10.10.100│       VLAN internal        ┌──────────────┐
┌──────────────┐       Management            │                  │───────────────────────────►│   web02       │
│ Admin Client  │      192.0.2.0/24          │                  │                            │ 10.20.20.12   │
│ 192.0.2.10    ├───────────────────────────►│                  │                            │ Apache :80    │
└──────────────┘                             └──────────────────┘                            └──────────────┘
```

### Software del entorno

| Componente | Versión exacta | Fuente oficial |
|---|---|---|
| F5 BIG-IP VE | 17.1.2 | https://my.f5.com/manage/s/downloads |
| Ubuntu Server (backends) | 22.04.4 LTS | https://releases.ubuntu.com/22.04/ |
| Apache HTTP Server | 2.4.58 | https://httpd.apache.org/download.cgi |
| curl (cliente) | 8.5.0 | https://curl.se/download.html |
| Mozilla Firefox ESR | 128.0 ESR | https://www.mozilla.org/en-US/firefox/enterprise/ |

### Verificación previa del entorno

Antes de comenzar, ejecuta las siguientes comprobaciones desde el **cliente de pruebas** (`10.10.10.10`):

**1. Conectividad al Self IP externo del BIG-IP:**

```bash
ping -c 3 10.10.10.245
```

Resultado esperado: 3 respuestas exitosas, 0% de pérdida de paquetes.

**2. Conectividad directa a los backends desde BIG-IP (ejecutar vía SSH al BIG-IP o desde TMSH):**

Accede por SSH al BIG-IP desde la estación de administración:

```bash
ssh admin@192.0.2.245
```

Desde el shell del BIG-IP:

```bash
ping -c 3 10.20.20.11
ping -c 3 10.20.20.12
```

Resultado esperado: ambos servidores responden.

**3. Servicio Apache activo en los backends (desde BIG-IP):**

```bash
curl -s -o /dev/null -w "%{http_code}" http://10.20.20.11/
curl -s -o /dev/null -w "%{http_code}" http://10.20.20.12/
```

Resultado esperado: ambos devuelven `200`.

**4. Verificar que no existen objetos LTM previos (desde TMSH en el BIG-IP):**

```bash
tmsh list ltm node
tmsh list ltm pool
tmsh list ltm virtual
```

Resultado esperado: los tres comandos no devuelven objetos o devuelven salida vacía. Si existen objetos previos, notifica al instructor antes de continuar.

## Instrucciones Paso a Paso

### Paso 1: Crear los Nodes para los servidores backend

**Objetivo:** Registrar cada servidor backend como un Node en BIG-IP, estableciendo la relación fundamental entre una dirección IP y un nombre descriptivo reutilizable.

**Instrucciones:**

1. Abre el navegador Firefox en la estación de administración y accede a `https://192.0.2.245`.
2. Inicia sesión con las credenciales `admin` y la contraseña del laboratorio.
3. Navega a **Local Traffic → Nodes → Node List**.
4. Haz clic en **Create** (esquina superior derecha).
5. Configura el primer Node con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | Name | `web01` |
   | Address | `10.20.20.11` |
   | Health Monitors | `default` (dejar sin cambios por ahora) |

6. Haz clic en **Finished**.
7. Repite los pasos 4–6 para el segundo Node:

   | Campo | Valor |
   |---|---|
   | Name | `web02` |
   | Address | `10.20.20.12` |
   | Health Monitors | `default` (dejar sin cambios por ahora) |

8. Haz clic en **Finished**.

**Resultado esperado:**

La lista de Nodes muestra dos entradas: `web01` (10.20.20.11) y `web02` (10.20.20.12). El estado de cada Node aparece como un círculo azul (Unknown) o verde (Available) dependiendo de si existe un monitor por defecto asociado al Node.

**Verificación:**

Desde TMSH en el BIG-IP, confirma la creación:

```bash
tmsh list ltm node web01
tmsh list ltm node web02
```

Salida esperada para cada Node (ejemplo para `web01`):

```
ltm node web01 {
    address 10.20.20.11
}
```

Verifica el estado operativo:

```bash
tmsh show ltm node web01
tmsh show ltm node web02
```

Confirma que el campo `Addr` muestra la IP correcta para cada Node. El estado `Availability` puede mostrar `unknown` si no hay monitor asignado directamente al Node; esto es correcto en este punto.

### Paso 2: Crear el Health Monitor HTTP personalizado

**Objetivo:** Definir un monitor de salud HTTP que valide no solo la conectividad TCP sino la respuesta HTTP real de la aplicación, asegurando que BIG-IP detecte fallos a nivel de aplicación.

**Instrucciones:**

1. En TMUI, navega a **Local Traffic → Monitors**.
2. Haz clic en **Create**.
3. Configura el monitor con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | Name | `monitor_http_app` |
   | Type | `HTTP` |
   | Interval | `5` |
   | Timeout | `16` |
   | Send String | `GET / HTTP/1.1\r\nHost: app.lab.local\r\nConnection: close\r\n\r\n` |
   | Receive String | (dejar vacío — cualquier respuesta HTTP válida se acepta) |

   > **Nota técnica:** El campo `Send String` envía una petición HTTP/1.1 completa con cabecera `Host` y `Connection: close`. El formato `\r\n` representa los retornos de carro y saltos de línea requeridos por el protocolo HTTP. La secuencia final `\r\n\r\n` marca el fin de las cabeceras HTTP.

   > **Nota sobre Interval y Timeout:** El Interval de 5 segundos indica que BIG-IP envía una comprobación cada 5 segundos. El Timeout de 16 segundos permite que fallen hasta 3 comprobaciones consecutivas (3 × 5 = 15 < 16) antes de marcar el miembro como Offline. Esta relación `Timeout > (3 × Interval)` es una práctica recomendada para evitar falsos positivos.

4. Deja el resto de campos en sus valores predeterminados.
5. Haz clic en **Finished**.

**Resultado esperado:**

El monitor `monitor_http_app` aparece en la lista de monitores bajo **Local Traffic → Monitors**, con tipo HTTP.

**Verificación:**

Desde TMSH:

```bash
tmsh list ltm monitor http monitor_http_app
```

Salida esperada (campos relevantes):

```
ltm monitor http monitor_http_app {
    adaptive disabled
    defaults-from http
    interval 5
    recv none
    send "GET / HTTP/1.1\r\nHost: app.lab.local\r\nConnection: close\r\n\r\n"
    time-until-up 0
    timeout 16
}
```

Confirma que `interval`, `timeout` y `send` coinciden con los valores configurados.

### Paso 3: Crear el Pool con Pool Members y asociar el Health Monitor

**Objetivo:** Agrupar ambos servidores backend como Pool Members dentro de un Pool, asignar el método de balanceo Round Robin y asociar el Health Monitor HTTP para validación automática de disponibilidad.

**Instrucciones:**

1. En TMUI, navega a **Local Traffic → Pools → Pool List**.
2. Haz clic en **Create**.
3. Configura el Pool con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | Name | `pool_http_app` |
   | Health Monitors | Selecciona `monitor_http_app` y muévelo a la columna **Active** |
   | Load Balancing Method | `Round Robin` |

4. En la sección **New Members**, agrega los Pool Members:

   - **Primer miembro:**
     - Selecciona **Node List** en el campo de selección.
     - Node: `web01`
     - Service Port: `80` (o escribe `HTTP`)
     - Haz clic en **Add**.

   - **Segundo miembro:**
     - Node: `web02`
     - Service Port: `80`
     - Haz clic en **Add**.

5. Verifica que ambos miembros aparecen en la tabla de miembros con las direcciones `10.20.20.11:80` y `10.20.20.12:80`.
6. Haz clic en **Finished**.

**Resultado esperado:**

El Pool `pool_http_app` aparece en la lista de Pools. Tras unos segundos (dependiendo del intervalo del monitor), el icono de estado del Pool debería cambiar a verde (Available), indicando que al menos un miembro está respondiendo correctamente al Health Monitor.

**Verificación:**

Desde TMSH:

```bash
tmsh list ltm pool pool_http_app
```

Salida esperada:

```
ltm pool pool_http_app {
    load-balancing-mode round-robin
    members {
        web01:http {
            address 10.20.20.11
        }
        web02:http {
            address 10.20.20.12
        }
    }
    monitor monitor_http_app
}
```

Confirma que el `load-balancing-mode` es `round-robin`, que ambos miembros están presentes con puerto `http` (80) y que el monitor es `monitor_http_app`.

### Paso 4: Verificar el estado de los Pool Members

**Objetivo:** Confirmar que el Health Monitor HTTP está validando correctamente ambos backends y que BIG-IP los marca como disponibles (Available).

**Instrucciones:**

1. En TMUI, navega a **Local Traffic → Pools → Pool List**.
2. Haz clic en el nombre del Pool `pool_http_app`.
3. Selecciona la pestaña **Members**.
4. Observa la columna **Status** de cada miembro. Ambos deben mostrar un círculo verde con la leyenda **Available (Enabled)**.

   > **Si algún miembro aparece en rojo (Offline) o azul (Unknown):** Espera al menos 20 segundos (4 ciclos del monitor con intervalo de 5 s) y recarga la página. Si persiste, consulta la sección **Solución de Problemas** al final de este documento.

5. Navega a **Statistics → Module Statistics → Local Traffic** y selecciona **Pools** en el menú desplegable de tipo de estadística.
6. Haz clic en el nombre del Pool `pool_http_app` para ver las estadísticas por miembro.
7. Verifica que la columna **Status** muestra verde para ambos miembros.

**Resultado esperado:**

Ambos Pool Members (`web01:80` y `web02:80`) muestran estado **Available (Enabled)** — círculo verde. Las columnas de estadísticas de tráfico muestran `0` en este momento porque aún no se ha enviado tráfico de aplicación.

**Verificación:**

Desde TMSH:

```bash
tmsh show ltm pool pool_http_app members
```

Salida esperada (extracto relevante para cada miembro):

```
Ltm::Pool: pool_http_app
  Availability     : available
  State            : enabled
  ...
  Pool Member:
    web01:http
      Addr            : 10.20.20.11
      Port            : 80
      Availability    : available
      State           : enabled
      Monitor Status  : up
    web02:http
      Addr            : 10.20.20.12
      Port            : 80
      Availability    : available
      State           : enabled
      Monitor Status  : up
```

Confirma que `Availability` es `available` y `Monitor Status` es `up` para ambos miembros.

### Paso 5: Crear el Virtual Server

**Objetivo:** Crear un Virtual Server de tipo Standard que exponga la aplicación web en la VIP `10.10.10.100:80` en la red externa, asociando el Pool, el perfil TCP y el perfil HTTP.

**Instrucciones:**

1. En TMUI, navega a **Local Traffic → Virtual Servers → Virtual Server List**.
2. Haz clic en **Create**.
3. Configura el Virtual Server con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | Name | `vs_http_app` |
   | Type | `Standard` |
   | Destination Address/Mask | `10.10.10.100` |
   | Service Port | `80` (o `HTTP`) |
   | Protocol | `TCP` |

4. En la sección **Configuration** (selecciona **Advanced** en el menú desplegable si los campos no son visibles):

   | Campo | Valor |
   |---|---|
   | HTTP Profile (Client) | `http` |
   | Source Address Translation | `Auto Map` |

   > **Nota sobre SNAT Auto Map:** Esta configuración es necesaria porque los servidores backend tienen como gateway predeterminado el Self IP interno del BIG-IP (`10.20.20.245`). Auto Map traduce la dirección de origen del cliente a la dirección del Self IP de la VLAN internal del BIG-IP, asegurando que el tráfico de retorno pase por el BIG-IP. Sin esta configuración, los servidores backend podrían intentar responder directamente al cliente, causando asimetría de tráfico.

5. En la sección **Resources**:

   | Campo | Valor |
   |---|---|
   | Default Pool | `pool_http_app` |

6. Verifica que el perfil TCP (`tcp`) esté incluido en la lista de perfiles. BIG-IP lo añade automáticamente al seleccionar el protocolo TCP; si no aparece, añádelo manualmente desde la sección de perfiles.
7. Haz clic en **Finished**.

**Resultado esperado:**

El Virtual Server `vs_http_app` aparece en la lista con estado verde (Available), indicando que tiene un Pool asociado con al menos un miembro disponible.

**Verificación:**

Desde TMSH:

```bash
tmsh list ltm virtual vs_http_app
```

Salida esperada (campos relevantes):

```
ltm virtual vs_http_app {
    destination 10.10.10.100:http
    ip-protocol tcp
    mask 255.255.255.255
    pool pool_http_app
    profiles {
        http { }
        tcp { }
    }
    source-address-translation {
        type automap
    }
}
```

Confirma: `destination` es `10.10.10.100:http`, `pool` es `pool_http_app`, los perfiles `http` y `tcp` están presentes, y `source-address-translation` es `automap`.

Verifica el estado operativo:

```bash
tmsh show ltm virtual vs_http_app
```

Confirma que `Availability` muestra `available` y `State` muestra `enabled`.

### Paso 6: Validar la distribución de tráfico Round Robin

**Objetivo:** Confirmar que el tráfico HTTP enviado a la VIP se distribuye de forma alternada entre ambos backends, verificando el funcionamiento del método Round Robin.

**Instrucciones:**

1. Desde el **cliente de pruebas** (`10.10.10.10`), abre una terminal.
2. Ejecuta el siguiente comando 6 veces consecutivas:

   ```bash
   curl http://10.10.10.100
   ```

3. Observa el contenido de cada respuesta. Las páginas `index.html` de cada servidor están personalizadas por el instructor, por lo que deberías ver respuestas alternadas que identifican a `web01` y `web02`.

4. Para una verificación más estructurada, ejecuta un bucle:

   ```bash
   for i in $(seq 1 6); do echo "--- Solicitud $i ---"; curl -s http://10.10.10.100; echo; done
   ```

**Resultado esperado:**

Las respuestas deben alternar entre los dos servidores. Con Round Robin, el patrón esperado es:

```
--- Solicitud 1 ---
Respuesta de web01 (o contenido identificativo de 10.20.20.11)
--- Solicitud 2 ---
Respuesta de web02 (o contenido identificativo de 10.20.20.12)
--- Solicitud 3 ---
Respuesta de web01
--- Solicitud 4 ---
Respuesta de web02
--- Solicitud 5 ---
Respuesta de web01
--- Solicitud 6 ---
Respuesta de web02
```

> **Nota:** La alternancia perfecta 1:1 depende de que no existan conexiones persistentes HTTP keep-alive que reutilicen la misma conexión TCP para múltiples solicitudes. El comando `curl` por defecto cierra la conexión después de cada solicitud, lo que favorece la distribución Round Robin observable. Si observas que todas las respuestas provienen del mismo servidor, consulta la sección de Solución de Problemas.

**Verificación:**

Verifica las estadísticas del Pool desde TMUI:

1. Navega a **Statistics → Module Statistics → Local Traffic**.
2. Selecciona **Pools** y haz clic en `pool_http_app`.
3. Confirma que ambos miembros muestran valores similares en las columnas **Total Requests** (la diferencia no debe ser mayor a 1 solicitud).

Desde TMSH:

```bash
tmsh show ltm pool pool_http_app members
```

Busca los contadores `Total requests` para cada miembro. Tras 6 solicitudes con Round Robin, cada miembro debería mostrar aproximadamente 3 solicitudes.

### Paso 7: Simular la caída de un Pool Member

**Objetivo:** Verificar que el Health Monitor detecta la indisponibilidad de un backend y que BIG-IP redirige automáticamente todo el tráfico al miembro sano restante.

**Instrucciones:**

1. Desde la estación de administración, abre una sesión SSH hacia `web01`:

   ```bash
   ssh usuario@10.20.20.11
   ```

   > Utiliza las credenciales proporcionadas por el instructor para los servidores backend.

2. Detén el servicio Apache en `web01`:

   ```bash
   sudo systemctl stop apache2
   ```

3. Confirma que Apache está detenido:

   ```bash
   sudo systemctl status apache2
   ```

   Resultado esperado: `Active: inactive (dead)`.

4. Espera al menos **20 segundos** para que el Health Monitor del BIG-IP detecte el fallo (Timeout de 16 segundos + margen).

5. En TMUI del BIG-IP, navega a **Local Traffic → Pools → pool_http_app → Members**.

6. Observa el estado de `web01:80`: debe mostrar un círculo rojo con estado **Offline**.

7. Observa el estado de `web02:80`: debe permanecer verde con estado **Available (Enabled)**.

8. Observa el estado general del Pool `pool_http_app`: debe permanecer **Available** (amarillo o verde) porque al menos un miembro está disponible.

9. Desde el **cliente de pruebas**, ejecuta nuevamente el bucle de solicitudes:

   ```bash
   for i in $(seq 1 6); do echo "--- Solicitud $i ---"; curl -s http://10.10.10.100; echo; done
   ```

**Resultado esperado:**

Todas las 6 respuestas provienen exclusivamente de `web02` (10.20.20.12). Ninguna solicitud falla con error de conexión; BIG-IP absorbe el fallo del backend de forma transparente para el cliente.

```
--- Solicitud 1 ---
Respuesta de web02
--- Solicitud 2 ---
Respuesta de web02
...
--- Solicitud 6 ---
Respuesta de web02
```

**Verificación:**

Desde TMSH:

```bash
tmsh show ltm pool pool_http_app members
```

Salida esperada (extracto):

```
  Pool Member:
    web01:http
      Availability    : offline
      State           : enabled
      Monitor Status  : down
    web02:http
      Availability    : available
      State           : enabled
      Monitor Status  : up
```

Confirma que `web01:http` tiene `Monitor Status: down` y `Availability: offline`, mientras `web02:http` permanece `up` y `available`.

Verifica también en los logs del BIG-IP:

```bash
tmsh show sys log ltm | grep pool_http_app | tail -5
```

Deberías observar una entrada indicando que el miembro `web01:80` fue marcado como `down` por el monitor.

### Paso 8: Restaurar el Pool Member y verificar la recuperación automática

**Objetivo:** Confirmar que al restaurar el servicio en el backend, el Health Monitor lo detecta automáticamente y BIG-IP vuelve a incluirlo en la distribución de tráfico.

**Instrucciones:**

1. En la sesión SSH a `web01` (`10.20.20.11`), reinicia Apache:

   ```bash
   sudo systemctl start apache2
   ```

2. Confirma que Apache está activo:

   ```bash
   sudo systemctl status apache2
   ```

   Resultado esperado: `Active: active (running)`.

3. Espera al menos **10 segundos** (2 ciclos del monitor con intervalo de 5 s) para que BIG-IP detecte la recuperación.

4. En TMUI, navega a **Local Traffic → Pools → pool_http_app → Members**.

5. Verifica que `web01:80` ha vuelto al estado verde **Available (Enabled)**.

6. Desde el **cliente de pruebas**, ejecuta nuevamente el bucle:

   ```bash
   for i in $(seq 1 6); do echo "--- Solicitud $i ---"; curl -s http://10.10.10.100; echo; done
   ```

**Resultado esperado:**

Las respuestas vuelven a alternar entre `web01` y `web02`, confirmando que Round Robin distribuye el tráfico entre ambos backends restaurados.

**Verificación:**

Desde TMSH:

```bash
tmsh show ltm pool pool_http_app members
```

Ambos miembros deben mostrar `Availability: available`, `State: enabled` y `Monitor Status: up`.

Verifica los logs para confirmar la recuperación:

```bash
tmsh show sys log ltm | grep pool_http_app | tail -5
```

Deberías observar una entrada indicando que el miembro `web01:80` fue marcado como `up`.

### Paso 9: Guardar la configuración

**Objetivo:** Persistir todos los cambios realizados en la configuración del BIG-IP para que sobrevivan a un reinicio, ya que los resultados de este laboratorio son el estado inicial requerido para los laboratorios subsiguientes.

**Instrucciones:**

1. Desde TMSH en el BIG-IP:

   ```bash
   tmsh save sys config
   ```

   Resultado esperado:

   ```
   Saving running configuration...
     /config/bigip.conf
     /config/bigip_base.conf
     /config/bigip_user.conf
   ```

2. Alternativamente, desde TMUI: navega a **System → Archives** y verifica que puedes crear un archivo UCS si el instructor lo solicita.

**Resultado esperado:**

La configuración queda persistida. Los objetos `web01`, `web02`, `monitor_http_app`, `pool_http_app` y `vs_http_app` estarán disponibles tras un reinicio del BIG-IP.

**Verificación:**

```bash
tmsh list ltm virtual vs_http_app
tmsh list ltm pool pool_http_app
tmsh list ltm monitor http monitor_http_app
tmsh list ltm node web01
tmsh list ltm node web02
```

Todos los comandos deben devolver la configuración creada durante el laboratorio.

## Validación y Pruebas

Completa la siguiente tabla de validación. Cada criterio debe verificarse con evidencia concreta (comando ejecutado y salida obtenida):

| # | Criterio de validación | Comando/Método | Resultado esperado | ¿Cumplido? |
|---|---|---|---|---|
| 1 | Nodes `web01` y `web02` existen con IPs correctas | `tmsh list ltm node` | Dos Nodes: `10.20.20.11` y `10.20.20.12` | ☐ |
| 2 | Monitor `monitor_http_app` configurado correctamente | `tmsh list ltm monitor http monitor_http_app` | Interval=5, Timeout=16, Send String con GET HTTP/1.1 | ☐ |
| 3 | Pool `pool_http_app` con 2 miembros, Round Robin y monitor | `tmsh list ltm pool pool_http_app` | `round-robin`, 2 miembros `:http`, monitor `monitor_http_app` | ☐ |
| 4 | Ambos Pool Members en estado Available | `tmsh show ltm pool pool_http_app members` | `Availability: available` y `Monitor Status: up` para ambos | ☐ |
| 5 | Virtual Server `vs_http_app` operativo | `tmsh show ltm virtual vs_http_app` | `Availability: available`, destination `10.10.10.100:http` | ☐ |
| 6 | Tráfico distribuido entre ambos backends | `for i in $(seq 1 6); do curl -s http://10.10.10.100; done` | Respuestas alternadas de web01 y web02 | ☐ |
| 7 | Failover: tráfico redirigido al miembro sano | Detener Apache en web01 + `curl` | Solo respuestas de web02, sin errores | ☐ |
| 8 | Recuperación: miembro restaurado automáticamente | Iniciar Apache en web01 + esperar + `curl` | Respuestas alternadas restauradas | ☐ |
| 9 | Configuración persistida | `tmsh save sys config` + `tmsh list ltm virtual` | Configuración presente tras guardar | ☐ |

### Prueba adversarial: solicitud a VIP inexistente

Para verificar que el comportamiento del BIG-IP es predecible y que no se producen respuestas inesperadas:

```bash
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 http://10.10.10.200
```

**Resultado esperado:** El comando debería fallar con timeout o conexión rechazada (código `000` o error de conexión), ya que `10.10.10.200` no tiene un Virtual Server configurado. Esto confirma que BIG-IP solo responde en las VIPs explícitamente definidas y no actúa como proxy abierto.

### Prueba adversarial: solicitud con método HTTP no estándar

```bash
curl -s -X DELETE http://10.10.10.100/
```

**Resultado esperado:** El servidor backend responde (probablemente con un error `405 Method Not Allowed` o una respuesta `200` según la configuración de Apache). Lo importante es verificar que BIG-IP reenvía la solicitud al Pool y no la bloquea, ya que no se han configurado políticas de seguridad de capa 7. Esto demuestra que el perfil HTTP por defecto no filtra métodos.

## Solución de Problemas

### Problema 1: Pool Members permanecen en estado Offline (rojo) tras crear el Pool

**Síntomas:**

- En TMUI, los Pool Members muestran un círculo rojo con estado **Offline**.
- `tmsh show ltm pool pool_http_app members` muestra `Monitor Status: down` para uno o ambos miembros.
- Las solicitudes `curl http://10.10.10.100` desde el cliente devuelven un error o una página de reset de conexión.

**Causa:**

El Health Monitor HTTP no puede alcanzar los servidores backend. Las causas más frecuentes son:

1. **Apache no está ejecutándose** en los servidores backend.
2. **Problema de conectividad de red** entre el Self IP interno del BIG-IP (`10.20.20.245`) y los servidores backend (`10.20.20.11`, `10.20.20.12`).
3. **El Send String del monitor contiene un error de formato** (por ejemplo, falta la secuencia final `\r\n\r\n`).

**Solución:**

1. Verifica que Apache está activo en ambos backends:

   ```bash
   ssh usuario@10.20.20.11 "sudo systemctl status apache2"
   ssh usuario@10.20.20.12 "sudo systemctl status apache2"
   ```

   Si Apache está detenido, inícialo: `sudo systemctl start apache2`.

2. Desde el BIG-IP, verifica conectividad HTTP directa a los backends:

   ```bash
   curl -v http://10.20.20.11/
   curl -v http://10.20.20.12/
   ```

   Si falla, verifica la configuración de Self IP y VLAN internal:

   ```bash
   tmsh list net self internal_self_ip
   tmsh list net vlan vlan_internal
   ```

3. Revisa la configuración del monitor:

   ```bash
   tmsh list ltm monitor http monitor_http_app
   ```

   Verifica que el campo `send` contiene exactamente: `"GET / HTTP/1.1\r\nHost: app.lab.local\r\nConnection: close\r\n\r\n"`. Si es incorrecto, modifícalo:

   ```bash
   tmsh modify ltm monitor http monitor_http_app send "GET / HTTP/1.1\r\nHost: app.lab.local\r\nConnection: close\r\n\r\n"
   ```

### Problema 2: Todas las respuestas de curl provienen del mismo servidor (sin alternancia Round Robin)

**Síntomas:**

- `curl http://10.10.10.100` ejecutado 6 veces devuelve siempre la respuesta del mismo backend (por ejemplo, siempre `web01`).
- Ambos Pool Members aparecen como Available (verde) en TMUI.
- Las estadísticas del Pool muestran que solo un miembro acumula solicitudes.

**Causa:**

Las causas más probables son:

1. **Source Address Translation (SNAT) no configurada como Auto Map.** Sin SNAT, el tráfico de retorno del backend puede no pasar por el BIG-IP, causando que el BIG-IP no vea las respuestas y reintente siempre con el mismo miembro.
2. **OneConnect o perfil de multiplexación activo** que reutiliza conexiones del lado servidor, haciendo que múltiples solicitudes del cliente se envíen por la misma conexión backend.
3. **Persistence profile accidentalmente asociado** al Virtual Server, forzando que un mismo cliente siempre se dirija al mismo backend.

**Solución:**

1. Verifica la configuración de SNAT en el Virtual Server:

   ```bash
   tmsh list ltm virtual vs_http_app source-address-translation
   ```

   Si no muestra `type automap`, corrige:

   ```bash
   tmsh modify ltm virtual vs_http_app source-address-translation { type automap }
   ```

2. Verifica que no hay perfil de persistencia asociado:

   ```bash
   tmsh list ltm virtual vs_http_app persist
   ```

   Si muestra algún perfil de persistencia, elimínalo:

   ```bash
   tmsh modify ltm virtual vs_http_app persist none
   ```

3. Verifica que no hay perfil OneConnect:

   ```bash
   tmsh list ltm virtual vs_http_app profiles
   ```

   Si aparece un perfil `oneconnect`, elimínalo:

   ```bash
   tmsh modify ltm virtual vs_http_app profiles delete { oneconnect }
   ```

4. Tras los cambios, reinicia las estadísticas y prueba nuevamente:

   ```bash
   tmsh reset-stats ltm pool pool_http_app
   ```

   Luego ejecuta el bucle de `curl` desde el cliente y verifica la alternancia.

## Limpieza

> **IMPORTANTE:** No ejecutes estos comandos de limpieza si vas a continuar con los laboratorios subsiguientes del módulo (Persistence Profiles, SSL). Los objetos creados en este laboratorio son el estado inicial requerido para esos laboratorios.

Si el instructor indica que se debe limpiar el entorno, ejecuta los siguientes comandos en orden desde TMSH en el BIG-IP:

```bash
## 1. Eliminar el Virtual Server (depende del Pool)
tmsh delete ltm virtual vs_http_app

## 2. Eliminar el Pool (depende de los Pool Members y del Monitor)
tmsh delete ltm pool pool_http_app

## 3. Eliminar el Health Monitor
tmsh delete ltm monitor http monitor_http_app

## 4. Eliminar los Nodes
tmsh delete ltm node web01
tmsh delete ltm node web02

## 5. Guardar la configuración limpia
tmsh save sys config
```

Verificación de limpieza:

```bash
tmsh list ltm virtual
tmsh list ltm pool
tmsh list ltm monitor http monitor_http_app
tmsh list ltm node
```

Todos los comandos deben devolver salida vacía o un error indicando que el objeto no existe.

Confirma que Apache sigue activo en ambos backends (no debe quedar detenido tras la prueba del Paso 7):

```bash
ssh usuario@10.20.20.11 "sudo systemctl status apache2 | grep Active"
ssh usuario@10.20.20.12 "sudo systemctl status apache2 | grep Active"
```

Resultado esperado: `Active: active (running)` en ambos servidores.

## Resumen

En este laboratorio has construido la cadena completa de publicación de una aplicación web a través de F5 BIG-IP LTM:

1. **Nodes** — Registraste los servidores backend como objetos reutilizables identificados por IP y nombre descriptivo.
2. **Health Monitor HTTP** — Definiste una comprobación activa que valida la respuesta HTTP real de cada backend, no solo la conectividad TCP.
3. **Pool con Pool Members** — Agrupaste los servicios HTTP de ambos backends, asignaste Round Robin como método de balanceo y asociaste el monitor para evaluación automática de disponibilidad.
4. **Virtual Server** — Expusiste la aplicación en una VIP accesible desde la red externa, con perfiles TCP y HTTP y traducción de direcciones (Auto Map).
5. **Validación extremo a extremo** — Confirmaste la distribución de tráfico, la detección automática de fallos y la recuperación sin intervención manual.

### Conceptos clave reforzados

- La relación jerárquica **Node → Pool Member → Pool → Virtual Server** define cómo BIG-IP organiza los destinos de tráfico.
- El **service port** en el Pool Member determina el servicio específico al que BIG-IP envía tráfico; puede diferir del puerto expuesto al cliente.
- La **disponibilidad se propaga por dependencia**: si un backend falla, su Pool Member se marca Offline; si todos los miembros fallan, el Pool se marca como no disponible.
- **Round Robin** distribuye solicitudes de forma secuencial entre los miembros disponibles, proporcionando una distribución equitativa cuando los backends tienen capacidad similar.
- **Auto Map (SNAT)** es necesario para garantizar que el tráfico de retorno pase por el BIG-IP cuando los backends usan el Self IP interno como gateway.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación F5: Pools y Pool Members LTM | https://clouddocs.f5.com/products/big-ip_ltm/manuals/product/ltm-basics-12-1-0/4.html |
| Referencia TMSH: ltm pool | https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_pool.html |
| Referencia TMSH: ltm node | https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html |
| Referencia TMSH: ltm monitor http | https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_monitor_http.html |
| Referencia TMSH: ltm virtual | https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_virtual.html |
| K13637: Health Monitor troubleshooting | https://my.f5.com/manage/s/article/K13637 |

---

# Laboratorio 5 — Control avanzado de sesiones y aplicaciones: Persistence, TCP/HTTP Profiles, SSL, Priority Groups, iRules e identificación de objetos iApp

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (Apply) |
| **Versión BIG-IP** | F5 BIG-IP VE 17.1.2 |
| **Alineación** | F5CAB3.01 – F5CAB3.02 |

## Descripción General

Este laboratorio amplía la configuración del Virtual Server HTTP construido en los Laboratorios 1–4 incorporando control avanzado de sesiones y procesamiento de aplicaciones. El estudiante configurará Persistence Profiles (Cookie Insert y Source Address Affinity), optimizará el transporte con TCP y HTTP Profiles, habilitará HTTPS con certificados autofirmados mediante Client SSL y Server SSL Profiles, implementará Priority Groups para failover controlado entre Pool Members, creará una iRule condicional basada en cabeceras HTTP e identificará las dependencias y restricciones de objetos generados por una iApp de demostración. Al finalizar, el Virtual Server dispondrá de cifrado extremo a extremo, persistencia de sesión, distribución de carga priorizada y lógica personalizada de enrutamiento de tráfico.

## Objetivos de Aprendizaje

- [ ] Configurar y verificar Cookie Persistence y Source Address Affinity Persistence en un Virtual Server, demostrando la afinidad de sesión mediante inspección de cookies con `curl`.
- [ ] Asignar TCP Profiles optimizados (client-side/server-side) y un HTTP Profile personalizado con X-Forwarded-For, validando la inserción de cabeceras en el backend.
- [ ] Generar un certificado autofirmado con OpenSSL, crear Client SSL y Server SSL Profiles, y habilitar cifrado extremo a extremo entre cliente, BIG-IP y servidores backend.
- [ ] Implementar Priority Groups en Pool Members con Priority Group Activation y verificar la promoción automática de miembros de menor prioridad ante la caída del grupo principal.
- [ ] Crear una iRule que evalúe cabeceras HTTP para enrutamiento condicional, e identificar las dependencias y restricciones de modificación de objetos creados por una iApp.

## Prerrequisitos

### Conocimientos requeridos

- Comprensión de la relación jerárquica Node → Pool Member → Pool → Virtual Server según la Lección 3.1.
- Conceptos fundamentales de TCP/IP, HTTP/1.1, HTTPS, handshake TLS y certificados X.509.
- Familiaridad con cookies de sesión HTTP y su función en la persistencia de aplicaciones.
- Navegación básica por TMUI y uso de TMSH adquiridos en los Laboratorios 1–4.

### Acceso y estado requerido

| Recurso | Detalle |
|---|---|
| Virtual Server `vs_http_lab` | IP `10.10.10.100:80`, estado Available, creado en laboratorios anteriores |
| Pool `pool_webapp` | Miembros `10.20.20.11:80` y `10.20.20.12:80`, ambos Available |
| Health Monitor `monitor_http_lab` | Asociado a `pool_webapp`, verificando respuesta HTTP |
| BIG-IP Management | SSH y TMUI en `192.0.2.245` desde `192.0.2.10` — credenciales `admin` / `F5lab2024!` |
| Cliente de pruebas | IP `10.10.10.10/24`, con `curl` 8.5.0 y Firefox ESR 128.0 |
| Servidores backend | `web01` (`10.20.20.11`) y `web02` (`10.20.20.12`) con Apache 2.4.58 en puerto 80 |
| OpenSSL | Versión 3.0.2 disponible en el cliente o en los servidores backend (Ubuntu 22.04.4 LTS) |

> **Nota sobre versiones:** La especificación del laboratorio menciona OpenSSL 3.3.0 y Apache 2.4.59; sin embargo, el entorno real del curso utiliza **OpenSSL 3.0.2** (incluido en Ubuntu 22.04.4 LTS) y **Apache 2.4.58**. Los procedimientos de este laboratorio están validados para estas versiones del entorno. Igualmente, la versión de BIG-IP del entorno es **17.1.2**, no 17.1.1.3.

## Entorno de Laboratorio

### Topología de red

```
┌──────────────┐     VLAN External      ┌──────────────┐     VLAN Internal     ┌──────────────┐
│   Cliente     │     10.10.10.0/24      │   BIG-IP VE  │     10.20.20.0/24     │   web01      │
│ 10.10.10.10   ├────────────────────────┤ ext: .245    ├───────────────────────┤ 10.20.20.11  │
│               │                        │ int: .245    │                       │  Apache:80   │
└──────────────┘                         │ mgmt: 192.0. │                       └──────────────┘
                                         │  2.245       │                       ┌──────────────┐
┌──────────────┐     VLAN Management     │              │                       │   web02      │
│ Admin Client  ├────────────────────────┤              ├───────────────────────┤ 10.20.20.12  │
│ 192.0.2.10    │     192.0.2.0/24       └──────────────┘                       │  Apache:80   │
└──────────────┘                                                                └──────────────┘
```

### Direcciones clave

| Objeto | Dirección |
|---|---|
| Virtual Server HTTP | `10.10.10.100:80` |
| Virtual Server HTTPS (a crear) | `10.10.10.101:443` |
| BIG-IP external Self IP | `10.10.10.245/24` |
| BIG-IP internal Self IP | `10.20.20.245/24` |
| web01 | `10.20.20.11:80` (y `:443` tras configuración SSL backend) |
| web02 | `10.20.20.12:80` (y `:443` tras configuración SSL backend) |

### Verificación previa del entorno

Desde el cliente de pruebas (`10.10.10.10`), confirmar que el Virtual Server HTTP responde:

```bash
curl -s -o /dev/null -w "%{http_code}" http://10.10.10.100
```

**Salida esperada:** `200`

Desde TMSH en BIG-IP, confirmar el estado del pool:

```bash
tmsh show ltm pool pool_webapp members
```

**Salida esperada:** Ambos miembros con estado `available` y `State: up`.

## Instrucciones Paso a Paso

### Paso 1: Configurar Cookie Persistence Profile y verificar afinidad de sesión

**Objetivo:** Crear un Cookie Persistence Profile de tipo HTTP Cookie Insert, asociarlo al Virtual Server HTTP y demostrar que las solicitudes subsiguientes del mismo cliente se dirigen al mismo Pool Member.

**Instrucciones:**

1. Acceder a TMUI en `https://192.0.2.245` con las credenciales `admin` / `F5lab2024!`.

2. Crear el Cookie Persistence Profile. Navegar a **Local Traffic → Profiles → Persistence** y hacer clic en **Create**.

   | Campo | Valor |
   |---|---|
   | Name | `persist_cookie_lab` |
   | Persistence Type | `Cookie` |
   | Cookie Method | `HTTP Cookie Insert` |
   | Timeout | `300` (segundos) |

   Hacer clic en **Finished**.

3. Alternativamente, crear el perfil mediante TMSH:

   ```bash
   tmsh create ltm persistence cookie persist_cookie_lab \
       method insert \
       timeout 300
   ```

4. Asociar el perfil al Virtual Server. Navegar a **Local Traffic → Virtual Servers → vs_http_lab → Resources**. En el campo **Default Persistence Profile**, seleccionar `persist_cookie_lab`. Hacer clic en **Update**.

   Equivalente TMSH:

   ```bash
   tmsh modify ltm virtual vs_http_lab persist replace-all-with { persist_cookie_lab }
   ```

5. Desde el cliente de pruebas, ejecutar una primera solicitud con `curl` mostrando las cabeceras de respuesta:

   ```bash
   curl -v http://10.10.10.100/ 2>&1 | grep -i "set-cookie\|server"
   ```

6. Observar la cookie `BIGipServerpool_webapp` en la cabecera `Set-Cookie`. Copiar el valor completo de la cookie.

7. Ejecutar solicitudes subsiguientes enviando la cookie recibida para verificar persistencia:

   ```bash
   COOKIE="BIGipServerpool_webapp=<valor_obtenido>"
   for i in $(seq 1 5); do
       curl -s -b "$COOKIE" http://10.10.10.100/ | grep -o "web0[12]"
   done
   ```

   Sustituir `<valor_obtenido>` por el valor real de la cookie del paso 6.

8. Verificar que las 5 solicitudes se dirigen al mismo servidor backend.

**Salida esperada:**

```
web01
web01
web01
web01
web01
```

(O `web02` en las 5 líneas, dependiendo del miembro seleccionado inicialmente.)

**Verificación:**

```bash
tmsh show ltm persistence persist-records all-properties
```

Confirmar que existe al menos un registro de persistencia asociado a `pool_webapp` con el tipo `cookie`.

---

### Paso 2: Configurar Source Address Affinity Persistence y comparar comportamiento

**Objetivo:** Crear un Source Address Affinity Persistence Profile, sustituirlo en el Virtual Server y comparar el mecanismo de afinidad con el Cookie Persistence.

**Instrucciones:**

1. Crear el Source Address Affinity Profile. En TMUI, navegar a **Local Traffic → Profiles → Persistence → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `persist_srcaddr_lab` |
   | Persistence Type | `Source Address Affinity` |
   | Timeout | `180` (segundos) |
   | Mask | `255.255.255.255` |

   Hacer clic en **Finished**.

   Equivalente TMSH:

   ```bash
   tmsh create ltm persistence source-addr persist_srcaddr_lab \
       timeout 180 \
       mask 255.255.255.255
   ```

2. Cambiar el perfil de persistencia del Virtual Server:

   ```bash
   tmsh modify ltm virtual vs_http_lab persist replace-all-with { persist_srcaddr_lab }
   ```

3. Limpiar los registros de persistencia anteriores:

   ```bash
   tmsh delete ltm persistence persist-records all
   ```

4. Desde el cliente, ejecutar 5 solicitudes sin enviar cookies:

   ```bash
   for i in $(seq 1 5); do
       curl -s http://10.10.10.100/ | grep -o "web0[12]"
   done
   ```

5. Verificar que todas las solicitudes se dirigen al mismo miembro (afinidad por IP de origen).

6. Inspeccionar las cabeceras de respuesta y confirmar que **no** se envía cookie `BIGipServer`:

   ```bash
   curl -v http://10.10.10.100/ 2>&1 | grep -i "set-cookie"
   ```

   **Salida esperada:** Sin líneas `Set-Cookie` de BIG-IP.

7. Verificar el registro de persistencia:

   ```bash
   tmsh show ltm persistence persist-records
   ```

   **Salida esperada:** Registro con la IP de origen `10.10.10.10` asociada a un Pool Member específico.

**Verificación:**

Documentar la diferencia observada: Cookie Persistence utiliza una cookie HTTP insertada por BIG-IP (visible en cabeceras), mientras que Source Address Affinity no modifica las cabeceras HTTP y persiste basándose en la dirección IP del cliente.

8. **Restaurar Cookie Persistence** para los pasos siguientes:

   ```bash
   tmsh modify ltm virtual vs_http_lab persist replace-all-with { persist_cookie_lab }
   ```

---

### Paso 3: Asignar TCP Profiles optimizados y HTTP Profile con X-Forwarded-For

**Objetivo:** Configurar perfiles TCP diferenciados para el lado cliente y servidor, y un HTTP Profile que inserte la cabecera `X-Forwarded-For` y elimine la cabecera `Server` de las respuestas.

**Instrucciones:**

1. Asignar los TCP Profiles al Virtual Server. Navegar a **Local Traffic → Virtual Servers → vs_http_lab → Properties** (pestaña **Advanced**).

   En la sección **Configuration: Advanced**:
   - **Protocol Profile (Client):** `tcp-wan-optimized`
   - **Protocol Profile (Server):** `tcp-lan-optimized`

   Hacer clic en **Update**.

   Equivalente TMSH:

   ```bash
   tmsh modify ltm virtual vs_http_lab \
       profiles replace-all-with { tcp-wan-optimized { context clientside } tcp-lan-optimized { context serverside } http }
   ```

2. Crear un HTTP Profile personalizado. Navegar a **Local Traffic → Profiles → Services → HTTP → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `http_lab_custom` |
   | Parent Profile | `http` |
   | Insert X-Forwarded-For | `Enabled` (marcar la casilla de override y activar) |
   | Server Agent Name | (dejar vacío y marcar **Erase** en Server Header) |

   > **Nota:** En BIG-IP 17.1.2, la opción para eliminar la cabecera Server se encuentra en el campo **Server Agent Name**. Establecerlo vacío con la opción de reescritura habilitada elimina la cabecera Server de las respuestas.

   Hacer clic en **Finished**.

   Equivalente TMSH:

   ```bash
   tmsh create ltm profile http http_lab_custom \
       defaults-from http \
       insert-xforwarded-for enabled \
       server-agent-name none
   ```

3. Asociar el HTTP Profile personalizado al Virtual Server:

   ```bash
   tmsh modify ltm virtual vs_http_lab \
       profiles replace-all-with { tcp-wan-optimized { context clientside } tcp-lan-optimized { context serverside } http_lab_custom }
   ```

4. Verificar la configuración aplicada:

   ```bash
   tmsh list ltm virtual vs_http_lab profiles
   ```

   **Salida esperada:**

   ```
   ltm virtual vs_http_lab {
       profiles {
           http_lab_custom {
               context all
           }
           tcp-lan-optimized {
               context serverside
           }
           tcp-wan-optimized {
               context clientside
           }
       }
   }
   ```

5. Verificar la inserción de `X-Forwarded-For` en el backend. Primero, habilitar temporalmente el log de cabeceras en uno de los servidores web. En `web01` (`10.20.20.11`), via SSH:

   ```bash
   # En web01, crear un script CGI de diagnóstico
   sudo bash -c 'cat > /var/www/html/headers.sh << "SCRIPT"
   #!/bin/bash
   echo "Content-Type: text/plain"
   echo ""
   env | grep HTTP_ | sort
   SCRIPT'
   sudo chmod +x /var/www/html/headers.sh
   sudo a2enmod cgid
   sudo systemctl restart apache2
   ```

   Alternativamente, si CGI no está disponible, inspeccionar el log de acceso de Apache:

   ```bash
   # En web01, modificar el formato de log para incluir X-Forwarded-For
   sudo bash -c 'echo "LogFormat \"%h %l %u %t \\\"%r\\\" %>s %b \\\"%{X-Forwarded-For}i\\\"\" xfflog" >> /etc/apache2/apache2.conf'
   sudo bash -c 'echo "CustomLog /var/log/apache2/xff_access.log xfflog" >> /etc/apache2/sites-enabled/000-default.conf'
   sudo systemctl reload apache2
   ```

6. Desde el cliente, enviar una solicitud:

   ```bash
   curl -s http://10.10.10.100/
   ```

7. En `web01`, verificar el log:

   ```bash
   tail -1 /var/log/apache2/xff_access.log
   ```

   **Salida esperada:** Una línea que contenga `"10.10.10.10"` como valor de X-Forwarded-For.

8. Verificar que la cabecera `Server` ha sido eliminada de la respuesta:

   ```bash
   curl -sI http://10.10.10.100/ | grep -i "^server:"
   ```

   **Salida esperada:** Sin salida (la cabecera Server no aparece o muestra un valor vacío).

**Verificación:**

```bash
tmsh show ltm virtual vs_http_lab
```

Confirmar que los perfiles `tcp-wan-optimized`, `tcp-lan-optimized` y `http_lab_custom` aparecen en la configuración del Virtual Server.

---

### Paso 4: Generar certificado autofirmado y configurar Client SSL Profile para HTTPS

**Objetivo:** Crear un certificado autofirmado con OpenSSL, importarlo al BIG-IP, configurar un Client SSL Profile y habilitar un Virtual Server HTTPS en `10.10.10.101:443`.

**Instrucciones:**

1. Desde el cliente de administración (`192.0.2.10`) o directamente en el BIG-IP vía SSH, generar la clave privada y el certificado autofirmado:

   ```bash
   openssl req -x509 -newkey rsa:2048 -keyout lab_f5curso.key -out lab_f5curso.crt \
       -sha256 -days 365 -nodes \
       -subj "/C=MX/ST=Lab/L=Lab/O=F5Curso/CN=lab.f5curso.local"
   ```

   > **Nota:** Se utiliza OpenSSL 3.0.2 (versión incluida en Ubuntu 22.04.4 LTS). Verificar la versión con `openssl version`.

2. Verificar el certificado generado:

   ```bash
   openssl x509 -in lab_f5curso.crt -noout -subject -dates
   ```

   **Salida esperada:**

   ```
   subject=C = MX, ST = Lab, L = Lab, O = F5Curso, CN = lab.f5curso.local
   notBefore=<fecha_actual>
   notAfter=<fecha_actual + 365 días>
   ```

3. Importar el certificado y la clave al BIG-IP mediante TMUI. Navegar a **System → Certificate Management → Traffic Certificate Management → SSL Certificate List → Import**.

   - **Import Type:** `Certificate`
   - **Certificate Name:** `lab_f5curso_cert`
   - **Certificate Source:** Pegar el contenido de `lab_f5curso.crt` o subir el archivo.
   - Hacer clic en **Import**.

   Repetir para la clave:
   - **Import Type:** `Key`
   - **Key Name:** `lab_f5curso_key`
   - **Key Source:** Pegar el contenido de `lab_f5curso.key` o subir el archivo.
   - Hacer clic en **Import**.

   Equivalente TMSH (si los archivos están en el BIG-IP en `/var/tmp/`):

   ```bash
   tmsh install sys crypto cert lab_f5curso_cert from-local-file /var/tmp/lab_f5curso.crt
   tmsh install sys crypto key lab_f5curso_key from-local-file /var/tmp/lab_f5curso.key
   ```

4. Crear el Client SSL Profile. Navegar a **Local Traffic → Profiles → SSL → Client → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `clientssl_lab` |
   | Parent Profile | `clientssl` |
   | Certificate Key Chain | Agregar: Certificate = `lab_f5curso_cert`, Key = `lab_f5curso_key` |

   Hacer clic en **Finished**.

   Equivalente TMSH:

   ```bash
   tmsh create ltm profile client-ssl clientssl_lab \
       cert-key-chain add { default { cert lab_f5curso_cert key lab_f5curso_key } }
   ```

5. Crear el Virtual Server HTTPS. Navegar a **Local Traffic → Virtual Servers → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `vs_https_lab` |
   | Type | `Standard` |
   | Destination Address/Mask | `10.10.10.101` |
   | Service Port | `443` |
   | Protocol | `TCP` |
   | Protocol Profile (Client) | `tcp-wan-optimized` |
   | Protocol Profile (Server) | `tcp-lan-optimized` |
   | HTTP Profile | `http_lab_custom` |
   | SSL Profile (Client) | `clientssl_lab` |
   | Source Address Translation | `Auto Map` |
   | Default Pool | `pool_webapp` |
   | Default Persistence Profile | `persist_cookie_lab` |

   Hacer clic en **Finished**.

   Equivalente TMSH:

   ```bash
   tmsh create ltm virtual vs_https_lab \
       destination 10.10.10.101:443 \
       ip-protocol tcp \
       profiles replace-all-with { tcp-wan-optimized { context clientside } tcp-lan-optimized { context serverside } http_lab_custom clientssl_lab } \
       source-address-translation { type automap } \
       pool pool_webapp \
       persist replace-all-with { persist_cookie_lab }
   ```

6. Verificar el estado del nuevo Virtual Server:

   ```bash
   tmsh show ltm virtual vs_https_lab
   ```

   **Salida esperada:** Estado `available` con la dirección de destino `10.10.10.101:443`.

7. Desde el cliente, probar el acceso HTTPS con `curl`:

   ```bash
   curl -k -v https://10.10.10.101/ 2>&1 | head -30
   ```

   **Salida esperada:** Handshake TLS exitoso con el certificado `CN=lab.f5curso.local`, seguido de la respuesta HTTP 200 del backend.

8. Desde Firefox ESR 128.0 en el cliente, acceder a `https://10.10.10.101/`. Aceptar la advertencia de certificado autofirmado. Verificar que la página del backend se muestra correctamente. Inspeccionar el certificado haciendo clic en el icono de candado para confirmar que el CN es `lab.f5curso.local`.

**Verificación:**

```bash
tmsh list ltm profile client-ssl clientssl_lab cert-key-chain
```

Confirmar que el perfil referencia `lab_f5curso_cert` y `lab_f5curso_key`.

---

### Paso 5: Configurar Server SSL Profile para cifrado extremo a extremo

**Objetivo:** Habilitar HTTPS en los servidores backend, crear un Server SSL Profile y asociarlo al Virtual Server HTTPS para cifrar el tráfico entre BIG-IP y los Pool Members.

**Instrucciones:**

1. Habilitar HTTPS en los servidores backend. En `web01` (`10.20.20.11`) vía SSH:

   ```bash
   sudo a2enmod ssl
   sudo a2ensite default-ssl
   sudo systemctl restart apache2
   ```

   Verificar que Apache escucha en el puerto 443:

   ```bash
   sudo ss -tlnp | grep :443
   ```

   **Salida esperada:** Línea mostrando `*:443` con `apache2`.

2. Repetir el paso 1 en `web02` (`10.20.20.12`).

3. Crear un pool alternativo con miembros en puerto 443 o modificar los miembros existentes. Para este laboratorio, crear un nuevo pool para el backend HTTPS:

   ```bash
   tmsh create ltm pool pool_webapp_ssl \
       members add { 10.20.20.11:443 10.20.20.12:443 } \
       monitor https
   ```

4. Crear el Server SSL Profile. Navegar a **Local Traffic → Profiles → SSL → Server → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `serverssl_lab` |
   | Parent Profile | `serverssl-insecure-compatible` |

   > **Nota:** Se utiliza `serverssl-insecure-compatible` porque los backends tienen certificados autofirmados de Apache por defecto. En producción se debe usar un perfil que valide el certificado del servidor.

   Hacer clic en **Finished**.

   Equivalente TMSH:

   ```bash
   tmsh create ltm profile server-ssl serverssl_lab \
       defaults-from serverssl-insecure-compatible
   ```

5. Asociar el Server SSL Profile y el nuevo pool al Virtual Server HTTPS:

   ```bash
   tmsh modify ltm virtual vs_https_lab \
       profiles add { serverssl_lab { context serverside } } \
       pool pool_webapp_ssl
   ```

6. Verificar la configuración completa de perfiles:

   ```bash
   tmsh list ltm virtual vs_https_lab profiles
   ```

   **Salida esperada:** Debe incluir `clientssl_lab` (clientside), `serverssl_lab` (serverside), `tcp-wan-optimized` (clientside), `tcp-lan-optimized` (serverside) y `http_lab_custom`.

7. Probar el cifrado extremo a extremo desde el cliente:

   ```bash
   curl -k https://10.10.10.101/
   ```

   **Salida esperada:** Contenido HTML del backend servido a través de HTTPS en ambos tramos.

8. Verificar en los logs de BIG-IP que el handshake SSL del lado servidor se completa:

   ```bash
   tmsh show ltm pool pool_webapp_ssl members
   ```

   **Salida esperada:** Ambos miembros en estado `available` con el monitor `https` activo.

**Verificación:**

Desde el cliente, confirmar la cadena completa de cifrado:

```bash
curl -k -v https://10.10.10.101/ 2>&1 | grep -E "SSL connection|subject|issuer"
```

El handshake del lado cliente debe mostrar el certificado `lab.f5curso.local`. El tráfico hacia el backend viaja cifrado (verificable porque el pool usa puerto 443 con monitor HTTPS exitoso).

---

### Paso 6: Configurar Priority Groups en Pool Members

**Objetivo:** Asignar prioridades diferenciadas a los Pool Members, configurar Priority Group Activation y verificar la promoción automática del grupo de menor prioridad cuando el grupo principal pierde disponibilidad.

**Instrucciones:**

1. Asignar Priority Group 10 a `web01` y Priority Group 5 a `web02` en el pool `pool_webapp_ssl`:

   ```bash
   tmsh modify ltm pool pool_webapp_ssl members modify { 10.20.20.11:443 { priority-group 10 } 10.20.20.12:443 { priority-group 5 } }
   ```

2. Configurar Priority Group Activation con valor `1` (activar el grupo de menor prioridad cuando queden menos de 1 miembro disponible en el grupo de alta prioridad):

   ```bash
   tmsh modify ltm pool pool_webapp_ssl min-active-members 1
   ```

3. Verificar la configuración:

   ```bash
   tmsh list ltm pool pool_webapp_ssl members
   ```

   **Salida esperada:**

   ```
   ltm pool pool_webapp_ssl {
       members {
           10.20.20.11:443 {
               priority-group 10
           }
           10.20.20.12:443 {
               priority-group 5
           }
       }
       min-active-members 1
   }
   ```

4. Enviar múltiples solicitudes y verificar que todas se dirigen a `web01` (prioridad 10):

   ```bash
   tmsh delete ltm persistence persist-records all
   for i in $(seq 1 6); do
       curl -k -s https://10.10.10.101/ | grep -o "web0[12]"
   done
   ```

   **Salida esperada:** Todas las respuestas deben indicar `web01`.

5. Simular la caída del miembro de alta prioridad. En `web01` vía SSH:

   ```bash
   sudo systemctl stop apache2
   ```

6. Esperar 15-30 segundos para que el Health Monitor detecte la caída, luego verificar:

   ```bash
   tmsh show ltm pool pool_webapp_ssl members
   ```

   **Salida esperada:** `10.20.20.11:443` en estado `offline` o `down`; `10.20.20.12:443` en estado `available`.

7. Enviar solicitudes y verificar que ahora se dirigen a `web02` (prioridad 5, promovido):

   ```bash
   for i in $(seq 1 4); do
       curl -k -s https://10.10.10.101/ | grep -o "web0[12]"
   done
   ```

   **Salida esperada:** Todas las respuestas indican `web02`.

8. Restaurar el servicio en `web01`:

   ```bash
   sudo systemctl start apache2
   ```

9. Esperar a que el monitor marque `web01` como disponible y verificar que el tráfico regresa al grupo de prioridad 10:

   ```bash
   sleep 20
   tmsh delete ltm persistence persist-records all
   for i in $(seq 1 4); do
       curl -k -s https://10.10.10.101/ | grep -o "web0[12]"
   done
   ```

   **Salida esperada:** Las respuestas regresan a `web01`.

**Verificación:**

```bash
tmsh show ltm pool pool_webapp_ssl
```

Confirmar que `Available Members` muestra 2, `min-active-members` es 1, y los Priority Groups están correctamente asignados.

---

### Paso 7: Crear una iRule condicional basada en cabeceras HTTP

**Objetivo:** Crear una iRule que inspeccione la cabecera HTTP `X-Lab-Route` y retorne una respuesta personalizada cuando su valor sea `secondary`, permitiendo enrutamiento condicional del tráfico.

**Instrucciones:**

1. Crear la iRule. Navegar a **Local Traffic → iRules → iRule List → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `lab_irule_redirect` |
   | Definition | (ver código abajo) |

   ```tcl
   when HTTP_REQUEST {
       if { [HTTP::header value "X-Lab-Route"] eq "secondary" } {
           HTTP::respond 200 content "iRule: Ruta secundaria activada para [IP::client_addr]" noserver
           return
       }
   }
   ```

   Hacer clic en **Finished**.

   Equivalente TMSH:

   ```bash
   tmsh create ltm rule lab_irule_redirect { \
       when HTTP_REQUEST { \
           if { [HTTP::header value "X-Lab-Route"] eq "secondary" } { \
               HTTP::respond 200 content "iRule: Ruta secundaria activada para [IP::client_addr]" noserver \
               return \
           } \
       } \
   }
   ```

2. Asociar la iRule al Virtual Server HTTPS:

   ```bash
   tmsh modify ltm virtual vs_https_lab rules { lab_irule_redirect }
   ```

3. Verificar la asociación:

   ```bash
   tmsh list ltm virtual vs_https_lab rules
   ```

   **Salida esperada:**

   ```
   ltm virtual vs_https_lab {
       rules {
           lab_irule_redirect
       }
   }
   ```

4. Probar la iRule enviando la cabecera `X-Lab-Route: secondary`:

   ```bash
   curl -k -H "X-Lab-Route: secondary" https://10.10.10.101/
   ```

   **Salida esperada:**

   ```
   iRule: Ruta secundaria activada para 10.10.10.10
   ```

5. Probar sin la cabecera (comportamiento normal, tráfico al pool):

   ```bash
   curl -k https://10.10.10.101/
   ```

   **Salida esperada:** Contenido HTML del backend (comportamiento normal del pool).

6. Probar con un valor diferente de la cabecera:

   ```bash
   curl -k -H "X-Lab-Route: primary" https://10.10.10.101/
   ```

   **Salida esperada:** Contenido HTML del backend (la iRule solo actúa cuando el valor es exactamente `secondary`).

7. Analizar el orden de procesamiento. Las iRules se ejecutan después de que los Profiles de protocolo (TCP, HTTP, SSL) hayan procesado la conexión. El Client SSL Profile descifra el tráfico TLS antes de que la iRule pueda inspeccionar las cabeceras HTTP. El HTTP Profile parsea las cabeceras antes de que el evento `HTTP_REQUEST` se dispare en la iRule.

**Verificación:**

Verificar las estadísticas de la iRule:

```bash
tmsh show ltm rule lab_irule_redirect
```

**Salida esperada:** Contadores de eventos `HTTP_REQUEST` incrementados, con `Aborts` en 0 y sin errores.

---

### Paso 8: Identificar objetos creados por una iApp de demostración

**Objetivo:** Explorar una Application Service desplegada mediante iApp, identificar los objetos creados automáticamente y documentar las restricciones de modificación directa.

**Instrucciones:**

> **Nota:** Este paso requiere que el instructor haya pre-desplegado una iApp utilizando la plantilla `f5.http` incluida en BIG-IP 17.1.2. Si no se ha desplegado, el estudiante puede crear una Application Service de demostración siguiendo los sub-pasos 1–3. Si ya existe, saltar al sub-paso 4.

1. Verificar las plantillas de iApp disponibles:

   ```bash
   tmsh list sys application template | grep "template "
   ```

   Confirmar que `f5.http` aparece en la lista.

2. Si el instructor no ha pre-desplegado la iApp, crear una Application Service de demostración mediante TMUI. Navegar a **iApps → Application Services → Applications → Create**:

   | Campo | Valor |
   |---|---|
   | Name | `demo_iapp_http` |
   | Template | `f5.http` |

   Completar los campos requeridos del asistente:
   - **Virtual Server IP:** `10.10.10.102`
   - **Virtual Server Port:** `80`
   - **Pool Members:** Agregar `10.20.20.11:80` y `10.20.20.12:80`

   Hacer clic en **Finished**.

3. Esperar a que BIG-IP cree todos los objetos asociados.

4. Navegar a **iApps → Application Services → demo_iapp_http → Components**. Documentar los objetos creados:

   | Tipo de objeto | Nombre generado (ejemplo) |
   |---|---|
   | Virtual Server | `demo_iapp_http_vs` o similar |
   | Pool | `demo_iapp_http_pool` o similar |
   | HTTP Profile | Referencia al perfil `http` o uno personalizado |
   | Monitor | Monitor asociado al pool |
   | Nodes | Nodos referenciados por los Pool Members |

   > **Nota:** Los nombres exactos dependen de la plantilla y versión. Registrar los nombres reales observados.

5. Intentar modificar directamente uno de los objetos creados por la iApp. Navegar a **Local Traffic → Virtual Servers**, localizar el Virtual Server creado por la iApp e intentar cambiar su pool o algún parámetro.

   **Resultado esperado:** BIG-IP muestra un mensaje de advertencia similar a:

   > *"This object is managed by an application service. Changes made here will be overwritten by the application service. To make changes, edit the application service."*

6. Documentar las restricciones observadas:

   - Los objetos creados por una iApp están **vinculados** a la Application Service.
   - Las modificaciones directas pueden ser sobrescritas al reconfigurar la iApp.
   - Para modificar un objeto gestionado por iApp, se debe editar la Application Service o, en casos excepcionales, cambiar la propiedad de **Strict Updates** (no recomendado en producción).

7. Verificar la propiedad de Strict Updates:

   ```bash
   tmsh list sys application service demo_iapp_http strict-updates
   ```

   **Salida esperada:** `strict-updates enabled`

8. Verificar la lista completa de objetos dependientes:

   ```bash
   tmsh list sys application service demo_iapp_http
   ```

**Verificación:**

El estudiante debe haber documentado al menos 3 tipos de objetos creados por la iApp (Virtual Server, Pool, Monitor/Profile) y haber observado el mensaje de restricción al intentar una modificación directa.

## Validación y Pruebas

Ejecutar las siguientes verificaciones para confirmar el estado final del laboratorio:

### Prueba 1: Verificación integral de persistencia por cookie

```bash
## Obtener la cookie y verificar persistencia
RESPONSE=$(curl -k -s -D - https://10.10.10.101/ -o /dev/null)
COOKIE=$(echo "$RESPONSE" | grep -i "Set-Cookie: BIGipServer" | head -1 | sed 's/Set-Cookie: //' | cut -d';' -f1)
echo "Cookie obtenida: $COOKIE"

## Verificar que 5 solicitudes van al mismo backend
for i in $(seq 1 5); do
    curl -k -s -b "$COOKIE" https://10.10.10.101/ | grep -o "web0[12]"
done
```

**Criterio de éxito:** Las 5 solicitudes retornan el mismo servidor backend.

### Prueba 2: Verificación de cifrado extremo a extremo

```bash
## Verificar handshake TLS del lado cliente
curl -k -v https://10.10.10.101/ 2>&1 | grep "subject: CN=lab.f5curso.local"

## Verificar que el pool SSL tiene miembros activos
tmsh show ltm pool pool_webapp_ssl members | grep -E "Addr|State|Monitor"
```

**Criterio de éxito:** El certificado del lado cliente muestra `CN=lab.f5curso.local` y ambos miembros del pool SSL están en estado `available` con monitor HTTPS exitoso.

### Prueba 3: Verificación de Priority Groups

```bash
## Verificar configuración de prioridades
tmsh list ltm pool pool_webapp_ssl members | grep -A2 "priority-group"

## Verificar min-active-members
tmsh list ltm pool pool_webapp_ssl min-active-members
```

**Criterio de éxito:** Prioridad 10 para `10.20.20.11:443`, prioridad 5 para `10.20.20.12:443`, `min-active-members 1`.

### Prueba 4: Verificación de la iRule

```bash
## Con cabecera X-Lab-Route: secondary
curl -k -s -H "X-Lab-Route: secondary" https://10.10.10.101/

## Sin cabecera (debe ir al pool)
HTTP_CODE=$(curl -k -s -o /dev/null -w "%{http_code}" https://10.10.10.101/)
echo "Código HTTP sin cabecera: $HTTP_CODE"
```

**Criterio de éxito:** Con la cabecera, respuesta contiene "Ruta secundaria activada"; sin la cabecera, código HTTP 200 con contenido del backend.

### Prueba 5: Verificación de X-Forwarded-For y eliminación de cabecera Server

```bash
## Verificar ausencia de cabecera Server
curl -k -sI https://10.10.10.101/ | grep -ci "^server:"
```

**Criterio de éxito:** El comando `grep` retorna `0` (sin coincidencias) o la cabecera Server muestra un valor vacío/modificado.

### Prueba 6: Caso adverso — Comportamiento con entrada no válida en iRule

```bash
## Enviar cabecera con valor vacío
curl -k -s -H "X-Lab-Route: " https://10.10.10.101/ | head -5

## Enviar cabecera con caracteres especiales
curl -k -s -H "X-Lab-Route: <script>alert(1)</script>" https://10.10.10.101/ | head -5

## Enviar múltiples cabeceras X-Lab-Route
curl -k -s -H "X-Lab-Route: primary" -H "X-Lab-Route: secondary" https://10.10.10.101/ | head -5
```

**Criterio de éxito:** En ningún caso la iRule debe generar errores (verificar `tmsh show ltm rule lab_irule_redirect` — Aborts debe ser 0). Con valor vacío o caracteres especiales, el tráfico debe pasar al pool normalmente. Con cabeceras duplicadas, documentar cuál valor toma precedencia (BIG-IP típicamente usa el primer valor encontrado).

### Resumen de estado final

```bash
## Comando consolidado de verificación
echo "=== Virtual Servers ==="
tmsh show ltm virtual vs_http_lab | grep -E "Availability|State|Destination"
tmsh show ltm virtual vs_https_lab | grep -E "Availability|State|Destination"
echo "=== Pool webapp_ssl ==="
tmsh show ltm pool pool_webapp_ssl members | grep -E "Addr|Avail|State|Priority"
echo "=== Profiles en vs_https_lab ==="
tmsh list ltm virtual vs_https_lab profiles
echo "=== iRule stats ==="
tmsh show ltm rule lab_irule_redirect | grep -E "Event|Total|Aborts"
```

## Solución de Problemas

### Problema 1: El Virtual Server HTTPS muestra estado "Offline" o no responde en puerto 443

**Síntomas:** Al ejecutar `curl -k https://10.10.10.101/` se obtiene `Connection refused` o timeout. En TMUI, `vs_https_lab` aparece con icono rojo o gris.

**Causa:** El Client SSL Profile no está correctamente asociado al Virtual Server, o el certificado/clave no se importaron correctamente. También puede ocurrir si el perfil HTTP se asignó sin un Client SSL Profile en un Virtual Server en puerto 443, lo que impide el procesamiento del handshake TLS.

**Solución:**

```bash
## Verificar los perfiles asignados
tmsh list ltm virtual vs_https_lab profiles

## Verificar que el certificado y la clave existen
tmsh list sys crypto cert lab_f5curso_cert
tmsh list sys crypto key lab_f5curso_key

## Verificar que el Client SSL Profile referencia correctamente el par
tmsh list ltm profile client-ssl clientssl_lab cert-key-chain

## Si falta el perfil SSL, reasociarlo
tmsh modify ltm virtual vs_https_lab profiles add { clientssl_lab { context clientside } }

## Verificar que el Virtual Server escucha en la IP correcta
tmsh show ltm virtual vs_https_lab | grep Destination

## Verificar logs de errores SSL
tail -20 /var/log/ltm | grep -i ssl
```

Si el certificado no fue importado correctamente, reimportar con:

```bash
tmsh install sys crypto cert lab_f5curso_cert from-local-file /var/tmp/lab_f5curso.crt
tmsh install sys crypto key lab_f5curso_key from-local-file /var/tmp/lab_f5curso.key
```

### Problema 2: Priority Group Activation no promueve el grupo de menor prioridad al desactivar el miembro principal

**Síntomas:** Tras detener Apache en `web01`, las solicitudes a `https://10.10.10.101/` fallan con error en lugar de ser redirigidas a `web02`. O bien, ambos miembros reciben tráfico simultáneamente sin respetar las prioridades.

**Causa:** El valor de `min-active-members` no está configurado correctamente, o los Priority Groups no se asignaron al pool correcto (`pool_webapp_ssl` vs. `pool_webapp`). Otra causa frecuente es que el Health Monitor no ha detectado aún la caída del servicio (intervalo de comprobación demasiado largo).

**Solución:**

```bash
## Verificar que las prioridades están en el pool correcto
tmsh list ltm pool pool_webapp_ssl members

## Verificar min-active-members
tmsh list ltm pool pool_webapp_ssl min-active-members

## Si min-active-members no está configurado, establecerlo
tmsh modify ltm pool pool_webapp_ssl min-active-members 1

## Verificar qué pool usa el Virtual Server
tmsh list ltm virtual vs_https_lab pool

## Si el VS apunta al pool incorrecto, corregir
tmsh modify ltm virtual vs_https_lab pool pool_webapp_ssl

## Verificar el estado actual de los miembros
tmsh show ltm pool pool_webapp_ssl members

## Si el monitor no ha detectado la caída, verificar el intervalo
tmsh list ltm monitor https interval timeout

## Forzar una comprobación verificando manualmente
curl -k https://10.20.20.11/ 2>&1 | head -5
```

Si ambos miembros reciben tráfico simultáneamente, verificar que los Priority Groups están asignados con valores **diferentes** (no ambos en 10) y que `min-active-members` es mayor que 0.

## Limpieza

> **Importante:** Este laboratorio establece configuraciones que serán utilizadas en los laboratorios posteriores del Módulo 4 (Alta Disponibilidad). **No eliminar** los siguientes objetos: `vs_https_lab`, `pool_webapp_ssl`, `clientssl_lab`, `serverssl_lab`, `http_lab_custom`, `persist_cookie_lab`, Priority Groups configurados y la iRule `lab_irule_redirect`.

Eliminar únicamente los objetos de la iApp de demostración si no serán necesarios:

```bash
## Eliminar la Application Service de demostración (esto elimina todos sus objetos dependientes)
tmsh delete sys application service demo_iapp_http
```

Revertir las modificaciones de diagnóstico en los servidores backend:

```bash
## En web01 y web02: eliminar el log personalizado de X-Forwarded-For
sudo sed -i '/xfflog/d' /etc/apache2/apache2.conf
sudo sed -i '/xff_access/d' /etc/apache2/sites-enabled/000-default.conf
sudo systemctl reload apache2
```

Eliminar el perfil de Source Address Affinity si no se utilizará más:

```bash
tmsh delete ltm persistence source-addr persist_srcaddr_lab
```

Guardar la configuración:

```bash
tmsh save sys config
```

## Resumen

En este laboratorio se aplicaron siete configuraciones avanzadas que amplían significativamente las capacidades del Virtual Server construido en los laboratorios anteriores:

1. **Persistence Profiles:** Se crearon y compararon Cookie Persistence (HTTP Cookie Insert) y Source Address Affinity, verificando que el primer mecanismo opera a nivel de cabeceras HTTP mientras el segundo persiste por dirección IP sin modificar el tráfico.

2. **TCP y HTTP Profiles:** Se asignaron perfiles TCP optimizados diferenciados para los lados cliente (`tcp-wan-optimized`) y servidor (`tcp-lan-optimized`), y se creó un HTTP Profile personalizado que inserta `X-Forwarded-For` y elimina la cabecera `Server`, mejorando la visibilidad del origen real y reduciendo la exposición de información del backend.

3. **Client SSL Profile:** Se generó un certificado autofirmado con OpenSSL 3.0.2, se importó al BIG-IP y se creó un Client SSL Profile para habilitar HTTPS en el Virtual Server `vs_https_lab`.

4. **Server SSL Profile:** Se habilitó cifrado extremo a extremo configurando HTTPS en los backends y asociando un Server SSL Profile al Virtual Server.

5. **Priority Groups:** Se implementó distribución de carga priorizada con failover automático, verificando que BIG-IP promueve miembros de menor prioridad únicamente cuando el grupo principal pierde disponibilidad.

6. **iRules:** Se creó lógica condicional basada en cabeceras HTTP, demostrando la capacidad de BIG-IP para tomar decisiones de enrutamiento programáticas a nivel de aplicación.

7. **iApps:** Se identificaron las dependencias y restricciones de los objetos gestionados por Application Services, comprendiendo que las modificaciones directas pueden ser sobrescritas por el framework de iApp.

Estos conceptos se alinean directamente con los dominios **F5CAB3.01** (identificación y configuración de objetos LTM) y **F5CAB3.02** (procesamiento de tráfico y perfiles). El estado resultante del laboratorio — un Virtual Server HTTPS con persistencia, perfiles optimizados, SSL de extremo a extremo, Priority Groups e iRule activa — constituye la base operativa para los laboratorios de Alta Disponibilidad del Módulo 4.

### Recursos adicionales

- [F5 BIG-IP LTM 17.1 — Persistence Profiles](https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-local-traffic-manager-profiles-reference/persistence-profiles.html)
- [F5 BIG-IP LTM 17.1 — SSL Profiles](https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-local-traffic-manager-profiles-reference/ssl-profiles.html)
- [F5 iRules DevCentral — Wiki y ejemplos](https://community.f5.com/kb/technicalarticles/irules-101---getting-started-with-irules/279711)
- [F5 BIG-IP 17.1 — Priority Group Activation](https://techdocs.f5.com/en-us/bigip-17-1-0/big-ip-local-traffic-manager-monitors-reference.html)
- [Referencia TMSH — ltm pool](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_pool.html)
