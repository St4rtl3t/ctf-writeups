# DockerLabs - Trust

**Resumen:** Máquina Linux que expone los servicios SSH y HTTP. Mediante la enumeración web se localiza el archivo `secret.php`, que proporciona una pista para identificar al usuario `mario`. A partir de esta información se realiza un ataque de diccionario contra SSH con `hydra`, obteniendo las credenciales de acceso. Una vez dentro del sistema, se identifica que el usuario puede ejecutar `vim` mediante `sudo`, lo que permite obtener una shell con privilegios de `root`.

## Información de la Máquina

| Campo          | Valor      |
| :------------- | :--------- |
| **Plataforma** | DockerLabs |
| **Dificultad** | Facil      |
| **SO**         | Linux      |
| **IP**         | 172.17.0.2 |

---

## Reconocimiento

### Escaneo inicial

Se verificó la conectividad con el contenedor y se realizó un escaneo completo de puertos para identificar los servicios expuestos.

```bash
nmap -sS -sV -p- 172.17.0.2 -Pn -oN trustresultadonmap.txt
```

![Resultado del escaneo inicial](assets/Pasted%20image%2020260920233827.png)

El escaneo permitió identificar los siguientes servicios:

- **Puerto 22:** SSH.
    
- **Puerto 80:** HTTP.
    

Debido a la presencia del servicio HTTP, se procedió a inspeccionar la aplicación web mediante Firefox.

### Enumeración web

Al acceder al puerto 80 se observó una página por defecto de Apache, sin información relevante visible.

![Página web por defecto](assets/Pasted%20image%2020260920234454.png)

Debido a que la página principal no proporcionaba información útil, se realizó un fuzzing de directorios utilizando `gobuster`:

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html --exclude-length 10701
```

**Resultado:**

```text
secret.php           (Status: 200) [Size: 927]
```

Se identificó el archivo `secret.php`, por lo que se procedió a acceder directamente a él.

![Contenido de secret.php](assets/Pasted%20image%2020260920234650.png)

---

## Análisis

### Observaciones

El archivo `secret.php` contenía una pista que permitía identificar el posible nombre de usuario `mario`.

A partir de este hallazgo se planteó la posibilidad de que dicho usuario existiera también en el sistema y pudiera utilizarse para acceder mediante SSH.

Como el puerto 22 se encontraba expuesto, se decidió comprobar esta hipótesis mediante un ataque de diccionario contra el servicio SSH utilizando `hydra` y el diccionario `rockyou.txt.gz`.

---

## Explotación

### Vulnerabilidad / Vector

**Tipo:** Credenciales débiles mediante ataque de diccionario + configuración insegura de `sudo`  
**Componente afectado:** Servicio SSH y configuración `sudo` del usuario `mario`

Se utilizó `hydra` para probar las contraseñas del diccionario contra el usuario identificado:

```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt.gz -t 4 ssh://172.17.0.2
```

El ataque encontró credenciales válidas:

```text
[22][ssh] host: 172.17.0.2   login: mario   password: chocolate
```

Las credenciales obtenidas fueron:

|Usuario|Contraseña|
|:--|:--|
|mario|chocolate|

### Acceso Inicial

Con las credenciales obtenidas se realizó el acceso mediante SSH.

Una vez dentro, se verificó la identidad del usuario, los grupos a los que pertenecía y los permisos disponibles mediante `sudo`.

![Información del usuario y permisos](assets/Pasted%20image%2020260920235335.png)

El resultado permitió identificar que `mario` podía ejecutar `vim` mediante `sudo`.

### Escalada de Privilegios

Se ejecutó `vim` con privilegios elevados:

```bash
sudo vim
```

Una vez abierto `vim`, se utilizó su funcionalidad de ejecución de comandos para obtener una shell.

Desde `vim` se accedió a la línea de comandos y se ejecutó:

```text
shell
```

Debido a que `vim` había sido iniciado mediante `sudo`, la shell resultante heredó los privilegios de `root`.

![Shell como root](assets/Pasted%20image%2020260921001848.png)

### Objetivo Alcanzado

Se obtuvo acceso inicial al sistema como `mario` mediante las credenciales encontradas con un ataque de diccionario contra SSH.

Posteriormente, se aprovechó el permiso de `sudo` sobre `vim` para escapar del editor y obtener una shell con privilegios de `root`.

---
## Más Allá de Root

**Análisis Post-Explotación**

- **Lo que funcionó:** La enumeración web permitió localizar `secret.php`, cuya información proporcionó una pista para identificar al usuario `mario`. Posteriormente, el ataque de diccionario contra SSH permitió obtener sus credenciales y acceder al sistema.
    
- **Lo que falló:** La página principal del servidor web únicamente mostraba la página por defecto de Apache, por lo que fue necesario realizar una enumeración de directorios.
    
- **Herramientas nuevas:** Uso de `gobuster` para localizar recursos ocultos y aprovechamiento de `vim` como vector de escalada de privilegios mediante `sudo`.
    
- **Para la próxima:** Revisar siempre los permisos obtenidos mediante `sudo -l` después de conseguir acceso a un sistema, especialmente cuando se encuentren binarios que puedan ejecutar comandos.
    

---

## Mitigación

Contramedidas recomendadas para cada vulnerabilidad identificada durante la auditoría, ordenadas según la fase del ataque en la que fueron explotadas.

### Exposición de información sensible en `secret.php`

**Vulnerabilidad:** El servicio web exponía un archivo accesible públicamente que contenía una pista que permitía identificar un posible usuario del sistema.

**Mitigaciones:**

- No almacenar información sensible o pistas relacionadas con usuarios en archivos accesibles públicamente.
    
- Restringir el acceso a archivos que no deban formar parte del contenido público de la aplicación.
    
- Revisar periódicamente los recursos accesibles desde el servidor web.
    
- Aplicar controles de acceso adecuados sobre información sensible.
    

### Autenticación débil en SSH

**Vulnerabilidad:** El usuario `mario` utilizaba una contraseña (`chocolate`) que podía encontrarse mediante un ataque de diccionario.

**Mitigaciones:**

- Utilizar contraseñas robustas y únicas.
    
- Preferir autenticación mediante claves SSH en lugar de contraseñas.
    
- Implementar `fail2ban` o mecanismos equivalentes para limitar intentos de autenticación.
    
- Restringir el acceso SSH mediante firewall, VPN o listas de IP autorizadas.
    
- Deshabilitar `PasswordAuthentication` cuando la autenticación mediante claves sea viable.
    

### Escalada de privilegios mediante `sudo vim`

**Vulnerabilidad:** El usuario `mario` podía ejecutar `vim` mediante `sudo`