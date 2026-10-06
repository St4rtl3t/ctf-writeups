# CTF Writeups

Colección de writeups de máquinas y retos de CTF.

## DockerLabs

* [Amor](./dockerlabs/amor/) — Fuerza bruta SSH + esteganografía + escalada vía `sudo ruby`.
* [Whoiam](./dockerlabs/whoiam/) — WordPress con backup expuesto + RCE vía plugin malicioso + escalada encadenada (`sudo find` → `debugfs` → inyección de comandos).
* [Trust](./dockerlabs/trust/) — Enumeración web + ataque de diccionario contra SSH + escalada vía `sudo vim`.
* [Hedgehog](./dockerlabs/hedgehog/) — Enumeración web + fuerza bruta SSH optimizada con diccionario invertido + escalada mediante `sudo` a `sonic`.
* [Borazuwarah](./dockerlabs/borazuwarah/) — Enumeración web + extracción de metadatos con ExifTool + ataque de diccionario contra SSH + escalada mediante `sudo`.
* [Analyst](./dockerlabs/analyst) — Análisis forense PCAP + Threat Intel + Bypass de subida de archivos (doble extensión `image.jpg.php`) + Webshell y RCE.
