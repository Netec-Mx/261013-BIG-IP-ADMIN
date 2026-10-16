# Laboratorio 1 — Aseguramiento del acceso administrativo: Management IP, ACLs, Port Lockdown, políticas administrativas y provisionamiento

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 35 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |

## Descripción General

Este laboratorio establece la línea base de seguridad administrativa del entorno BIG-IP VE que se utilizará durante todo el curso. El estudiante configurará la Management IP, aplicará listas de control de acceso (ACLs) a los servicios SSHD y HTTPD, configurará Port Lockdown en las Self IPs del plano de datos, establecerá una política de contraseñas administrativa, verificará el estado de licenciamiento y aprovisionará el módulo LTM. Cada control se validará de forma práctica mediante TMUI y TMSH, confirmando que los accesos no autorizados son efectivamente rechazados.

## Objetivos de Aprendizaje

- [ ] Verificar y configurar la Management IP de BIG-IP y confirmar conectividad administrativa desde la estación de trabajo mediante `ping`, SSH y HTTPS (TMUI).
- [ ] Aplicar ACLs restrictivas a SSHD y HTTPD para limitar el acceso administrativo exclusivamente a la dirección autorizada `192.0.2.10` y verificar que direcciones no autorizadas son rechazadas.
- [ ] Configurar Port Lockdown en Self IPs del plano de datos utilizando los modos `Allow None`, `Allow Default` y `Allow Custom`, y validar el comportamiento de cada modo.
- [ ] Establecer una política de contraseñas administrativa (longitud mínima, complejidad, expiración) y crear un usuario con rol `Resource Administrator` que cumpla dicha política.
- [ ] Verificar el estado de licenciamiento del sistema, aprovisionar el módulo LTM a nivel `nominal` y confirmar la utilización de recursos post-provisionamiento.

## Prerrequisitos

### Conocimientos Requeridos

| Conocimiento | Detalle |
|---|---|
| Direccionamiento IP y CIDR | Cálculo de hosts, máscaras /24, distinción de subredes |
| TCP/IP básico | Puertos bien conocidos: 22/SSH, 80/HTTP, 443/HTTPS, 4353/iQuery |
| Línea de comandos | Capacidad para ejecutar SSH, `ping`, `curl` desde un terminal Linux |
| Planos de BIG-IP | Comprensión conceptual del plano de administración, control y datos (cubierto en la lección 1.1) |

### Acceso y Entorno

| Requisito | Estado esperado |
|---|---|
| Instancia BIG-IP VE 17.1.2 importada y encendida | Configuración de fábrica (factory default) |
| Estación de trabajo del estudiante | Conectividad a la red Management `192.0.2.0/24` y a la red External `10.10.10.0/24` |
| Credenciales iniciales | Usuario: `admin`, contraseña inicial proporcionada por el instructor |
| Acceso a consola virtual del hypervisor | Disponible como respaldo en caso de pérdida de conectividad de gestión |
| Software en la estación de trabajo | `curl` 8.5.0, Mozilla Firefox ESR 128.0, cliente SSH (OpenSSH 9.x) |

## Entorno de Laboratorio

### Topología de Red

```
┌──────────────┐         Management: 192.0.2.0/24          ┌──────────────────┐
│   Cliente     │ mgmt: 192.0.2.10 ──────────────────────── │  BIG-IP VE       │
│   Admin       │                                           │  mgmt: 192.0.2.245│
│               │ ext:  10.10.10.10 ────── External ─────── │  ext:  10.10.10.245│
└──────────────┘          10.10.10.0/24                     │  int:  10.20.20.245│
                                                            └────────┬─────────┘
                                                                     │ Internal
                                                              10.20.20.0/24
                                                          ┌──────────┴──────────┐
                                                    ┌─────┴─────┐         ┌─────┴─────┐
                                                    │  web01     │         │  web02     │
                                                    │ 10.20.20.11│         │ 10.20.20.12│
                                                    └───────────┘         └───────────┘
```

### Asignación de Interfaces BIG-IP VE

| Interfaz BIG-IP | VLAN | Subred | Self IP |
|---|---|---|---|
| mgmt | Management | 192.0.2.0/24 | 192.0.2.245/24 |
| 1.1 | external | 10.10.10.0/24 | 10.10.10.245/24 |
| 1.2 | internal | 10.20.20.0/24 | 10.20.20.245/24 |

### Software del Laboratorio

