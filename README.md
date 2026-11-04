# devops-1050B

WordPress (PHP-FPM) + Nginx + MariaDB con **Podman sin root** (probado en Debian 13).

| Servicio    | Contenedor | Imagen                          | Puerto            |
|-------------|------------|---------------------------------|-------------------|
| `db`        | `wp-db`    | `docker.io/library/mariadb:11.4` | interno (3306)    |
| `wordpress` | `wp-app`   | `localhost/wp-app:1.0`          | interno (9000)    |
| `nginx`     | `wp-nginx` | `localhost/wp-nginx:1.0`        | `8080` → 80       |

Solo Nginx se publica en el host. La base de datos y PHP-FPM únicamente son accesibles desde la red interna.

## Requisitos

## Configuración

Copia el archivo de variables y cambia las contraseñas:

```bash
sudo -iu podman
cd ~/unir

git clone https://github.com/MiguelGuzmanDev/devops-1050B.git
cd devops-1050B/

cp env.example .env
vim .env
```

Variables:

| Variable           | Descripción                        |
|--------------------|------------------------------------|
| `DB_NAME`          | Nombre de la base de datos         |
| `DB_USER`          | Usuario de WordPress en MariaDB    |
| `DB_PASSWORD`      | Contraseña de `DB_USER`            |
| `DB_ROOT_PASSWORD` | Contraseña de `root` de MariaDB    |

## Levantar los contenedores (podman-compose)

```bash
# Construir las imágenes wp-app y wp-nginx
podman-compose build

# Levantar todo en segundo plano
podman-compose up -d

# Ver el estado de los contenedores
podman-compose ps
```

Abre en el navegador: `http://IP-DE-LA-VM:8080`

> Podman sin root no puede abrir puertos < 1024, por eso se usa el `8080` en el host.

## Comandos útiles

```bash
# Logs de todos los servicios (o de uno: podman-compose logs -f nginx)
podman-compose logs -f

# Estado del healthcheck de MariaDB
podman healthcheck run wp-db && echo "db OK"

# Entrar a un contenedor
podman exec -it wp-app bash
podman exec -it wp-nginx sh

# Conectarse a MariaDB
podman exec -it wp-db mariadb -u root -p

# Validar y recargar la configuración de Nginx
podman exec wp-nginx nginx -t
podman exec wp-nginx nginx -s reload

# Reconstruir tras cambiar un Dockerfile o la configuración
podman-compose build
podman-compose up -d

# Reiniciar servicios
podman-compose restart
```

## Detener y limpiar

```bash
# Detener y eliminar los contenedores (los datos se conservan en los volúmenes)
podman-compose down

# Detener y eliminar TAMBIÉN los volúmenes (borra la base de datos y WordPress)
podman-compose down -v

# Ver volúmenes e imágenes
podman volume ls
podman images
```

## Arranque automático tras reiniciar la VM

`restart: unless-stopped` solo actúa mientras el usuario tiene sesión. Para que los contenedores sin root arranquen con el sistema:

```bash
# Permitir que los servicios del usuario sigan activos sin sesión iniciada
sudo loginctl enable-linger $USER

# Servicio que reinicia los contenedores con política "restart"
systemctl --user enable --now podman-restart.service
```

## Estructura

```
.
├── compose.yaml          # Definición de los servicios
├── env.example           # Plantilla de variables (.env)
├── wordpress/
│   ├── Dockerfile        # Imagen wp-app (WordPress PHP-FPM)
│   └── uploads.ini       # Límites de subida de PHP (64M)
└── nginx/
    ├── Dockerfile        # Imagen wp-nginx
    └── default.conf      # Configuración de Nginx para WordPress
```
