# SOC-LAB // TICKET #PGN-2026-0417 - El caso de Pinguinito

**Resumen:** Durante el análisis forense del laboratorio "El caso de Pinguinito", se evalúa una captura de tráfico (`incidente_pinguino.pcap`) y un archivo de inteligencia de amenazas (`threat_intel_feed.json`). La investigación revela un compromiso web mediante escaneos automatizados, un bypass de restricciones de subida de archivos mediante doble extensión para desplegar una webshell, y actividades posteriores de reconocimiento interno y modificación de credenciales de usuario.

## Información de la Máquina

| **Campo**       | **Valor**   |
| --------------- | ----------- |
| **Plataforma**  | SOC-LAB     |
| **Dificultad**  | Facil / CTF |
| **SO**          | Linux       |
| **IP Objetivo** | 10.10.2.15  |

## Reconocimiento e Inicio del Caso

### Escaneo inicial

Iniciamos las tareas de reconocimiento ejecutando un escaneo con `nmap` para identificar los puertos y servicios activos:

```
nmap -sS -p- -vvv --min-rate 5000 172.17.0.2
```

![](assets/Pasted%20image%2020261004232340.png)

Posteriormente, se inspecciona el servicio web expuesto en el puerto 80, descargando el archivo de captura de tráfico (`.pcap`) y dando inicio formal al caso interactivo desde la plataforma.

![](assets/Pasted%20image%2020261004232957.png)

## Análisis de Tráfico e Investigación (Forense Web)

### 1. Identificación del origen

Empezamos la investigación filtrando en Wireshark por `http`. Entre varios resultados, se observa una solicitud `POST` al recurso `/reviews/upload.php`. Analizándola más a fondo, se ve claramente que es un intento de subir un archivo `.php` simulando un formulario que intenta generar una webshell, utilizando una herramienta automatizada como `ReconBot`. Al final, se observa cómo la petición es rechazada con un código de error `403` al tratarse de un archivo no permitido.

- **IP Source:** `198.51.100.23`
    ![](assets/Pasted%20image%2020261004234717.png)
  
    ![](assets/Pasted%20image%2020261004235314.png)

Continuando la investigación y revisando las tramas subsiguientes, se constata que la misma IP logró subir una imagen con el siguiente nombre y extensión (`image.jpg.php`), dentro de la ruta `/reviews/uploads/image.jpg.php`, obteniendo de este modo una webshell.

- ![](assets/Pasted%20image%2020261005201951.png)

### 2. Atribución geográfica

Según el feed de threat intelligence adjunto (`threat_intel_feed.json`), analizando específicamente la IP atacante identificada en la sección anterior:

Utilizando el archivo mencionado en la realización de la consulta (mediante herramientas de texto como `nano`), se comprueba que los metadatos de la IP determinan su procedencia.

- **País:** Vietnam
  
 ![](assets/Pasted%20image%2020261005205709.png)

### 3. Huella del atacante

Identificación del agente de usuario utilizado en las peticiones HTTP automatizadas del atacante.

- **User-Agent:** `Mozilla/5.0 (compatible; ReconBot/1.0; +http://lab.invalid/reconbot)`
    ![](assets/Pasted%20image%2020261005205850.png)

### 4. Endpoint de subida

La ruta o script específico del servidor hacia donde se envían las peticiones `POST` que gestionan la carga de archivos.

- **Endpoint:** `/reviews/upload.php`
    ![](assets/Pasted%20image%2020261005210120.png)

### 5. Directorio de subidas

Durante la fase de reconocimiento de rutas, el atacante prueba varias ubicaciones hasta dar con el directorio real habilitado en el sitio.

- **Directorio:** `/reviews/uploads/`
    ![](assets/Pasted%20image%2020261005210539.png)

### 6. El archivo malicioso

La primera tentativa de subida es bloqueada por el filtro del servidor, pero la segunda ejecución tiene éxito gracias a una técnica de bypass.

- **Archivo cargado:** `image.jpg.php`

### 7. Conexión saliente

Tras acceder al recurso malicioso alojado, el servidor ejecuta las instrucciones y establece una conexión saliente (reverse shell) hacia la infraestructura del atacante.

- **Puerto de conexión:** `8080`
    ![](assets/Pasted%20image%2020261005210830.png)

### 8. Reconocimiento interno

Dentro de la sesión interactiva de shell inversa obtenida, el atacante ejecuta comandos para inspeccionar ficheros críticos del sistema.

- **Fichero consultado:** `/etc/passwd`
    ![](assets/Pasted%20image%2020261005212138.png)

### 9. Usuario afectado

Como parte de las acciones finales registradas en el análisis, el atacante ejecuta un script interno para manipular las credenciales o restablecer la contraseña de una cuenta en particular.

- **Usuario afectado:** `pinguinito`
    ![](assets/Pasted%20image%2020261005212216.png)

## Más Allá del Incidente

**Análisis Post-Explotación**

- **Lo que funcionó:** El uso de herramientas de automatización de reconocimiento (`ReconBot`) y el bypass de restricciones de extensión mediante doble nomenclatura (`image.jpg.php`) para forzar la ejecución de código en un entorno web vulnerable.
    
- **Lo que falló:** Los intentos iniciales de subir scripts puros en `.php`, que fueron detectados y bloqueados por las políticas iniciales de validación de extensiones del servidor.
    
- **Herramientas nuevas:** Correlación avanzada de trazas de red en Wireshark con ficheros estructurados de inteligencia de amenazas (`.json`).
    
- **Para la próxima:** Reforzar la sanitización de nombres de archivo en el backend y evitar restricciones basadas puramente en la extensión declarada por el cliente.
    

## Mitigación

Contramedidas recomendadas para blindar la aplicación frente a este tipo de incidentes:

### Validación robusta de subida de archivos

**Vulnerabilidad:** El servidor permitía el almacenamiento de archivos aplicando filtros débiles sobre las extensiones, permitiendo un bypass con doble extensión (`image.jpg.php`).

**Mitigaciones:**

- Implementar listas blancas estricta de extensiones permitidas y validar el tipo MIME real del archivo en el servidor.
    
- Deshabilitar la ejecución de scripts en las carpetas de subida de contenido de usuarios mediante directivas de configuración en el servidor web.
    
- Renovar los nombres de los archivos subidos utilizando hashes o identificadores únicos aleatorios.
    

## Referencias

- [OWASP - Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload "null")
    
- [OWASP - Threat Intelligence Guide](https://owasp.org/ "null")
