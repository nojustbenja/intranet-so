# Despliegue de una Intranet Corporativa en Ubuntu Server

**Asignatura:** Sistemas Operativos
**Evaluación:** Tercera Solemne
**Tema:** Despliegue de un sitio web como intranet institucional sobre Linux

---

## Descripción del proyecto

Este proyecto documenta el despliegue completo de una **intranet corporativa** sobre
**Ubuntu Server**, ejecutándose dentro de una máquina virtual con IP fija. El objetivo
es levantar un sitio web interno (HTML/CSS) servido por un servidor web, con la
administración de usuarios, permisos, firewall y análisis de logs que corresponden a
un entorno de servidor real.

La intranet simula el portal interno de una institución ficticia y es accesible desde
un equipo cliente dentro de la misma red.

### Arquitectura general

```
+---------------------+                         +------------------------------+
|   Equipo Cliente    |   Red local (LAN)       |   VM - Ubuntu Server         |
|   (Navegador web)   | <---------------------> |   IP fija: 192.168.x.x       |
|                     |     HTTP (puerto 80)    |   Servidor web + Intranet    |
+---------------------+                         +------------------------------+
                                                 |  - Servidor web (systemd)   |
                                                 |  - /var/www/intranet        |
                                                 |  - Usuarios y permisos POSIX|
                                                 |  - Firewall ufw (deny)      |
                                                 |  - Logs access/error        |
                                                 +------------------------------+
```

---

## Objetivos

- Seleccionar e instalar una distribución Linux de tipo servidor.
- Configurar red con IP fija y verificar conectividad cliente-servidor.
- Instalar y configurar un servidor web con arranque automático gestionado por `systemd`.
- Diseñar y publicar un sitio web institucional con al menos 3 páginas.
- Administrar usuarios, grupos y permisos POSIX con el principio de mínimo privilegio.
- Aplicar seguridad de red mediante firewall con política *default deny*.
- Analizar logs, recopilar evidencias y respaldar la configuración.

---

## Distribución de responsabilidades del equipo

| # | Integrante | Área de responsabilidad |
|---|------------|-------------------------|
| 1 | **Benjamín** | Infraestructura, distribución y red |
| 2 | **Vicente** | Servidor web y `systemd` |
| 3 | **Guille** | Contenido de la intranet |
| 4 | **Simón** | Usuarios, grupos y permisos |
| 5 | **Pía** | Firewall y seguridad de red |
| 6 | **Carlita** | Logs, evidencias y limitaciones |

---

## 1. Infraestructura, distribución y red — *Benjamín*

### Elección de la distribución: Ubuntu Server

Se eligió **Ubuntu Server (LTS)** por las siguientes razones:

- **Soporte extendido (LTS):** 5 años de actualizaciones de seguridad.
- **Comunidad y documentación amplia**, ideal para un entorno académico.
- **Sin entorno gráfico:** menor consumo de recursos, orientado a servidor.
- **Gestor de paquetes `apt`** y repositorios estables.

### Instalación y configuración de la VM

1. Crear la VM (VirtualBox / VMware / KVM) con:
   - 2 vCPU, 2 GB RAM, 20 GB de disco.
   - Adaptador de red en modo **Puente (Bridged)** o **Red interna** para que el
     cliente alcance al servidor.
2. Instalar Ubuntu Server desde la ISO oficial.

### Configuración de IP fija (Netplan)

Ubuntu Server usa **Netplan**. Editar el archivo en `/etc/netplan/`:

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Aplicar la configuración:

```bash
sudo netplan apply
ip addr show enp0s3          # Verificar la IP asignada
```

### Demostración de conectividad cliente-servidor

```bash
# Desde el CLIENTE hacia el SERVIDOR
ping 192.168.1.100

# Verificar que el puerto web responde (una vez instalado el servidor)
curl http://192.168.1.100
```

---

## 2. Servidor web y `systemd` — *Vicente*

### Comparativa técnica: Nginx vs Apache

| Criterio | **Nginx** | **Apache (httpd)** |
|----------|-----------|--------------------|
| Arquitectura | Asíncrona, orientada a eventos | Basada en procesos/hilos |
| Rendimiento con estáticos | Muy alto | Bueno |
| Consumo de memoria | Bajo | Mayor bajo alta carga |
| Configuración | Centralizada (`nginx.conf`) | Modular + `.htaccess` |
| Casos de uso | Alta concurrencia, reverse proxy | Flexibilidad, módulos dinámicos |

**Decisión:** Para una intranet de contenido mayormente **estático (HTML/CSS)**, se
recomienda **Nginx** por su bajo consumo y eficiencia. *(Ambas opciones son válidas;
se documenta la elección tomada.)*

### Instalación del servicio

```bash
sudo apt update
sudo apt install -y nginx
```

### Configuración del VirtualHost (server block)

```nginx
# /etc/nginx/sites-available/intranet
server {
    listen 80;
    server_name intranet.local 192.168.1.100;

    root /var/www/intranet;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/intranet_access.log;
    error_log  /var/log/nginx/intranet_error.log;
}
```

Activar el sitio y recargar:

```bash
sudo ln -s /etc/nginx/sites-available/intranet /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default    # Deshabilitar sitio por defecto
sudo nginx -t                                  # Validar sintaxis
sudo systemctl reload nginx
```

### Verificación del arranque automático (systemd)

```bash
sudo systemctl enable nginx      # Habilitar inicio automático en el arranque
sudo systemctl start nginx
sudo systemctl status nginx      # Verificar estado: active (running)
systemctl is-enabled nginx       # Debe devolver: enabled
```

