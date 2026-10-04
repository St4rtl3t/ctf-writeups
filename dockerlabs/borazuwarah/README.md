# DockerLabs - Borazuwarah

**Resumen:** La máquina expone un servidor web y un servicio SSH. Durante la enumeración web se identifican imágenes que contienen metadatos, donde se obtiene información que permite descubrir el usuario `borazuwarah` y la contraseña `empty`. Posteriormente, mediante `hydra`, se obtiene una contraseña válida para SSH y se consigue acceso al sistema. Finalmente, se comprueban los privilegios del usuario y se identifica que dispone de permisos `ALL` mediante `sudo`.

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

Se verificó la conectividad con el contenedor y, posteriormente, se ejecutó un escaneo completo de puertos con `nmap`, incluyendo detección de versiones de los servicios.

```bash
nmap -sS -sV -p- 172.17.0.2 -Pn -oN boraresultadonmap.txt
```

![Escaneo inicial](assets/Pasted%20image%2020260921002841.png)

El resultado mostró dos puertos abiertos:

- **22/tcp:** SSH
    
- **80/tcp:** HTTP
    

Debido a que el puerto 80 estaba abierto, se procedió a inspeccionar el servicio web mediante Firefox.

### Enumeración web

Al ingresar al servicio web se encontró la siguiente página:

![Página web](assets/Pasted%20image%2020260921003047.png)

La información mostrada no aportaba demasiadas pistas, por lo que, siguiendo el proceso habitual de enumeración, se realizó un fuzzing de directorios para comprobar si existían recursos adicionales que pudieran aportar información pero no hubo mas pistas que la ya vista.

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/big.txt -x php,txt,html --exclude-length 10701
```

---

## Análisis

### Metadatos en imágenes

Después de probar con varios usuarios mediante hydra, llegue a la conclusión de que las fotografías disponibles en la máquina podían contener metadatos útiles.

Para comprobar esta posibilidad, se utilizó la herramienta ExifTool la cual realiza extracción de metadatos. 

El resultado fue el siguiente:
![Metadatos de las imágenes](assets/Pasted%20image%2020260921005955.png)

A partir de la información obtenida se identificó el usuario:

- **Usuario:** `borazuwarah`
- **Contraseña:** `empty`
    

Con el usuario identificado, se procedió a comprobar el acceso mediante SSH.

---

## Explotación

### Acceso mediante SSH

Teniendo identificado el usuario `borazuwarah`, se utilizó `hydra` para realizar un ataque de diccionario contra el servicio SSH:

```bash
hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt.gz -t 4 ssh://172.17.0.2
```

El resultado fue:

![Credenciales obtenidas mediante Hydra](assets/Pasted%20image%2020260921010208.png)

Con las credenciales obtenidas se procedió a iniciar sesión mediante SSH.

### Enumeración de privilegios

Una vez dentro del sistema, se comenzó con la enumeración básica del usuario, verificando su identidad y los grupos a los que pertenece.

![Usuario y grupos](assets/Pasted%20image%2020260921010342.png)

Posteriormente, se comprobó que el usuario disponía de permisos `ALL` sobre `ALL` mediante `sudo`.


A partir de estos permisos, se optó por utilizar el siguiente comando para continuar con la escalada de privilegios: `sudo su`

![Permisos sudo](assets/Pasted%20image%2020260921010526.png)

---

## Más Allá de Root

**Análisis Post-Explotación**

- **Lo que funcionó:** La enumeración de las imágenes permitió encontrar información útil en sus metadatos. Posteriormente, `hydra` permitió obtener acceso al servicio SSH y la enumeración de `sudo` reveló permisos `ALL`.
    
- **Lo que falló:** Los primeros intentos de enumeración de usuarios no permitieron obtener información suficiente hasta analizar los metadatos de las imágenes.
    
- **Herramientas nuevas:** Uso de herramientas para extraer metadatos de imágenes y utilización de `hydra` contra SSH.
    
- **Para la próxima:** Incluir desde el principio el análisis de metadatos como parte de la enumeración cuando una máquina exponga imágenes u otros archivos multimedia.
---

## Mitigación

### Exposición de información mediante metadatos

**Vulnerabilidad:** Las imágenes disponibles en el servicio web contenían metadatos que permitieron obtener información útil para continuar el ataque.

**Mitigaciones:**

- Eliminar metadatos innecesarios antes de publicar imágenes.
    
- Evitar almacenar información sensible en los metadatos de archivos públicos.
    
- Revisar los archivos multimedia antes de incorporarlos a un sitio web.
    
- Limitar la información que pueda utilizarse para identificar usuarios, equipos o ubicaciones.
    

### Ataques de diccionario contra SSH

**Vulnerabilidad:** El servicio SSH permitió realizar intentos automatizados de autenticación contra un usuario válido.

**Mitigaciones:**

- Utilizar autenticación mediante claves SSH en lugar de contraseñas cuando sea posible.
    
- Implementar mecanismos de bloqueo o limitación de intentos.
    
- Utilizar herramientas como Fail2ban para detectar ataques de fue