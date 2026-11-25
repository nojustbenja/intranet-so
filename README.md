# Intranet institucional — contenido de Guille

Proyecto de la Solemne 3 de Sistemas Operativos. Esta entrega corresponde al contenido
web de la intranet del **Instituto Tecnológico del Sur (ITS), Sede Bellavista**. El sitio
es estático y está preparado para servirse con Nginx en Ubuntu Server 24.04 LTS.

## Alcance de esta entrega

Se prepararon tres páginas HTML conectadas entre sí, una hoja de estilos compartida y
estas instrucciones para revisar y publicar el contenido. Las páginas usan enlaces
relativos: pueden navegarse entre sí dentro de la carpeta `intranet/` sin backend ni
servicios externos.

Esta entrega prepara los archivos del sitio; no instala ni configura Ubuntu, Nginx, la
red, el firewall o los permisos finales del servidor. Esas tareas corresponden a las
otras áreas del equipo.

## Archivos y contenido

```text
README.md
intranet/
├── index.html
├── procedimientos.html
├── horarios.html
└── css/
    └── styles.css
```

| Archivo | Contenido |
|---|---|
| `intranet/index.html` | Bienvenida para usuarios autorizados, tres noticias y accesos rápidos |
| `intranet/procedimientos.html` | Tres procedimientos resumidos, con código, área responsable, vigencia y pasos |
| `intranet/horarios.html` | Horarios de cuatro departamentos y cinco contactos internos |
| `intranet/css/styles.css` | Diseño compartido, institucional y adaptable a pantallas pequeñas |

El contenido, incluidos los nombres, noticias, horarios, anexos y correos, es ficticio.
El diseño utiliza CSS local y fuentes instaladas en el sistema; no carga imágenes,
JavaScript, fuentes web, CDNs ni otros recursos desde Internet. Las tres páginas incluyen
la navegación común y el pie institucional requerido.

En la portada, el enlace **“Archivo Histórico 2023”** apunta intencionalmente a
`archivo-inexistente.html`. Esa página no forma parte del sitio: al solicitarla, Nginx
debe devolver `404 Not Found`, útil para la demostración y revisión de registros.

## 1. Revisar el sitio en el equipo de desarrollo

Desde la raíz del repositorio, inicia un servidor web local:

```bash
python -m http.server 8080 --directory intranet
```

Abre `http://localhost:8080` en el navegador. Recorre los enlaces del menú y los accesos
rápidos; comprueba también que el estilo y las tablas se vean correctamente. Para detener
el servidor local, vuelve a la terminal y presiona `Ctrl+C`.

## 2. Preparar el servidor

Los siguientes pasos presuponen que Ubuntu Server ya está instalado, que Nginx está
instalado y que su configuración utiliza `/var/www/intranet` como raíz del sitio, con
`index.html` como página de inicio. Si Nginx aún no está configurado, esa preparación
debe coordinarse con el integrante responsable del servidor web.

### Opción A: copiar los archivos que ya están en el servidor

Si la carpeta del repositorio ya fue transferida al servidor, ejecuta estos comandos
desde la raíz del repositorio:

```bash
sudo mkdir -p /var/www/intranet
sudo cp -r intranet/. /var/www/intranet/
sudo chown -R root:root /var/www/intranet
sudo find /var/www/intranet -type d -exec chmod 755 {} \;
sudo find /var/www/intranet -type f -exec chmod 644 {} \;
sudo nginx -t
sudo systemctl reload nginx
```

Los permisos `755` en directorios permiten recorrerlos y `644` en archivos permite
leerlos. Son permisos iniciales para servir el contenido; el integrante responsable de
usuarios y permisos puede ajustarlos después para habilitar la edición del sitio.

### Opción B: transferir los archivos con `rsync`

Ejecuta el primer comando desde el equipo de desarrollo y la segunda parte en el
servidor. Reemplaza el usuario y la IP por los valores reales:

```bash
rsync -av intranet/ usuario_administrador@<IP_SERVIDOR>:/tmp/intranet/
ssh usuario_administrador@<IP_SERVIDOR>
```

Una vez conectado al servidor:

```bash
sudo mkdir -p /var/www/intranet
sudo rsync -av /tmp/intranet/ /var/www/intranet/
sudo chown -R root:root /var/www/intranet
sudo find /var/www/intranet -type d -exec chmod 755 {} \;
sudo find /var/www/intranet -type f -exec chmod 644 {} \;
sudo nginx -t
sudo systemctl reload nginx
```

### Opción C: obtener los archivos con Git en el servidor

Reemplaza `<URL_DEL_REPOSITORIO>` por la dirección del repositorio y ejecuta en Ubuntu:

```bash
cd /tmp
git clone <URL_DEL_REPOSITORIO> intranet-so
sudo mkdir -p /var/www/intranet
sudo cp -r /tmp/intranet-so/intranet/. /var/www/intranet/
sudo chown -R root:root /var/www/intranet
sudo find /var/www/intranet -type d -exec chmod 755 {} \;
sudo find /var/www/intranet -type f -exec chmod 644 {} \;
sudo nginx -t
sudo systemctl reload nginx
```

## 3. Comprobar la publicación

Desde el propio servidor, solicita las tres páginas:

```bash
curl -I http://127.0.0.1/index.html
curl -I http://127.0.0.1/procedimientos.html
curl -I http://127.0.0.1/horarios.html
```

Cada solicitud debe devolver `HTTP/1.1 200 OK`. Desde un equipo cliente de la misma red,
repite la comprobación usando la IP real del servidor:

```bash
curl -I http://<IP_SERVIDOR>/index.html
curl -I http://<IP_SERVIDOR>/procedimientos.html
curl -I http://<IP_SERVIDOR>/horarios.html
```

Para comprobar el 404 previsto y localizar su registro:

```bash
curl -I http://<IP_SERVIDOR>/archivo-inexistente.html
sudo tail -n 20 /var/log/nginx/access.log
```

La respuesta esperada para el archivo inexistente es `404 Not Found`. La ubicación del
registro puede variar si el administrador de Nginx configuró un archivo `access_log`
propio para la intranet.