| Componente | Versión exacta | Fuente |
|---|---|---|
| F5 BIG-IP VE | 17.1.2 | [https://my.f5.com/manage/s/downloads](https://my.f5.com/manage/s/downloads) |
| Ubuntu Server (servidores web) | 22.04.4 LTS | [https://releases.ubuntu.com/22.04/](https://releases.ubuntu.com/22.04/) |
| Apache HTTP Server | 2.4.58 | Repositorio estándar de Ubuntu 22.04 |
| curl (estación cliente) | 8.5.0 | Repositorio estándar de Ubuntu |
| Mozilla Firefox ESR | 128.0 ESR | [https://www.mozilla.org/en-US/firefox/enterprise/](https://www.mozilla.org/en-US/firefox/enterprise/) |

### Verificación Previa del Entorno

Desde la estación de trabajo del estudiante, confirme conectividad básica antes de comenzar:

```bash
## Verificar conectividad a la red de gestión
ping -c 3 192.0.2.245

## Verificar conectividad a la red external
ping -c 3 10.10.10.245
```

> **Nota:** Si la instancia BIG-IP VE tiene configuración de fábrica, la Management IP podría no estar configurada aún. En ese caso, utilice la consola virtual del hypervisor para el Paso 1.

## Instrucciones Paso a Paso

### Paso 1: Verificar y configurar la Management IP de BIG-IP

**Objetivo:** Confirmar la configuración de la Management IP, establecer la ruta de gestión y validar la conectividad administrativa bidireccional desde la estación de trabajo.

**Instrucciones:**

1. Acceda a la consola virtual de BIG-IP VE a través del hypervisor. Inicie sesión con las credenciales iniciales:
   - Usuario: `admin`
   - Contraseña: (proporcionada por el instructor)

2. Al iniciar sesión por primera vez en una instancia con configuración de fábrica, BIG-IP puede presentar el asistente de configuración. Si se encuentra en la shell de bash, acceda a TMSH:

   ```bash
   tmsh
   ```

3. Verifique si existe una Management IP configurada:

   ```bash
   list sys management-ip
   ```

4. Si la Management IP no está configurada o difiere de `192.0.2.245/24`, configúrela:

   ```bash
   create sys management-ip 192.0.2.245/24
   ```

   Si ya existe una Management IP diferente y necesita reemplazarla, primero elimínela:

   ```bash
   delete sys management-ip all
   create sys management-ip 192.0.2.245/24
   ```

5. Configure la ruta de gestión predeterminada:

   ```bash
   create sys management-route default gateway 192.0.2.1
   ```

6. Guarde la configuración:

   ```bash
   save sys config
   ```

7. Verifique la configuración aplicada:

   ```bash
   list sys management-ip
   list sys management-route
   ```

**Resultado Esperado:**

```
sys management-ip 192.0.2.245/24 {
    description configured
}
```

```
sys management-route default {
    gateway 192.0.2.1
    network default
}
```

**Verificación:**

Desde la estación de trabajo del estudiante (`192.0.2.10`), ejecute:

```bash
ping -c 4 192.0.2.245
```

Debe obtener respuestas exitosas con 0% de pérdida de paquetes. A continuación, verifique el acceso SSH:

```bash
ssh admin@192.0.2.245 'tmsh show sys version'
```

Confirme que el output muestra `Version 17.1.2`. Finalmente, abra Firefox ESR y navegue a `https://192.0.2.245`. Acepte el certificado autofirmado y confirme que la pantalla de inicio de sesión de TMUI se muestra correctamente.

---

### Paso 2: Aplicar ACLs restrictivas a SSHD y HTTPD

**Objetivo:** Restringir el acceso a los servicios SSH y HTTPS administrativos exclusivamente a la dirección IP de la estación de trabajo autorizada (`192.0.2.10`), verificando que conexiones desde otras direcciones son rechazadas.

**Instrucciones:**

1. Desde la sesión SSH ya establecida en el Paso 1, acceda a TMSH si no se encuentra ya en la shell de TMSH:

   ```bash
   tmsh
   ```

2. Consulte la configuración actual de SSHD para conocer las direcciones permitidas:

   ```bash
   list sys sshd allow
   ```

   En configuración de fábrica, el valor típico es `All`, lo que permite conexiones SSH desde cualquier dirección.

3. Restrinja SSHD para permitir únicamente la dirección `192.0.2.10`:

   ```bash
   modify sys sshd allow replace-all-with { 192.0.2.10 }
   ```

4. Consulte la configuración actual de HTTPD:

   ```bash
   list sys httpd allow
   ```

5. Restrinja HTTPD para permitir únicamente la dirección `192.0.2.10`:

   ```bash
   modify sys httpd allow replace-all-with { 192.0.2.10 }
   ```

6. Guarde la configuración:

   ```bash
   save sys config
   ```

7. Verifique que ambas ACLs están correctamente aplicadas:

   ```bash
   list sys sshd allow
   list sys httpd allow
   ```

**Resultado Esperado:**

```
sys sshd {
    allow { 192.0.2.10 }
}
```

```
sys httpd {
    allow { 192.0.2.10 }
}
```

**Verificación:**

Desde la estación de trabajo autorizada (`192.0.2.10`), confirme que el acceso sigue funcionando:

```bash
## Verificar SSH
ssh admin@192.0.2.245 'tmsh show sys version'

## Verificar HTTPS (TMUI)
curl -sk https://192.0.2.245/tmui/login.jsp | head -5
```

El comando SSH debe retornar la versión del sistema. El comando `curl` debe retornar contenido HTML de la página de inicio de sesión (código HTTP 200).

Para verificar que una dirección **no autorizada** es rechazada, ejecute la siguiente prueba conceptual. Si la estación de trabajo tiene una segunda interfaz en la red `10.10.10.0/24`, intente acceder desde esa dirección:

```bash
## Desde la interfaz 10.10.10.10 (dirección NO autorizada en la ACL de gestión)
curl -sk --connect-timeout 5 --interface 10.10.10.10 https://192.0.2.245/tmui/login.jsp
```

El resultado esperado es un **timeout o conexión rechazada**, confirmando que la ACL de HTTPD bloquea el acceso desde direcciones no autorizadas.

> **Importante:** No cierre la sesión SSH actual desde `192.0.2.10` hasta haber confirmado que la ACL funciona correctamente. Si comete un error en la ACL y pierde acceso, utilice la consola virtual del hypervisor para corregir la configuración.

---

### Paso 3: Configurar VLANs, Self IPs y Port Lockdown

**Objetivo:** Crear las VLANs `external` e `internal`, asignar Self IPs y configurar Port Lockdown en cada Self IP para controlar los servicios accesibles desde el plano de datos.

**Instrucciones:**

1. Desde TMSH, cree la VLAN `external` asociada a la interfaz `1.1`:

   ```bash
   create net vlan external interfaces add { 1.1 { untagged } }
   ```

2. Cree la VLAN `internal` asociada a la interfaz `1.2`:

   ```bash
   create net vlan internal interfaces add { 1.2 { untagged } }
   ```

3. Verifique las VLANs creadas:

   ```bash
   list net vlan
   ```

4. Cree la Self IP para la VLAN `external` con Port Lockdown en modo `allow-none`:

   ```bash
   create net self 10.10.10.245/24 vlan external allow-service none
   ```

5. Cree la Self IP para la VLAN `internal` con Port Lockdown en modo `allow-default`:

   ```bash
   create net self 10.20.20.245/24 vlan internal allow-service default
   ```

6. Verifique las Self IPs y su configuración de Port Lockdown:

   ```bash
   list net self
   ```

**Resultado Esperado:**

```
net self 10.10.10.245/24 {
    address 10.10.10.245/24
    allow-service {
        none
    }
    traffic-group traffic-group-local-only
    vlan external
}
net self 10.20.20.245/24 {
    address 10.20.20.245/24
    allow-service {
        default
    }
    traffic-group traffic-group-local-only
    vlan internal
}
```

7. **Prueba de Port Lockdown — Allow None:** Desde la estación de trabajo (`10.10.10.10`), intente conectarse por SSH a la Self IP external:

   ```bash
   ssh -o ConnectTimeout=5 admin@10.10.10.245
   ```

   **Resultado esperado:** Timeout o conexión rechazada. El modo `allow-none` bloquea todos los servicios en la Self IP.

8. **Prueba de Port Lockdown — Allow Default:** Desde uno de los servidores web (si tiene acceso SSH a `web01` en `10.20.20.11`), intente un ping ICMP a la Self IP internal:

   ```bash
   ping -c 3 10.20.20.245
   ```

   **Resultado esperado:** Respuestas exitosas. El modo `allow-default` permite ICMP, entre otros servicios predeterminados (SSH en puerto 22, HTTPS en 443, iQuery en 4353, entre otros).

9. **Demostración de Port Lockdown — Allow Custom:** Modifique la Self IP external para permitir únicamente ICMP (para pruebas de conectividad) y HTTPS en puerto 443 (para acceso de monitoreo, si fuera necesario):

   ```bash
   modify net self 10.10.10.245/24 allow-service replace-all-with { icmp tcp:443 }
   ```

10. Verifique la configuración actualizada:

    ```bash
    list net self 10.10.10.245/24 allow-service
    ```

**Resultado Esperado:**

```
net self 10.10.10.245/24 {
    allow-service {
        icmp
        tcp:443
    }
}
```

11. Valide que ICMP ahora funciona desde la estación de trabajo hacia la Self IP external:

    ```bash
    ping -c 3 10.10.10.245
    ```

    **Resultado esperado:** Respuestas exitosas.

12. Valide que SSH sigue bloqueado en la Self IP external (no está en la lista `allow-service`):

    ```bash
    ssh -o ConnectTimeout=5 admin@10.10.10.245
    ```

    **Resultado esperado:** Timeout o conexión rechazada.

13. Restaure la Self IP external al modo `allow-none` para mantener la postura de seguridad más restrictiva en el plano de datos:

    ```bash
    modify net self 10.10.10.245/24 allow-service none
    ```

14. Guarde la configuración:

    ```bash
    save sys config
    ```

**Verificación Final del Paso:**

```bash
list net self
```

Confirme que `10.10.10.245/24` tiene `allow-service none` y `10.20.20.245/24` tiene `allow-service default`.

---

### Paso 4: Establecer política de contraseñas y crear usuario administrativo

**Objetivo:** Configurar una política de contraseñas que exija longitud mínima, complejidad y expiración, y crear un usuario con rol `Resource Administrator` que valide el cumplimiento de dicha política.

**Instrucciones:**

1. Desde TMSH, consulte la política de contraseñas actual:

   ```bash
   list auth password-policy
   ```

   En configuración de fábrica, los valores suelen ser permisivos o no estar configurados.

2. Establezca una política de contraseñas con los siguientes requisitos:
   - Longitud mínima: 8 caracteres
   - Al menos una letra mayúscula requerida
   - Al menos una letra minúscula requerida
   - Al menos un dígito numérico requerido
   - Al menos un carácter especial requerido
   - Expiración de contraseña: 90 días
   - Número máximo de intentos de inicio de sesión fallidos: 5

   ```bash
   modify auth password-policy minimum-length 8
   modify auth password-policy required-uppercase 1
   modify auth password-policy required-lowercase 1
   modify auth password-policy required-numeric 1
   modify auth password-policy required-special 1
   modify auth password-policy password-memory 0
   modify auth password-policy expiration-warning 7
   modify auth password-policy max-login-failures 5
   modify auth password-policy max-duration 90
   ```

3. Verifique la política aplicada:

   ```bash
   list auth password-policy
   ```

**Resultado Esperado:**

```
auth password-policy {
    expiration-warning 7
    max-duration 90
    max-login-failures 5
    minimum-length 8
    password-memory 0
    policy-enforcement enabled
    required-lowercase 1
    required-numeric 1
    required-special 1
    required-uppercase 1
}
```

> **Nota:** Algunos atributos como `policy-enforcement` pueden habilitarse automáticamente al configurar los parámetros. Si `policy-enforcement` aparece como `disabled`, actívelo:
> ```bash
> modify auth password-policy policy-enforcement enabled
> ```

4. Cree un usuario administrativo adicional con rol `Resource Administrator`. Primero, intente usar una contraseña **débil** para verificar que la política la rechaza:

   ```bash
   create auth user laboperator partition-access add { all-partitions { role resource-admin } } password "abc"
   ```

   **Resultado esperado:** Error indicando que la contraseña no cumple con los requisitos de la política (longitud insuficiente, faltan caracteres requeridos).

5. Ahora cree el usuario con una contraseña que cumpla la política:

   ```bash
   create auth user laboperator partition-access add { all-partitions { role resource-admin } } password "L4bOp$2025x"
   ```

   > **Nota:** La contraseña `L4bOp$2025x` cumple: ≥8 caracteres, mayúscula (L, O), minúscula (ab, p, x), dígito (4, 2, 0, 2, 5), carácter especial ($).

6. Verifique que el usuario fue creado:

   ```bash
   list auth user laboperator
   ```

**Resultado Esperado:**

```
auth user laboperator {
    description none
    partition-access {
        all-partitions {
            role resource-admin
        }
    }
    shell none
}
```

7. Guarde la configuración:

   ```bash
   save sys config
   ```

**Verificación:**

Desde la estación de trabajo, valide que el nuevo usuario puede iniciar sesión en TMUI:

1. Abra Firefox ESR y navegue a `https://192.0.2.245`.
2. Inicie sesión con:
   - Usuario: `laboperator`
   - Contraseña: `L4bOp$2025x`
3. Confirme que el usuario accede a TMUI pero con permisos limitados correspondientes al rol `Resource Administrator` (no puede modificar configuraciones del sistema, pero puede visualizar estadísticas y recursos).
4. Cierre la sesión del usuario `laboperator`.

---

### Paso 5: Verificar el estado de licenciamiento y Service Check Date

**Objetivo:** Inspeccionar el estado de la licencia de BIG-IP, identificar los módulos licenciados, la fecha de expiración del servicio (Service Check Date) y distinguir entre módulos licenciados y módulos aprovisionados.

**Instrucciones:**

1. Desde TMSH (sesión SSH como `admin`), consulte el estado de la licencia:

   ```bash
   show sys license
   ```

2. Identifique en el output los siguientes campos clave:
   - **Licensed Version**: debe coincidir con `17.1.2`
   - **Service Check Date**: fecha hasta la cual el contrato de soporte es válido
   - **Active Modules**: lista de módulos incluidos en la licencia
   - **Registration Key**: clave de registro del sistema

3. Para una vista resumida de los módulos licenciados:

   ```bash
   show sys license | grep -i "module"
   ```

4. Compare los módulos **licenciados** con los módulos **aprovisionados** actualmente:

   ```bash
   list sys provision
   ```

   En configuración de fábrica, es probable que ningún módulo esté aprovisionado a nivel `nominal` o superior.

**Resultado Esperado:**

El comando `show sys license` produce un output extenso. Los campos críticos a documentar son:

```
Licensed Version    17.1.2
...
Service Check Date  <fecha del contrato — varía según la licencia>
...
Active Modules
  LTM, ...  (lista de módulos incluidos en la licencia)
```

El comando `list sys provision` en estado de fábrica muestra:

```
sys provision ltm {
    level none
}
```

(Pueden aparecer otros módulos con `level none`.)

**Verificación:**

Confirme que la licencia está activa ejecutando:

```bash
show sys license | grep -i "license state"
```

El resultado debe indicar un estado válido (por ejemplo, `LICENSED`). Si la licencia no está activa, consulte con el instructor antes de continuar.

Adicionalmente, verifique el estado de licencia desde TMUI:
1. Navegue a **System > License** en TMUI.
2. Confirme que la pantalla muestra la clave de registro, los módulos activos y la fecha del servicio.

---

### Paso 6: Aprovisionar el módulo LTM y verificar la utilización de recursos

**Objetivo:** Aprovisionar el módulo LTM a nivel `nominal`, observar el reinicio de servicios TMOS resultante y verificar la utilización de recursos del sistema post-provisionamiento.

**Instrucciones:**

1. Antes de aprovisionar, registre el uso actual de recursos como línea base:

   ```bash
   show sys provision
   show sys performance all-stats
   ```

   Tome nota de los valores de CPU y memoria reportados.

2. Aprovisione el módulo LTM a nivel `nominal`:

   ```bash
   modify sys provision ltm level nominal
   ```

   > **Advertencia:** Este comando provocará un reinicio de los servicios TMOS. La sesión SSH puede interrumpirse temporalmente. El proceso puede tardar entre 30 segundos y 2 minutos. **No reinicie la máquina virtual manualmente.**

3. Espere a que los servicios se restablezcan. Puede monitorear la disponibilidad intentando reconectar por SSH:

   ```bash
   # Ejecute desde la estación de trabajo, repetidamente si es necesario
   ssh admin@192.0.2.245 'tmsh show sys version'
   ```

   Cuando el comando retorne la versión correctamente, los servicios están disponibles.

4. Verifique el estado de provisionamiento:

   ```bash
   show sys provision
   ```

**Resultado Esperado:**

```
---------------------------------
Sys::Provision
Module   CPU (%)   Memory (MB)   Host-Memory (MB)   Disk (MB)
ltm            1          ...           ...              ...
---------------------------------
```

El módulo `ltm` debe aparecer con `level nominal` y valores asignados de CPU y memoria.

5. Confirme el provisionamiento con el comando de listado:

   ```bash
   list sys provision ltm
   ```

**Resultado Esperado:**

```
sys provision ltm {
    level nominal
}
```

6. Verifique la utilización de recursos post-provisionamiento:

   ```bash
   show sys performance all-stats
   ```

   Compare los valores de CPU y memoria con los registrados antes del provisionamiento. Es normal observar un incremento en la utilización de memoria, ya que LTM reserva recursos para sus subsistemas.

7. Verifique que TMUI refleja el provisionamiento. Navegue en Firefox ESR a `https://192.0.2.245` e inicie sesión como `admin`. Confirme que:
   - El menú lateral izquierdo ahora incluye la sección **Local Traffic** (indicativo de que LTM está aprovisionado).
   - Navegue a **System > Resource Provisioning** y confirme que LTM aparece como `Nominal`.

8. Guarde la configuración final:

   ```bash
   save sys config
   ```

**Verificación:**

Ejecute el siguiente comando como verificación integral del estado del sistema:

```bash
show sys version
show sys license | grep -i "license state"
list sys provision ltm
list sys management-ip
list sys sshd allow
list sys httpd allow
list net self
list auth user laboperator
list auth password-policy
```

Cada comando debe retornar los valores configurados durante este laboratorio. Documente los outputs como evidencia de la línea base del entorno.

## Validación y Pruebas

### Prueba 1: Validación integral de conectividad administrativa

Desde la estación de trabajo (`192.0.2.10`), ejecute la siguiente secuencia completa:

```bash
## 1. Ping a Management IP
ping -c 2 192.0.2.245

## 2. Acceso SSH
ssh admin@192.0.2.245 'tmsh show sys version'

## 3. Acceso TMUI (verificar código HTTP 200)
curl -sk -o /dev/null -w "%{http_code}" https://192.0.2.245/tmui/login.jsp
```

**Criterio de éxito:** Ping responde, SSH retorna versión `17.1.2`, curl retorna `200`.

### Prueba 2: Validación de ACLs — Rechazo de acceso no autorizado

Intente acceder a la Management IP desde la interfaz external de la estación de trabajo (si está disponible):

```bash
## Intento SSH desde dirección no autorizada
ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no -b 10.10.10.10 admin@192.0.2.245 'echo test' 2>&1
```

**Criterio de éxito:** Timeout o conexión rechazada. Si la estación de trabajo no tiene ruta desde `10.10.10.10` hacia `192.0.2.245`, el resultado será igualmente un fallo de conexión, lo cual es consistente con el diseño de aislamiento de redes.

### Prueba 3: Validación de Port Lockdown

```bash
## SSH a Self IP external (debe fallar — allow-service none)
ssh -o ConnectTimeout=5 admin@10.10.10.245 2>&1 | head -2

## Ping a Self IP internal desde web01 (debe funcionar — allow-service default)
## Ejecutar desde web01 si tiene acceso:
## ping -c 2 10.20.20.245
```

**Criterio de éxito:** SSH a `10.10.10.245` falla con timeout. Ping a `10.20.20.245` desde la red interna responde exitosamente.

### Prueba 4: Validación de política de contraseñas (caso adverso)

Desde TMSH, intente crear un usuario con una contraseña que viole la política:

```bash
## Contraseña sin caracteres especiales ni mayúsculas
create auth user testbadpw partition-access add { all-partitions { role guest } } password "simple123"
```

**Criterio de éxito:** El sistema debe rechazar la creación del usuario con un mensaje de error indicando incumplimiento de la política de contraseñas. Si el usuario se crea exitosamente, la política no está correctamente aplicada — revise que `policy-enforcement` esté habilitado.

> **Limpieza de la prueba adversa:** Si el usuario se creó por error, elimínelo:
> ```bash
> delete auth user testbadpw
> ```

### Prueba 5: Validación de provisionamiento LTM

```bash
## Verificar que LTM está aprovisionado a nivel nominal
tmsh list sys provision ltm

## Verificar que los objetos LTM están disponibles
tmsh show ltm virtual
```

**Criterio de éxito:** `ltm` muestra `level nominal`. El comando `show ltm virtual` se ejecuta sin errores (puede retornar una lista vacía si no hay Virtual Servers configurados, lo cual es correcto en este punto).

### Prueba 6: Caso adverso — Verificación de información inexistente

Esta prueba evalúa el comportamiento del sistema al consultar objetos que no existen:

```bash
## Consultar un Virtual Server inexistente
tmsh show ltm virtual vs_inexistente

## Consultar un usuario inexistente
tmsh list auth user usuario_fantasma
```

**Criterio de éxito:** TMSH debe retornar un mensaje de error claro como `was not found` o un output vacío, sin provocar fallos en el sistema. Esto confirma que los comandos de consulta son seguros para troubleshooting exploratorio.

## Solución de Problemas

### Problema 1: Pérdida de acceso SSH/HTTPS tras aplicar ACLs de SSHD/HTTPD

**Síntomas:** Después de ejecutar `modify sys sshd allow` o `modify sys httpd allow`, la sesión SSH se desconecta y no es posible reconectar. TMUI tampoco responde desde el navegador.

**Causa:** Se configuró una dirección IP incorrecta en la ACL (por ejemplo, un error tipográfico como `192.0.2.100` en lugar de `192.0.2.10`), o se utilizó `replace-all-with` con una lista vacía, bloqueando todo acceso.

**Solución:**

1. Acceda a la consola virtual de BIG-IP VE a través del hypervisor.
2. Inicie sesión como `admin` con la contraseña configurada.
3. Acceda a TMSH:
   ```bash
   tmsh
   ```
4. Corrija la ACL de SSHD:
   ```bash
   modify sys sshd allow replace-all-with { 192.0.2.10 }
   ```
5. Corrija la ACL de HTTPD:
   ```bash
   modify sys httpd allow replace-all-with { 192.0.2.10 }
   ```
6. Guarde la configuración:
   ```bash
   save sys config
   ```
7. Desde la estación de trabajo, verifique que SSH y HTTPS funcionan nuevamente:
   ```bash
   ssh admin@192.0.2.245 'tmsh show sys version'
   curl -sk https://192.0.2.245/tmui/login.jsp | head -3
   ```

**Prevención:** Siempre mantenga una sesión de consola virtual abierta como respaldo antes de modificar ACLs de acceso administrativo.

---

### Problema 2: El provisionamiento de LTM no completa y TMUI muestra error de carga

**Síntomas:** Después de ejecutar `modify sys provision ltm level nominal`, la sesión SSH se interrumpe (esperado), pero tras varios minutos TMUI muestra una página de error o un mensaje de "service unavailable". Los intentos de SSH retornan `Connection refused`.

**Causa:** El reinicio de servicios TMOS tras el provisionamiento puede tardar más de lo habitual si la máquina virtual tiene recursos limitados (por debajo de los 4 GiB de RAM recomendados) o si el hypervisor está bajo carga. En casos extremos, el proceso `mcpd` puede no completar la inicialización.

**Solución:**

1. Espere al menos 5 minutos antes de tomar acciones correctivas. El provisionamiento en entornos virtuales con recursos limitados puede ser lento.
2. Acceda a la consola virtual del hypervisor e intente iniciar sesión.
3. Si la consola responde, verifique el estado de los servicios:
   ```bash
   tmsh show sys service
   ```
   Busque servicios en estado `restarting` o `failed`.
4. Si `mcpd` está en estado anormal, espere a que complete su ciclo. Puede monitorear los logs:
   ```bash
   tail -f /var/log/ltm
   ```
5. Si después de 10 minutos el sistema no se estabiliza, revise que la máquina virtual tiene al menos 4 GiB de RAM asignados:
   ```bash
   tmsh show sys hardware | grep -i "memory"
   ```
6. Si los recursos son insuficientes, apague la VM, aumente la RAM a 4 GiB en el hypervisor, reinicie y verifique nuevamente:
   ```bash
   tmsh show sys provision
   tmsh list sys provision ltm
   ```

**Prevención:** Asegúrese de que la instancia BIG-IP VE cumple con los requisitos mínimos de 4 vCPU y 4 GiB de RAM antes de aprovisionar módulos.

## Limpieza

Este laboratorio establece la **línea base del entorno** para todos los laboratorios subsiguientes del curso. **No deshaga las configuraciones realizadas.** Los siguientes elementos deben permanecer configurados:

| Elemento | Valor | Persistir |
|---|---|---|
| Management IP | 192.0.2.245/24 | ✅ Sí |
| Ruta de gestión | gateway 192.0.2.1 | ✅ Sí |
| ACL SSHD | 192.0.2.10 | ✅ Sí |
| ACL HTTPD | 192.0.2.10 | ✅ Sí |
| VLAN external | interfaz 1.1 | ✅ Sí |
| VLAN internal | interfaz 1.2 | ✅ Sí |
| Self IP external | 10.10.10.245/24, allow-service none | ✅ Sí |
| Self IP internal | 10.20.20.245/24, allow-service default | ✅ Sí |
| Política de contraseñas | longitud 8, complejidad habilitada, expiración 90 días | ✅ Sí |
| Usuario laboperator | rol resource-admin | ✅ Sí |
| Provisionamiento LTM | level nominal | ✅ Sí |

Si creó el usuario de prueba `testbadpw` durante la validación adversa, elimínelo:

```bash
tmsh delete auth user testbadpw
tmsh save sys config
```

Confirme la integridad de la configuración final:

```bash
tmsh save sys config
tmsh show sys version
```

## Resumen

En este laboratorio se estableció la línea base de seguridad administrativa del entorno BIG-IP VE 17.1.2 que se utilizará a lo largo del curso. Se completaron los siguientes hitos:

1. **Management IP:** Se configuró `192.0.2.245/24` con ruta de gestión hacia `192.0.2.1`, verificando conectividad bidireccional con la estación de trabajo.
2. **ACLs de SSHD y HTTPD:** Se restringió el acceso administrativo exclusivamente a `192.0.2.10`, validando que direcciones no autorizadas son rechazadas.
3. **VLANs y Self IPs con Port Lockdown:** Se crearon las VLANs `external` e `internal`, se asignaron Self IPs y se configuró Port Lockdown. Se probaron los tres modos (`Allow None`, `Allow Default`, `Allow Custom`) y se dejó la configuración más restrictiva en la Self IP external.
4. **Política de contraseñas:** Se establecieron requisitos de longitud, complejidad y expiración. Se validó que contraseñas débiles son rechazadas. Se creó el usuario `laboperator` con rol `Resource Administrator`.
5. **Licenciamiento:** Se verificó el estado de la licencia, la Service Check Date y la distinción entre módulos licenciados y aprovisionados.
6. **Provisionamiento de LTM:** Se aprovisionó el módulo LTM a nivel `nominal`, se observó el reinicio de servicios TMOS y se confirmó la asignación de recursos.

Estos controles implementan los principios de separación de planos (administración, control y datos) y defensa en profundidad cubiertos en la lección 1.1, asegurando que el acceso administrativo al BIG-IP está debidamente restringido antes de configurar servicios de producción en laboratorios posteriores.

### Alineación con Objetivos de Certificación

| Objetivo | Cobertura en este laboratorio |
|---|---|
| **F5CAB1.01** — Identificar estado de licencia, Service Check Date, módulos | Paso 5: verificación de licencia y Service Check Date |
| **F5CAB1.02** — Administrar plano de gestión (Management IP, ACLs, Port Lockdown, provisionamiento) | Pasos 1–4 y 6: configuración completa del plano de gestión |
| **F5CAB1.05** — Procedimientos iniciales de troubleshooting con TMUI y TMSH | Todos los pasos: uso sistemático de TMUI y TMSH para configuración y validación |

### Recursos Adicionales

- [F5 BIG-IP TMOS — Guía de administración de plataforma (v17.1.x)](https://my.f5.com/manage/s/article/K9476)
- [Referencia de comandos TMSH — sys sshd, sys httpd, net self](https://clouddocs.f5.com/cli/tmsh-reference/latest/)
- [K13309: Restricción de acceso administrativo a BIG-IP](https://my.f5.com/manage/s/article/K13309)
- [K17333: Configuración de Port Lockdown en Self IPs](https://my.f5.com/manage/s/article/K17333)
- [K15497: Provisionamiento de módulos en BIG-IP](https://my.f5.com/manage/s/article/K15497)

---

# Laboratorio 2 — Gestión del ciclo de vida del sistema: licenciamiento, Service Check Date, boot locations, software images y preparación de la actualización

## Metadatos

| Campo | Valor |
|---|---|
| **Duración** | 35 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar |
| **Alineación curricular** | F5CAB1.03 – F5CAB1.05; relación con F5CAB4.09 |
| **Versión TMOS base** | 17.1.1.3 (pre-actualización) |
| **Versión TMOS objetivo** | 17.1.2.1 (imagen ISO a instalar) |

## Descripción General

Este laboratorio guía al estudiante a través del ciclo de vida operativo de un BIG-IP VE: validación del estado de licencia y Service Check Date, revisión del provisionamiento de módulos, gestión de boot locations e imágenes de software, instalación de una nueva versión en un volumen inactivo y documentación de un procedimiento de actualización controlada con estrategia de rollback. El punto de partida es un BIG-IP con TMOS 17.1.1.3 operativo, licenciado y con acceso administrativo funcional, tal como quedó configurado en el Laboratorio 1 (01-00-01).

## Objetivos de Aprendizaje

- [ ] Validar el estado de licencia BIG-IP mediante TMUI y TMSH, identificando Registration Key, Service Check Date, módulos licenciados y fecha de vencimiento del servicio.
- [ ] Revisar y documentar el estado de provisionamiento de módulos, diferenciando módulos licenciados de módulos efectivamente provisionados y sus niveles de asignación de recursos.
- [ ] Identificar boot locations disponibles, determinar el volumen activo e instalar una imagen de software BIG-IP en un volumen inactivo, verificando la integridad de la imagen mediante checksum.
- [ ] Documentar un procedimiento de actualización controlada que incluya checklist pre-actualización, pasos de instalación, criterios de validación post-actualización y procedimiento de rollback.

## Prerrequisitos

### Conocimientos requeridos

| Área | Detalle |
|---|---|
| Laboratorio anterior | Haber completado Lab 01-00-01: Management IP (192.0.2.245/24), hostname, DNS, NTP, acceso TMUI y SSH funcionales |
| TMSH básico | Navegación de módulos (`sys`, `ltm`), comandos `show`, `list`, `install` |
| Transferencia de archivos | Uso de SCP o SFTP desde la estación de trabajo hacia BIG-IP |
| Conceptos de licenciamiento | Registration Key, Service Check Date (SCD), reactivación online/offline |
| Sistema de archivos BIG-IP | Directorio `/shared/images/` para imágenes ISO |

### Acceso requerido

| Recurso | Dirección / Método | Credenciales |
|---|---|---|
| TMUI (HTTPS) | `https://192.0.2.245` | `admin` / contraseña configurada en Lab 01-00-01 |
| SSH (TMSH) | `ssh admin@192.0.2.245` | `admin` / contraseña configurada en Lab 01-00-01 |
| Cliente de administración | 192.0.2.10 (red Management) | Acceso local |
| Imagen ISO BIG-IP 17.1.2.1 | Archivo `BIG-IP-17.1.2.1-0.0.4.iso` proporcionado por el instructor o descargado de [my.f5.com](https://my.f5.com) | Cuenta F5 con acceso a descargas |
| Archivo de licencia (offline) | Archivo `.lic` proporcionado por el instructor (si no hay conectividad a activate.f5.com) | N/A |

> **Nota sobre versiones exactas:** Este laboratorio utiliza F5 BIG-IP VE 17.1.1.3 como versión base y la imagen `BIG-IP-17.1.2.1-0.0.4.iso` como versión objetivo. El checksum SHA256 de la imagen debe obtenerse de la página de descargas de [my.f5.com — Downloads](https://my.f5.com/manage/s/downloads). No sustituir por otra versión sin revalidar los procedimientos.

## Entorno de Laboratorio

### Topología de red

```
┌──────────────┐        Management: 192.0.2.0/24        ┌──────────────────┐
│   Cliente     │───────────────────────────────────────│   BIG-IP VE      │
│  Admin/Prueba │  192.0.2.10              192.0.2.245  │  TMOS 17.1.1.3   │
│              │                                        │                  │
│  10.10.10.10 │───── External: 10.10.10.0/24 ─────────│  10.10.10.245    │
└──────────────┘                                        │                  │
                                                        │  10.20.20.245    │
                           Internal: 10.20.20.0/24      │                  │
                    ┌───────────────────────────────────│                  │
                    │                                    └──────────────────┘
              ┌─────┴──────┐    ┌──────────────┐
              │   web01    │    │    web02      │
              │ 10.20.20.11│    │ 10.20.20.12  │
              └────────────┘    └──────────────┘
```

### Componentes de software

| Componente | Versión exacta | Función en el laboratorio |
|---|---|---|
| F5 BIG-IP VE | TMOS 17.1.1.3 (base) | Plataforma bajo gestión |
| Imagen ISO objetivo | BIG-IP-17.1.2.1-0.0.4.iso | Imagen a instalar en volumen inactivo |
| Ubuntu Server (cliente) | 22.04.4 LTS | Estación de administración |
| OpenSSH (cliente) | Incluido en Ubuntu 22.04.4 | Transferencia SCP y acceso SSH |
| curl | 8.5.0 | Validaciones HTTP opcionales |
| Mozilla Firefox ESR | 128.0 ESR | Acceso a TMUI |

### Verificación previa del entorno

Desde el cliente de administración (192.0.2.10), confirme la conectividad antes de iniciar:

```bash
## Verificar acceso SSH al BIG-IP
ssh admin@192.0.2.245 'tmsh show sys version' 

## Verificar acceso HTTPS (código 200 o 302 esperado)
curl -sk -o /dev/null -w "%{http_code}" https://192.0.2.245
```

**Resultado esperado:** El comando SSH devuelve la versión TMOS 17.1.1.3. El comando curl devuelve `200` o `302`.

Si alguna verificación falla, revise que el Lab 01-00-01 se completó correctamente antes de continuar.

## Instrucciones Paso a Paso

### Paso 1: Validar el estado de licencia mediante TMUI y TMSH

**Objetivo:** Identificar el Registration Key, la fecha de Service Check Date (SCD), los módulos licenciados y la fecha de vencimiento del servicio de soporte activo.

**Instrucciones:**

1. Abra Firefox en el cliente de administración y navegue a `https://192.0.2.245`. Inicie sesión con las credenciales de administrador.

2. En TMUI, navegue a **System > License**. Examine la información mostrada y registre los siguientes campos en su documento de trabajo:

   | Campo | Valor observado |
   |---|---|
   | Registration Key | |
   | Licensed Date | |
   | Service Check Date | |
   | Licensed Version (platform) | |
   | Módulos listados bajo "Active Modules" | |

3. Conéctese al BIG-IP por SSH y ejecute los siguientes comandos TMSH para obtener información detallada de la licencia:

   ```bash
   tmsh show sys license
   ```

4. En la salida del comando anterior, localice y documente:
   - La línea `Service Check Date` — esta fecha determina la versión máxima de TMOS elegible para instalación.
   - La sección `Active Modules` — lista de módulos incluidos en la licencia.
   - La línea `License Date` — fecha de registro original de la licencia.
   - La línea `Registration Key`.

5. Ejecute el siguiente comando para obtener una vista compacta de los módulos licenciados:

   ```bash
   tmsh list sys license | grep -E "^  (auth|optional|perf|platform)" 
   ```

6. Valide la relación entre el Service Check Date y la versión objetivo (17.1.2.1). La regla es: **el SCD debe ser igual o posterior a la fecha de publicación de la imagen de software objetivo**. Consulte la fecha de publicación de TMOS 17.1.2.1 en [AskF5 Release Notes — K000137353](https://my.f5.com/manage/s/article/K000137353) `[ENLACE OFICIAL — verificar disponibilidad]`.

   > **Criterio de elegibilidad:** Si el SCD es anterior a la fecha de publicación de 17.1.2.1, será necesario reactivar la licencia antes de proceder con la actualización.

**Resultado esperado:**

```
Sys::License
  Registration Key     : XXXXX-XXXXX-XXXXX-XXXXX-XXXXXXX
  Licensed Date        : YYYY/MM/DD
  Service Check Date   : YYYY/MM/DD
  ...
  Active Modules
    LTM, ...  (lista de módulos según licencia)
```

**Verificación:**

- [ ] El Registration Key está documentado.
- [ ] El Service Check Date está identificado y comparado con la fecha de publicación de TMOS 17.1.2.1.
- [ ] La lista de módulos licenciados está registrada.
- [ ] Se determinó si la licencia es elegible para la actualización a 17.1.2.1 o si requiere reactivación.

---

### Paso 2: Comprender el proceso de reactivación de licencia

**Objetivo:** Practicar el flujo de reactivación de licencia (online y offline) y comprender cuándo es necesario reactivar para actualizar el Service Check Date.

**Instrucciones:**

1. En TMUI, navegue a **System > License** y observe el botón **Re-activate**. **No ejecute la reactivación a menos que el instructor lo indique** — este paso es de comprensión del flujo.

2. Documente los dos métodos de reactivación disponibles:

   **Método A — Reactivación online (Automatic):**
   - BIG-IP contacta `activate.f5.com` (TCP/443) desde la Management IP.
   - Requiere que el BIG-IP tenga resolución DNS funcional y ruta hacia Internet desde la red de gestión.
   - El SCD se actualiza automáticamente si el contrato de servicio está vigente.

   **Método B — Reactivación offline (Manual):**
   - Se genera un archivo `dossier` desde el BIG-IP.
   - El administrador carga el dossier en `https://activate.f5.com/license/` desde un equipo con acceso a Internet.
   - Se descarga el archivo `.lic` resultante.
   - Se carga el archivo `.lic` en el BIG-IP mediante TMUI o TMSH.

3. Desde TMSH, genere un dossier (sin enviarlo) para familiarizarse con el comando:

   ```bash
   tmsh show sys license dossier
   ```

   > **Nota:** El dossier es una cadena codificada en Base64 que identifica de forma única al dispositivo. No contiene información sensible de configuración, pero sí identifica la plataforma.

4. Si el instructor proporcionó un archivo `.lic` para ejercicio offline, documente el procedimiento de carga:

   ```bash
   # Desde el cliente, copiar el archivo .lic al BIG-IP
   scp /ruta/local/bigip.lic admin@192.0.2.245:/config/bigip.license
   
   # Desde TMSH, recargar la licencia
   tmsh install sys license file /config/bigip.license
   ```

   > **Importante:** En un entorno de producción, la reactivación de licencia puede reiniciar servicios. Planifique una ventana de mantenimiento adecuada.

5. Después de una reactivación (real o simulada), verifique el nuevo SCD:

   ```bash
   tmsh show sys license | grep "Service Check Date"
   ```

**Resultado esperado:**

El estudiante comprende ambos flujos de reactivación y puede determinar cuándo es necesario reactivar la licencia antes de una actualización. Si se ejecutó la reactivación, el SCD muestra una fecha actualizada.

**Verificación:**

- [ ] Se documentaron los dos métodos de reactivación (online y offline).
- [ ] Se generó un dossier de ejemplo (sin enviar).
- [ ] Se comprende la relación: SCD actualizado → elegibilidad para instalar versiones más recientes.

---

### Paso 3: Revisar el estado de provisionamiento de módulos

**Objetivo:** Identificar qué módulos están licenciados versus provisionados, comprender los niveles de provisionamiento y documentar el estado actual como baseline.

**Instrucciones:**

1. En TMUI, navegue a **System > Resource Provisioning**. Observe la tabla de módulos y sus niveles asignados.

2. Desde TMSH, ejecute:

   ```bash
   tmsh show sys provision
   ```

   Este comando muestra el estado de provisionamiento efectivo de cada módulo, incluyendo la asignación de CPU y memoria.

3. Ejecute también la vista de configuración:

   ```bash
   tmsh list sys provision
   ```

   Compare la salida de `show` (estado operativo) con `list` (configuración declarada).

4. Documente el estado actual en la siguiente tabla:

   | Módulo | ¿Licenciado? | Nivel de provisionamiento | CPU (%) | Memoria (%) |
   |---|---|---|---|---|
   | LTM | | | | |
   | GTM/DNS | | | | |
   | ASM | | | | |
   | APM | | | | |
   | AFM | | | | |
   | (otros) | | | | |

5. Comprenda los niveles de provisionamiento disponibles:

   | Nivel | Descripción |
   |---|---|
   | `none` | Módulo no provisionado; no consume recursos |
   | `minimum` | Funcionalidad básica; mínimo consumo de recursos |
   | `nominal` | Nivel estándar recomendado; asignación compartida de recursos |
   | `dedicated` | Todos los recursos disponibles se asignan a este módulo |

6. Verifique el consumo actual de recursos del sistema:

   ```bash
   tmsh show sys memory
   tmsh show sys cpu
   ```

   > **Concepto clave:** Un módulo puede estar **licenciado** (incluido en la licencia) pero **no provisionado** (nivel `none`). Solo los módulos provisionados consumen recursos y ofrecen funcionalidad operativa. Antes de una actualización, documente qué módulos están provisionados para validar que funcionen correctamente después del upgrade.

**Resultado esperado:**

```
Sys::Provision
Module   CPU (%)   Memory (%)   Host-Memory (%)   Disk (MB)   Level
ltm           1            0                 0           0    nominal
...
```

**Verificación:**

- [ ] La tabla de provisionamiento está completa con todos los módulos visibles.
- [ ] Se distingue entre módulos licenciados y módulos provisionados.
- [ ] Se documentó el consumo de CPU y memoria como baseline pre-actualización.

---

### Paso 4: Identificar y analizar boot locations

**Objetivo:** Determinar los volúmenes de disco disponibles (HD1.1, HD1.2, HD1.3), identificar el volumen activo y seleccionar el volumen inactivo donde se instalará la nueva imagen.

**Instrucciones:**

1. En TMUI, navegue a **System > Software Management > Boot Locations**. Observe los volúmenes listados y su estado.

2. Desde TMSH, ejecute:

   ```bash
   tmsh show sys software status
   ```

3. Analice la salida e identifique:

   | Volumen | Versión instalada | Build | Estado (active/complete/empty) |
   |---|---|---|---|
   | HD1.1 | | | |
   | HD1.2 | | | |
   | HD1.3 | | | |

4. Confirme el volumen activo actual:

   ```bash
   tmsh show sys software status | grep -i "yes"
   ```

   El volumen con `Active` marcado como `yes` es el que está ejecutando TMOS actualmente. En una instalación estándar del laboratorio, debería ser **HD1.1** con la versión 17.1.1.3.

5. Determine el volumen objetivo para la instalación. La estrategia recomendada es:
   - Instalar la nueva versión en **HD1.2** (volumen inactivo).
   - Mantener **HD1.1** con la versión actual como punto de rollback.
   - **HD1.3** queda disponible como volumen de reserva.

6. Verifique el espacio disponible en disco:

   ```bash
   tmsh run util bash -c "df -h /shared/images"
   ```

   > **Requisito:** El directorio `/shared/images/` debe tener al menos 4 GB disponibles para almacenar la imagen ISO. La instalación en el volumen requiere espacio adicional en la partición de destino.

**Resultado esperado:**

```
Sys::Software Status
Volume  Product   Version    Build  Active  Status
HD1.1   BIG-IP   17.1.1     3.0    yes     complete
HD1.2   none      none      none    no      complete
HD1.3   none      none      none    no      complete
```

> **Nota:** La salida exacta puede variar. En algunos entornos, HD1.2 o HD1.3 pueden contener versiones previas de fábrica.

**Verificación:**

- [ ] Se identificó correctamente el volumen activo (HD1.1, versión 17.1.1.3).
- [ ] Se seleccionó HD1.2 como volumen objetivo para la instalación.
- [ ] Se confirmó espacio suficiente en `/shared/images/`.

---

### Paso 5: Transferir la imagen de software y verificar su integridad

**Objetivo:** Cargar la imagen ISO de BIG-IP 17.1.2.1 al directorio `/shared/images/` del BIG-IP mediante SCP y verificar su integridad con checksum SHA256.

**Instrucciones:**

1. Obtenga el checksum SHA256 oficial de la imagen `BIG-IP-17.1.2.1-0.0.4.iso` desde la página de descargas de [my.f5.com](https://my.f5.com/manage/s/downloads). Registre el valor:

   ```
   SHA256 oficial: ____________________________________________
   ```

   > **Fuente del checksum:** El hash SHA256 se encuentra en la página de descarga junto al enlace de la imagen ISO. Si el instructor proporcionó la imagen localmente, también debe proporcionar el checksum de referencia.

2. Desde el cliente de administración (192.0.2.10), transfiera la imagen al BIG-IP mediante SCP:

   ```bash
   scp /ruta/local/BIG-IP-17.1.2.1-0.0.4.iso admin@192.0.2.245:/shared/images/
   ```

   > **Tiempo estimado:** La transferencia puede tomar varios minutos dependiendo del rendimiento de la red de gestión. La imagen ISO típica de BIG-IP VE ocupa entre 2.5 y 4.5 GB.

3. Verifique que la imagen aparece en el directorio correcto del BIG-IP:

   ```bash
   tmsh run util bash -c "ls -lh /shared/images/BIG-IP-17.1.2.1-0.0.4.iso"
   ```

4. Calcule el checksum SHA256 de la imagen transferida:

   ```bash
   tmsh run util bash -c "sha256sum /shared/images/BIG-IP-17.1.2.1-0.0.4.iso"
   ```

5. Compare el hash calculado con el hash oficial obtenido en el paso 1. **Los valores deben coincidir exactamente.**

   > **Advertencia crítica:** Si los checksums no coinciden, la imagen está corrupta o fue alterada durante la transferencia. **No proceda con la instalación.** Elimine la imagen y repita la transferencia:
   > ```bash
   > tmsh run util bash -c "rm -f /shared/images/BIG-IP-17.1.2.1-0.0.4.iso"
   > ```

6. Verifique también que TMUI reconoce la imagen. Navegue a **System > Software Management > Image List**. La imagen `BIG-IP-17.1.2.1-0.0.4.iso` debe aparecer en la lista.

7. Desde TMSH, confirme la disponibilidad:

   ```bash
   tmsh list sys software image
   ```

**Resultado esperado:**

```
sys software image BIG-IP-17.1.2.1-0.0.4.iso {
    build 0.0.4
    build-date 2024-XX-XXTXX:XX:XXZ
    file-size 3XXX MB
    product BIG-IP
    version 17.1.2.1
}
```

El checksum SHA256 calculado coincide con el valor oficial.

**Verificación:**

- [ ] La imagen ISO está presente en `/shared/images/`.
- [ ] El checksum SHA256 calculado coincide con el valor oficial de F5.
- [ ] La imagen aparece en TMUI (Image List) y en la salida de `tmsh list sys software image`.

---

### Paso 6: Instalar la imagen en el volumen inactivo y monitorear el progreso

**Objetivo:** Ejecutar la instalación de TMOS 17.1.2.1 en el volumen HD1.2 y monitorear el proceso hasta su completación exitosa.

**Instrucciones:**

1. **Antes de instalar**, genere un respaldo UCS del estado actual como medida de precaución:

   ```bash
   tmsh save sys ucs /var/local/ucs/pre-upgrade-17.1.1.3_$(date +%Y%m%d).ucs
   ```

   Verifique que el archivo se creó correctamente:

   ```bash
   tmsh run util bash -c "ls -lh /var/local/ucs/pre-upgrade-*.ucs"
   ```

2. Inicie la instalación de la imagen en el volumen HD1.2:

   ```bash
   tmsh install sys software image BIG-IP-17.1.2.1-0.0.4.iso volume HD1.2
   ```

   > **Nota importante:** Este comando inicia la instalación en segundo plano. El volumen activo (HD1.1) **no se ve afectado** y el sistema continúa operando normalmente con la versión 17.1.1.3 durante toda la instalación.

3. Monitoree el progreso de la instalación ejecutando repetidamente:

   ```bash
   tmsh show sys software status
   ```

   Los estados progresarán aproximadamente así:

   | Fase | Estado en HD1.2 |
   |---|---|
   | Inicio | `installing XX.XXX pct` |
   | Progreso | `installing XX.XXX pct` (porcentaje creciente) |
   | Finalización | `complete` |

4. Puede automatizar el monitoreo con un bucle simple (ejecutar desde bash del BIG-IP):

   ```bash
   tmsh run util bash -c "while true; do tmsh show sys software status | grep HD1.2; sleep 15; done"
   ```

   Presione `Ctrl+C` cuando el estado muestre `complete`.

   > **Tiempo estimado de instalación:** Entre 5 y 15 minutos dependiendo del rendimiento de I/O del entorno virtualizado.

5. Una vez completada la instalación, verifique el estado final:

   ```bash
   tmsh show sys software status
   ```

6. **No active el volumen HD1.2 ni reinicie el BIG-IP.** En este laboratorio, la instalación se realiza únicamente para demostrar el procedimiento. La activación del nuevo volumen se reserva para una ventana de mantenimiento planificada según el procedimiento documentado en el Paso 7.

7. Verifique desde TMUI navegando a **System > Software Management > Boot Locations** que HD1.2 ahora muestra la versión 17.1.2.1.

**Resultado esperado:**

```
Sys::Software Status
Volume  Product   Version    Build  Active  Status
HD1.1   BIG-IP   17.1.1     3.0    yes     complete
HD1.2   BIG-IP   17.1.2.1   0.0.4  no      complete
HD1.3   none      none      none    no      complete
```

**Verificación:**

- [ ] El respaldo UCS pre-actualización se generó exitosamente.
- [ ] La instalación en HD1.2 completó sin errores (estado `complete`).
- [ ] HD1.1 permanece como volumen activo con 17.1.1.3.
- [ ] HD1.2 muestra la versión 17.1.2.1 instalada correctamente.
- [ ] El sistema BIG-IP continúa operando normalmente en la versión actual.

---

### Paso 7: Documentar el procedimiento de actualización controlada

**Objetivo:** Completar una plantilla estructurada de procedimiento de actualización que sirva como referencia operativa, incluyendo checklist pre-actualización, pasos de activación, validación post-actualización y procedimiento de rollback.

**Instrucciones:**

1. Copie la siguiente plantilla en su documento de trabajo y complete cada sección con los datos reales de su entorno de laboratorio:

#### Procedimiento de Actualización Controlada — BIG-IP VE

#### Sección A: Información del sistema

| Campo | Valor |
|---|---|
| Hostname | |
| Management IP | 192.0.2.245 |
| Versión actual (HD1.1) | 17.1.1.3 |
| Versión objetivo (HD1.2) | 17.1.2.1 |
| Registration Key | |
| Service Check Date | |
| ¿SCD permite la versión objetivo? | Sí / No |
| Módulos provisionados | |
| Fecha del procedimiento | |
| Responsable | |

#### Sección B: Checklist pre-actualización

Complete cada ítem con ✅ (completado) o ❌ (pendiente):

| # | Verificación | Estado | Comando/Método de verificación |
|---|---|---|---|
| 1 | Service Check Date permite la versión objetivo | | `tmsh show sys license \| grep "Service Check"` |
| 2 | Respaldo UCS generado y almacenado fuera del BIG-IP | | `tmsh save sys ucs` + SCP a ubicación externa |
| 3 | Imagen ISO verificada con checksum SHA256 | | `sha256sum /shared/images/BIG-IP-*.iso` |
| 4 | Imagen instalada en volumen inactivo (HD1.2) | | `tmsh show sys software status` |
| 5 | Estado de HA documentado (si aplica) | | `tmsh show cm sync-status` |
| 6 | Ventana de mantenimiento aprobada | | Proceso organizacional |
| 7 | Procedimiento de rollback documentado y probado | | Sección D de este documento |
| 8 | Notificación a equipos afectados enviada | | Proceso organizacional |
| 9 | Estadísticas de Virtual Servers documentadas como baseline | | `tmsh show ltm virtual` |
| 10 | Estado de monitores y pools documentado | | `tmsh show ltm pool members` |

#### Sección C: Procedimiento de activación

Documente los pasos exactos para activar el nuevo volumen:

```
## Paso C1: Verificar estado final pre-activación
tmsh show sys software status
tmsh show sys failover

## Paso C2: Activar el volumen con la nueva versión
## ADVERTENCIA: Este comando reinicia el sistema BIG-IP
tmsh reboot volume HD1.2

## Paso C3: Esperar el reinicio completo (3-8 minutos)
## Monitorear mediante ping a Management IP o consola del hypervisor

## Paso C4: Validación post-reinicio
tmsh show sys version
tmsh show sys license
tmsh show sys provision
tmsh show sys software status
tmsh show ltm virtual
tmsh show net self
```

> **Nota sobre entornos HA:** En una topología Active/Standby con dos BIG-IP (no aplicable al laboratorio actual con un solo BIG-IP VE), el procedimiento correcto es:
> 1. Actualizar primero el dispositivo **Standby**.
> 2. Validar la versión y configuración en el Standby.
> 3. Realizar failover controlado (`tmsh run sys failover standby`) para que el dispositivo actualizado tome el rol Active.
> 4. Actualizar el segundo dispositivo (ahora Standby).
> 5. Validar ConfigSync entre ambos dispositivos.

#### Sección D: Procedimiento de rollback

```
## Paso D1: Si la validación post-actualización falla, 
## reactivar el volumen anterior
tmsh reboot volume HD1.1

## Paso D2: Después del reinicio, verificar
tmsh show sys version
## Resultado esperado: versión 17.1.1.3

## Paso D3: Si la configuración presenta inconsistencias,
## restaurar desde el UCS pre-actualización
tmsh load sys ucs /var/local/ucs/pre-upgrade-17.1.1.3_YYYYMMDD.ucs

## Paso D4: Validar operación completa
tmsh show ltm virtual
tmsh show ltm pool members
tmsh show net self
```

#### Sección E: Criterios de validación post-actualización

| # | Criterio | Comando de verificación | Resultado esperado |
|---|---|---|---|
| 1 | Versión correcta | `tmsh show sys version` | 17.1.2.1 Build 0.0.4 |
| 2 | Licencia activa | `tmsh show sys license` | Registration Key presente, módulos activos |
| 3 | Módulos provisionados | `tmsh show sys provision` | Mismos módulos que pre-actualización |
| 4 | Self IPs operativas | `tmsh show net self` | Todas las Self IPs con estado UP |
| 5 | Virtual Servers disponibles | `tmsh show ltm virtual` | Estado `available` (verde) |
| 6 | Pool members respondiendo | `tmsh show ltm pool members` | Monitores marcando `up` |
| 7 | Conectividad de gestión | Ping/SSH a 192.0.2.245 | Respuesta exitosa |
| 8 | Logs sin errores críticos | `tmsh run util bash -c "tail -50 /var/log/ltm"` | Sin mensajes CRITICAL o EMERGENCY |

2. Exporte el respaldo UCS a una ubicación externa al BIG-IP como parte del procedimiento:

   ```bash
   # Desde el cliente de administración
   scp admin@192.0.2.245:/var/local/ucs/pre-upgrade-17.1.1.3_*.ucs /ruta/local/backups/
   ```

3. Verifique que el archivo UCS es accesible y tiene un tamaño razonable (típicamente entre 1 MB y 50 MB dependiendo de la complejidad de la configuración):

   ```bash
   ls -lh /ruta/local/backups/pre-upgrade-*.ucs
   ```

**Resultado esperado:**

El estudiante tiene un documento de procedimiento completo con todas las secciones rellenadas con datos reales del entorno de laboratorio, incluyendo valores específicos de Registration Key, SCD, versiones, y comandos ejecutables.

**Verificación:**

- [ ] Todas las secciones del procedimiento (A–E) están completadas con datos reales.
- [ ] El respaldo UCS fue exportado exitosamente fuera del BIG-IP.
- [ ] El procedimiento de rollback está documentado con comandos específicos.
- [ ] Las consideraciones de HA están documentadas (aunque no apliquen al entorno actual de un solo BIG-IP).

## Validación y Pruebas

Ejecute las siguientes validaciones finales para confirmar que todos los objetivos del laboratorio se cumplieron:

### Validación 1: Estado de licencia documentado

```bash
tmsh show sys license | grep -E "(Registration Key|Service Check Date|Licensed Date)"
```

**Criterio de éxito:** Los tres campos están presentes en la salida y documentados en el reporte del estudiante.

### Validación 2: Provisionamiento documentado como baseline

```bash
tmsh list sys provision | grep -E "(ltm|gtm|asm|apm|afm)" 
```

**Criterio de éxito:** El nivel de provisionamiento de cada módulo coincide con lo documentado en la tabla del Paso 3.

### Validación 3: Imagen instalada en volumen inactivo

```bash
tmsh show sys software status
```

**Criterio de éxito:** HD1.1 muestra `active: yes` con versión 17.1.1.3. HD1.2 muestra `active: no` con versión 17.1.2.1 y estado `complete`.

### Validación 4: Respaldo UCS existente y exportado

```bash
tmsh run util bash -c "ls -lh /var/local/ucs/pre-upgrade-*.ucs"
```

**Criterio de éxito:** Existe al menos un archivo UCS con fecha del día actual y tamaño mayor a 0 bytes. El archivo también existe en la estación de trabajo del administrador.

### Validación 5: Integridad del sistema actual no afectada

```bash
tmsh show sys version
tmsh show net self
tmsh show sys global-settings | grep hostname
```

**Criterio de éxito:** La versión activa sigue siendo 17.1.1.3, las Self IPs están operativas y el hostname es el configurado en Lab 01-00-01. La instalación en HD1.2 no alteró el funcionamiento del volumen activo.

### Validación 6: Caso adverso — Verificación de checksum con valor incorrecto

Este caso prueba que el estudiante comprende la importancia de la verificación de integridad y no confía ciegamente en una imagen transferida.

```bash
## Simule un checksum incorrecto comparando con un hash falso
echo "aabbccdd00112233aabbccdd00112233aabbccdd00112233aabbccdd00112233  /shared/images/BIG-IP-17.1.2.1-0.0.4.iso" > /tmp/fake_checksum.txt
tmsh run util bash -c "sha256sum -c /tmp/fake_checksum.txt"
```

**Criterio de éxito:** El comando devuelve `FAILED`. El estudiante debe explicar por qué un checksum que no coincide invalida la imagen y por qué **nunca** se debe proceder con una instalación si la verificación de integridad falla, independientemente de que la imagen "parezca" correcta por su nombre o tamaño. Este caso refuerza que la confianza en la integridad debe basarse en evidencia criptográfica verificable, no en suposiciones.

## Solución de Problemas

### Problema 1: La instalación en HD1.2 falla con error de espacio insuficiente

**Síntomas:** Al ejecutar `tmsh install sys software image BIG-IP-17.1.2.1-0.0.4.iso volume HD1.2`, el estado en `show sys software status` muestra `failed` o `install failed` en lugar de progresar. En `/var/log/ltm` aparecen mensajes relacionados con espacio en disco.

**Causa:** El volumen HD1.2 o el sistema de archivos `/shared/` no tiene espacio suficiente. Esto puede ocurrir si existen imágenes ISO antiguas en `/shared/images/` o si volúmenes previos contienen instalaciones que consumen espacio.

**Solución:**

```bash
## Verificar espacio disponible
tmsh run util bash -c "df -h /shared"
tmsh run util bash -c "df -h"

## Listar imágenes almacenadas y eliminar las que no se necesitan
tmsh list sys software image
tmsh delete sys software image <nombre-imagen-antigua.iso>

## Si un volumen inactivo tiene una instalación vieja que no se necesita,
## se puede eliminar el contenido del volumen
tmsh delete sys software volume HD1.2

## Reintentar la instalación
tmsh install sys software image BIG-IP-17.1.2.1-0.0.4.iso volume HD1.2
```

Verifique que `/shared/images/` tiene al menos 4 GB libres antes de reintentar.

### Problema 2: El SCP al BIG-IP falla con "Permission denied" o "Connection refused"

**Síntomas:** Al ejecutar `scp /ruta/local/BIG-IP-17.1.2.1-0.0.4.iso admin@192.0.2.245:/shared/images/`, la conexión es rechazada o se deniega el permiso. El acceso SSH con el mismo usuario funciona correctamente para comandos TMSH.

**Causa:** Las restricciones de acceso SSH configuradas en el Lab 01-00-01 (ACL de SSHD que permite únicamente 192.0.2.10) pueden estar mal configuradas, o el usuario `admin` no tiene permisos de escritura en `/shared/images/` a través de SCP. En BIG-IP VE, el acceso SCP requiere que el usuario tenga shell de bash habilitado o que se utilice el usuario `root`.

**Solución:**

```bash
## Opción A: Verificar la ACL de SSHD permite la IP del cliente
tmsh list sys sshd allow
## Debe incluir 192.0.2.10 o la subred 192.0.2.0/24

## Opción B: Usar el usuario root para SCP (si está habilitado)
scp /ruta/local/BIG-IP-17.1.2.1-0.0.4.iso root@192.0.2.245:/shared/images/

## Opción C: Transferir primero al home del usuario admin y luego mover
scp /ruta/local/BIG-IP-17.1.2.1-0.0.4.iso admin@192.0.2.245:/var/tmp/
ssh admin@192.0.2.245
tmsh run util bash -c "mv /var/tmp/BIG-IP-17.1.2.1-0.0.4.iso /shared/images/"

## Opción D: Usar TMUI para cargar la imagen
## System > Software Management > Image List > Import
## (Solo viable para archivos de menor tamaño debido a timeouts del navegador)
```

Después de la transferencia exitosa, verifique con `ls -lh /shared/images/` y continúe con la verificación de checksum.

## Limpieza

Este laboratorio **no requiere eliminar** la imagen instalada en HD1.2 ni el respaldo UCS, ya que ambos son artefactos útiles para laboratorios posteriores. El estado final deseado es:

- **HD1.1:** TMOS 17.1.1.3 — volumen **activo** (el sistema continúa operando en esta versión).
- **HD1.2:** TMOS 17.1.2.1 — volumen **inactivo** con la nueva versión instalada y lista para activación futura.
- **UCS:** Respaldo pre-actualización almacenado en `/var/local/ucs/` y en la estación de trabajo.

Si por alguna razón necesita liberar espacio o revertir los cambios de este laboratorio:

```bash
## Eliminar la instalación del volumen HD1.2 (solo si es necesario)
tmsh delete sys software volume HD1.2

## Eliminar la imagen ISO del directorio de imágenes (solo si es necesario)
tmsh run util bash -c "rm -f /shared/images/BIG-IP-17.1.2.1-0.0.4.iso"

## Verificar el estado resultante
tmsh show sys software status
tmsh list sys software image
```

> **No elimine el respaldo UCS.** Este archivo será referenciado en laboratorios posteriores del curso.

## Resumen

En este laboratorio se aplicaron los procedimientos fundamentales de gestión del ciclo de vida de BIG-IP:

| Fase completada | Resultado |
|---|---|
| Validación de licencia | Registration Key, SCD y módulos licenciados documentados; elegibilidad de actualización determinada |
| Revisión de provisionamiento | Baseline de módulos provisionados y consumo de recursos documentado |
| Gestión de boot locations | Volúmenes identificados; imagen 17.1.2.1 instalada en HD1.2 con integridad verificada |
| Procedimiento de actualización | Documento completo con checklist, pasos de activación, criterios de validación y rollback |

**Relación con objetivos curriculares:**

- **F5CAB1.03:** Se validó el estado de licenciamiento y se comprendió el impacto del SCD en la elegibilidad de actualización.
- **F5CAB1.04:** Se identificaron boot locations, se gestionaron imágenes de software y se ejecutó una instalación controlada.
- **F5CAB1.05:** Se documentó un procedimiento de actualización completo con estrategia de rollback.
- **F5CAB4.09:** Las consideraciones de HA documentadas en el procedimiento (actualización secuencial standby-primero) preparan al estudiante para el análisis de alta disponibilidad del Módulo 4.

**Dependencias hacia adelante:** El respaldo UCS generado y el procedimiento de actualización documentado serán referenciados en laboratorios posteriores de administración del plano de control (Módulo 1) y en el análisis de alta disponibilidad (Módulo 4).

### Recursos adicionales

| Recurso | URL |
|---|---|
| K13845: Gestión de imágenes de software en BIG-IP | [https://my.f5.com/manage/s/article/K13845](https://my.f5.com/manage/s/article/K13845) |
| K34745165: Procedimiento de actualización de BIG-IP | [https://my.f5.com/manage/s/article/K34745165](https://my.f5.com/manage/s/article/K34745165) |
| K7727: Gestión de licencias BIG-IP | [https://my.f5.com/manage/s/article/K7727](https://my.f5.com/manage/s/article/K7727) |
| K9476: Configuración inicial de BIG-IP | [https://my.f5.com/manage/s/article/K9476](https://my.f5.com/manage/s/article/K9476) |
| Referencia de comandos TMSH | [https://clouddocs.f5.com/cli/tmsh-reference/latest/](https://clouddocs.f5.com/cli/tmsh-reference/latest/) |
| Release Notes TMOS 17.1.2.1 | [https://my.f5.com/manage/s/article/K000137353](https://my.f5.com/manage/s/article/K000137353) `[ENLACE OFICIAL — verificar disponibilidad]` |