---

## 3. Contenido de la intranet — *Guille*

### Institución ficticia

**"Instituto Tecnológico Austral"** — portal interno para funcionarios, con acceso a
noticias, directorio de contactos y recursos internos.

### Estructura del sitio (mínimo 3 páginas)

```
/var/www/intranet/
├── index.html         # Página de inicio / bienvenida
├── contacto.html      # Directorio y datos de contacto
├── recursos.html      # Recursos internos y enlaces
└── css/
    └── estilos.css    # Hoja de estilos común
```

### Publicación del contenido

```bash
sudo mkdir -p /var/www/intranet/css
# Copiar los archivos HTML/CSS al directorio
sudo cp -r sitio/* /var/www/intranet/
```

El sitio debe incluir navegación entre las 3 páginas y una hoja de estilos CSS común
que dé identidad visual a la institución (logo, colores, cabecera y pie de página).

---

## 4. Usuarios, grupos y permisos — *Simón*

### Creación del grupo y usuario editor (sin `sudo`)

```bash
# Grupo de editores de la intranet
sudo groupadd editores

# Usuario editor SIN privilegios administrativos
sudo useradd -m -s /bin/bash -G editores editor_intranet
sudo passwd editor_intranet
```

> El usuario **no** se agrega al grupo `sudo`, por lo que no puede ejecutar comandos
> administrativos (principio de **mínimo privilegio**).

### Configuración de permisos POSIX sobre `/var/www/intranet`

```bash
# El grupo editores es dueño del directorio del sitio
sudo chown -R root:editores /var/www/intranet

# Permisos: dueño rwx, grupo rwx, otros r-x
sudo chmod -R 775 /var/www/intranet

# setgid: los archivos nuevos heredan el grupo editores
sudo chmod g+s /var/www/intranet
```

### Demostración del bloqueo de comandos administrativos

```bash
# Iniciar sesión como editor_intranet y probar:
whoami                       # editor_intranet
sudo systemctl restart nginx # -> "editor_intranet is not in the sudoers file"

# En cambio, SÍ puede editar el contenido del sitio:
nano /var/www/intranet/index.html   # permitido por pertenecer a 'editores'
```

---

## 5. Firewall y seguridad de red — *Pía*

### Configuración de `ufw` con política *default deny*

```bash
sudo apt install -y ufw

# Política por defecto: denegar todo lo entrante, permitir lo saliente
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### Apertura exclusiva de los puertos necesarios

```bash
sudo ufw allow 22/tcp      # SSH (administración)
sudo ufw allow 80/tcp      # HTTP (intranet)

sudo ufw enable            # Activar el firewall
sudo ufw status verbose    # Verificar reglas aplicadas
```

Resultado esperado:

```
Status: active
Default: deny (incoming), allow (outgoing)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
```

Cualquier otro puerto queda **bloqueado** por la política *default deny*.

---

## 6. Logs, evidencias y limitaciones — *Carlita*

### Análisis de `access.log` y `error.log`

```bash
# Ver accesos en tiempo real
sudo tail -f /var/log/nginx/intranet_access.log

# Contar respuestas por código de estado
sudo awk '{print $9}' /var/log/nginx/intranet_access.log | sort | uniq -c | sort -rn
```

### Códigos de estado a evidenciar

| Código | Significado | Cómo provocarlo |
|--------|-------------|-----------------|
| **200 OK** | Petición exitosa | Acceder a `http://192.168.1.100/index.html` |
| **403 Forbidden** | Acceso denegado por permisos | Quitar permisos de lectura a un archivo y solicitarlo |
| **404 Not Found** | Recurso inexistente | Acceder a `http://192.168.1.100/noexiste.html` |

```bash
# Errores del servidor
sudo tail -n 50 /var/log/nginx/intranet_error.log
```

### Respaldo de la configuración y el sitio

```bash
# Respaldo del sitio y de la configuración de nginx
sudo tar -czvf respaldo_intranet_$(date +%F).tar.gz \
    /var/www/intranet \
    /etc/nginx/sites-available/intranet
```

### Entrega final

- Recopilar capturas de pantalla como evidencia de cada etapa.
- Elaborar la presentación final.
- **Subir la presentación a Blackboard** dentro del plazo establecido.

---

## Limitaciones conocidas

- La intranet usa **HTTP sin cifrado**; en producción debería implementarse HTTPS (TLS).
- El acceso está restringido a la **red local**, no está expuesto a Internet.
- Contenido **estático**: no hay backend ni base de datos.
- La IP fija depende de la configuración de red del entorno de virtualización.

---

## Tabla resumen de comandos clave

| Tarea | Comando |
|-------|---------|
| Aplicar red | `sudo netplan apply` |
| Instalar servidor web | `sudo apt install -y nginx` |
| Validar config | `sudo nginx -t` |
| Habilitar arranque | `sudo systemctl enable nginx` |
| Crear usuario editor | `sudo useradd -m -G editores editor_intranet` |
| Firewall deny | `sudo ufw default deny incoming` |
| Abrir puertos | `sudo ufw allow 80/tcp` |
| Ver logs | `sudo tail -f /var/log/nginx/intranet_access.log` |
| Respaldo | `sudo tar -czvf respaldo.tar.gz /var/www/intranet` |

---

## Integrantes

- Benjamín — Infraestructura, distribución y red
- Vicente — Servidor web y `systemd`
- Guille — Contenido de la intranet
- Simón — Usuarios, grupos y permisos
- Pía — Firewall y seguridad de red
- Carlita — Logs, evidencias y limitaciones
