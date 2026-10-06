# DockerLabs - Hedgehog

**Resumen:** Máquina Linux que expone un servicio web en el puerto 80 con una pista que apunta a un posible nombre de usuario (`tails`). A partir de esa pista se realizó un ataque de fuerza bruta contra SSH con `hydra`, optimizando el diccionario `rockyou.txt` mediante `tac` para invertir su orden.

El acceso inicial como `tails` permitió identificar una configuración permisiva de `sudo`, mediante la cual fue posible escalar al usuario `sonic` sin contraseña.

## Información de la Máquina

| Campo          | Valor      |
| :------------- | :--------- |
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil      |
| **SO**         | Linux      |
| **IP**         | 172.17.0.2 |

---

## Reconocimiento

### Escaneo inicial

Se realizó un escaneo inicial para identificar los puertos abiertos y los servicios expuestos en la máquina objetivo.

```bash
nmap -sV -sC 172.17.0.2
```

Se identificó el servicio web expuesto en el puerto 80, por lo que se procedió a inspeccionar su contenido.

### Enumeración web

Al acceder al servicio web se encontró un mensaje que hacía referencia a `tails`.

![Escaneo inicial](assets/Pasted%20image%2020260920220015.png)

La referencia a `tails` podía corresponder a un nombre de usuario válido dentro del sistema.

---

## Análisis

### Observaciones

El contenido del servicio web hacía referencia a `tails`, lo que sugería un posible nombre de usuario del sistema.

Dado que SSH suele estar expuesto en este tipo de laboratorios, se planteó la hipótesis de que `tails` fuera una cuenta válida y que su contraseña pudiera obtenerse mediante un ataque de diccionario.

El razonamiento clave vino del propio significado del comando `tail` en Linux: **muestra las últimas líneas de un archivo**. A partir de esta pista se decidió invertir el orden del diccionario `rockyou.txt`, colocando al principio las entradas que originalmente se encontraban al final.

Esto permitía probar primero las contraseñas que estaban en la parte final del diccionario, reduciendo el tiempo necesario respecto al ataque estándar.

---

## Explotación

### Vulnerabilidad / Vector

**Tipo:** Autenticación débil por contraseña en SSH + filtración de nombres de usuario
**Componente afectado:** Servicio SSH + servicio web en puerto 80

El primer intento con `hydra` utilizando el diccionario estándar arrojaba un tiempo estimado de **3568 horas**, por lo que el ataque no resultaba viable.

Para optimizarlo se invirtió el diccionario mediante `tac`:

```bash
tac /usr/share/wordlists/rockyou.txt > rockyou_inv.txt
```

* **`tac`**: lee las líneas de un archivo en orden inverso.
* **`/usr/share/wordlists/rockyou.txt`**: diccionario utilizado para el ataque.
* **`>`**: redirige la salida hacia un nuevo archivo.

Posteriormente se eliminaron los espacios en blanco residuales:

```bash
sed -i 's/ //g' rockyou_inv.txt
```

* **`sed`**: herramienta de procesamiento de texto.
* **`-i`**: modifica el archivo directamente.
* **`'s/ //g'`**: reemplaza los espacios por nada de forma global.

Finalmente se ejecutó el ataque contra SSH utilizando el diccionario invertido:

```bash
hydra -l tails -P rockyou_inv.txt -t 4 ssh://172.17.0.2
```

**Resultado:**

```text
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
```

Con las credenciales obtenidas se accedió al sistema como `tails`.

Una vez dentro, se verificó la identidad del usuario, sus grupos y sus permisos de `sudo`:

```bash
whoami && id && sudo -l
```

![Credenciales obtenidas](assets/Pasted%20image%2020260920232016.png)

Los resultados mostraron que `tails` podía ejecutar comandos como el usuario `sonic` **sin proporcionar contraseña**.

### Escalada al usuario sonic

Se abusó de la regla de `sudo` para obtener una shell como `sonic`:

```bash
sudo su -u sonic /bin/bash
```

**Resultado:**

![Permisos sudo](assets/Pasted%20image%2020260920232633.png)

```text
Acceso inicial como tails y escalada al usuario sonic mediante una regla permisiva de sudo.
```

### Objetivo Alcanzado

Se obtuvo acceso inicial al sistema como `tails` mediante un ataque de diccionario contra SSH y posteriormente se realizó una escalada al usuario `sonic` aprovechando una regla `sudo` configurada sin contraseña.

---

## Más Allá de Root

**Análisis Post-Explotación**

* **Lo que funcionó:** El razonamiento lateral a partir del nombre `tails` permitió invertir el diccionario y reducir drásticamente el tiempo del ataque.
* **Lo que falló:** El ataque inicial utilizando el diccionario estándar era inviable, con un tiempo estimado de 3568 horas.
* **Herramientas nuevas:** Uso de `tac` y `sed` para transformar un diccionario antes de utilizarlo con `hydra`.
* **Para la próxima:** Antes de ejecutar un ataque de fuerza bruta, calcular el tiempo estimado y considerar variantes del diccionario, como inversión, filtrado o mutaciones.

---

## Mitigación

Contramedidas recomendadas para cada vulnerabilidad identificada durante la auditoría, ordenadas según la fase del ataque en la que fueron explotadas.

### Filtración de nombres de usuario en el servicio web

**Vulnerabilidad:** El sitio público exponía una pista que permitía deducir un nombre de usuario válido del sistema (`tails`).

**Mitigaciones:**

* No publicar nombres de usuario reales ni pistas que permitan deducirlos en servicios expuestos.
* Evitar mensajes de error diferenciados que revelen si un usuario existe o no.
* Aplicar rate limiting y bloqueo por IP en el servicio SSH.

### Autenticación débil por contraseña en SSH

**Vulnerabilidad:** La contraseña de `tails` estaba presente en `rockyou.txt`, lo que permitió realizar un ataque de diccionario exitoso.

**Mitigaciones:**

* Deshabilitar la autenticación por contraseña (`PasswordAuthentication no`) y utilizar claves SSH.
* Implementar `fail2ban` o `sshguard` para bloquear IPs después de varios intentos fallidos.
* Aplicar políticas de contraseñas robustas mediante PAM.
* Limitar el acceso SSH por rango de IP o mediante VPN.

### Regla permisiva en sudoers

**Vulnerabilidad:** El usuario `tails` podía ejecutar comandos como `sonic` sin contraseña, permitiendo una escalada directa de privilegios.

**Mitigaciones:**

* Aplicar el principio de mínimo privilegio.
* Evitar reglas `NOPASSWD` que permitan cambios de usuario o ejecución de intérpretes de comandos.
* Auditar periódicamente `/etc/sudoers` y `/etc/sudoers.d/`.
* Revisar las capacidades de cualquier binario delegado mediante `sudo`.

**Defensa en Profundidad:** La combinación de controles preventivos, detectivos y correctivos es lo que realmente reduce la superficie de ataque.

Los controles preventivos incluyen el principio de mínimo privilegio y el uso de autenticación mediante claves. Los controles detectivos pueden incluir herramientas como `fail2ban` y el monitoreo de logs SSH, mientras que los controles correctivos incluyen la rotación de credenciales comprometidas.

---

## Referencias

* [GTFOBins](https://gtfobins.github.io/)
* [Hydra - Documentación oficial](https://github.com/vanhauser-thc/thc-hydra)
* [OWASP - Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
* [Sudoers Manual](https://www.sudo.ws/docs/man/sudoers.man/)
