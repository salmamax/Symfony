# 🐳 Entorno de Desarrollo Symfony con Docker

Este proyecto proporciona un entorno de desarrollo local para aplicaciones Symfony utilizando Docker y Docker Compose. Es una guía práctica para entender cómo orquestar servicios con contenedores, lo cual es fundamental en DevOps.

---

## 📦 Estructura del proyecto

Mantendremos una estructura limpia y organizada:

```plaintext
symfony-docker/
│
├── docker-compose.yml         # Define todos los servicios y cómo se conectan.
│
├── php/                       # Carpeta para archivos relacionados con el contenedor PHP.
│   └── Dockerfile             # Instrucciones para construir la imagen PHP.
│
├── nginx/                     # Carpeta para archivos relacionados con el contenedor Nginx.
│   └── default.conf           # Configuración del servidor web Nginx.
│
└── (el código Symfony aquí)   # Aquí residirán los archivos de tu proyecto Symfony.
```

---

## ⚙️ Servicios incluidos

| Servicio | Imagen        | Función                                         | Notas                              |
|----------|---------------|-------------------------------------------------|------------------------------------|
| php      | Custom (8.3)  | Contenedor con PHP 8.3, extensiones y Composer  | Construido desde `./php/Dockerfile` |
| db       | mysql:5.7     | Contenedor con MySQL 5.7 y volumen persistente  | Compatible con más arquitecturas de CPU |
| nginx    | nginx:alpine  | Servidor web que expone Symfony en el puerto 8080 | Configurado por `./nginx/default.conf` |

---

## 📌 Requisitos previos

Asegúrate de tener instalado en tu sistema operativo (Ubuntu):

- **Docker:** Sigue las instrucciones oficiales para tu distribución.
- **Docker Compose:** Viene incluido con las instalaciones recientes de Docker Desktop o puede requerir instalación aparte en Linux.

---

## 🔨 Paso a paso para levantar el entorno

### 1. Crea la estructura inicial

Si aún no lo has hecho, crea las carpetas y los archivos de configuración básicos:

```bash
mkdir -p php nginx
touch php/Dockerfile nginx/default.conf docker-compose.yml
```

*(Asegúrate de estar en el directorio `~/Proyectos/symfony-docker` antes de ejecutar esto).*

### 2. Dockerfile para PHP 8.3 (`php/Dockerfile`)

Edita el archivo `php/Dockerfile` y pega el siguiente contenido. Este Dockerfile define cómo construir la imagen de nuestro contenedor PHP, incluyendo las extensiones necesarias y Composer.

```dockerfile
# Usamos la imagen oficial de PHP 8.3 con FPM.
FROM php:8.3-fpm

# Instalamos extensiones de PHP necesarias para Symfony y otras comunes.
RUN apt-get update && apt-get install -y \
    libpq-dev \
    libicu-dev \
    libxml2-dev \
    libzip-dev \
    libonig-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libwebp-dev \
    git \
    curl \
    && docker-php-ext-install -j$(nproc) pdo_mysql intl xml zip exif gd opcache

# Limpiamos la caché de apt.
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Instalamos Composer globalmente.
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# Establecemos el directorio de trabajo.
WORKDIR /var/www/symfony

# El contenedor PHP-FPM escucha por defecto en el puerto 9000.
EXPOSE 9000

# Comando por defecto al iniciar el contenedor.
CMD ["php-fpm"]
```

### 3. Configuración de Nginx (`nginx/default.conf`)

Edita el archivo `nginx/default.conf` y pega la configuración para que Nginx sirva la aplicación Symfony.

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/symfony/public;
    index index.php;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### 4. `docker-compose.yml`

Edita el archivo `docker-compose.yml` y define los servicios (php, db, nginx), sus imágenes, volúmenes, redes y dependencias.

```yaml
services:
  php:
    build:
      context: ./php
    volumes:
      - .:/var/www/symfony
    networks:
      - symfony
    depends_on:
      - db

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: symfony
      MYSQL_USER: symfony
      MYSQL_PASSWORD: symfony
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - symfony

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - .:/var/www/symfony
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    networks:
      - symfony
    depends_on:
      - php

volumes:
  db_data:

networks:
  symfony:
```

---

### 5. Limpiar y levantar el entorno Docker

Si has tenido intentos anteriores o archivos de Symfony en tu directorio local, es crucial limpiar antes de levantar el entorno para evitar conflictos, especialmente con el volumen montado.

Asegúrate de que tu directorio local `~/Proyectos/symfony-docker` contenga solo los archivos de configuración de Docker (docker-compose.yml, carpetas php/ y nginx/) antes de continuar. Si tienes archivos de Symfony (como `bin/`, `vendor/`, `composer.json`), elimínalos manualmente desde tu terminal local (puedes necesitar `sudo rm -r` para las carpetas si los permisos dan problemas).

Una vez limpio el directorio local, detén y elimina cualquier contenedor, red o volumen residual del intento anterior:

```bash
docker-compose down
```

Ahora, construye las imágenes (si es necesario) y levanta todos los servicios definidos en `docker-compose.yml`:

```bash
docker-compose up -d --build
```

El flag `--build` asegura que la imagen PHP se construya con la configuración de nuestro Dockerfile. El flag `-d` inicia los contenedores en segundo plano.

Verifica que los contenedores estén funcionando:

```bash
docker-compose ps
```

Deberías ver los servicios `db`, `php` y `nginx` en estado **Up**.

---

### 6. Instalar Symfony **DENTRO** del contenedor PHP (Método recomendado)

Es muy importante instalar el proyecto Symfony utilizando Composer dentro del contenedor PHP, ya que este tiene el entorno PHP y Composer configurados correctamente. Para evitar problemas con el volumen montado y el error "Project directory is not empty", instalaremos Symfony en un subdirectorio temporal y luego moveremos los archivos a la raíz del volumen montado.

Primero, entra al contenedor PHP:

```bash
docker-compose exec php bash
```

Una vez dentro del contenedor (`root@...:/var/www/symfony#`), crea un subdirectorio temporal:

```bash
mkdir symfony_temp
```

Navega a ese subdirectorio:

```bash
cd symfony_temp
```

Ahora, dentro del subdirectorio temporal, ejecuta el comando para crear el proyecto Symfony:

```bash
composer create-project symfony/skeleton:^7.0 .
```

El punto (`.`) instala Symfony en `/var/www/symfony/symfony_temp`.

Una vez terminada la instalación, navega de regreso a la raíz del directorio montado (`/var/www/symfony`):

```bash
cd ..
```

Ahora, mueve todos los contenidos del subdirectorio temporal a la raíz del directorio montado (incluyendo archivos ocultos):

```bash
find symfony_temp/ -mindepth 1 -maxdepth 1 -exec mv {} . \;
```

Finalmente, elimina el subdirectorio temporal vacío:

```bash
rmdir symfony_temp
```

Sal del contenedor:

```bash
exit
```

Ahora, tu directorio local `~/Proyectos/symfony-docker` debería contener los archivos de configuración de Docker **Y** los archivos del proyecto Symfony 7 en la raíz.

---

### 7. Acceder a la aplicación

Abre tu navegador web y visita:

👉 **http://localhost:8080**

Deberías ver la página de bienvenida de Symfony. Si la ves, significa que Nginx está sirviendo correctamente la aplicación desde el contenedor PHP.

---

### 🚀 Desarrollando con Symfony

Ahora que tu entorno está listo, puedes empezar a construir tu aplicación:

- **Conexión a la Base de Datos:** Edita el archivo `.env` en la raíz de tu proyecto Symfony para configurar la conexión a la base de datos MySQL. Usa el nombre del servicio `db` como host:

```env
DATABASE_URL="mysql://symfony:symfony@db:3306/symfony?serverVersion=5.7&charset=utf8mb4"
```

- **Comandos de Consola:** Cualquier comando de consola de Symfony (como `php bin/console make:...`, `doctrine:...`, etc.) debe ejecutarse dentro del contenedor PHP:

```bash
docker-compose exec php bash
php bin/console make:controller NombreController
exit
```

- **Instalar Dependencias:** Si necesitas añadir una nueva librería con Composer, hazlo dentro del contenedor PHP:

```bash
docker-compose exec php bash
composer require nombre/paquete
composer require --dev nombre/paquete-dev
exit
```

- **Hot-Reload:** Los cambios en los archivos de código en tu máquina local se reflejarán instantáneamente en la aplicación gracias al volumen montado.

---

### 🌱 Conceptos de DevOps aprendidos

Al completar esta guía, has aplicado y entendido conceptos clave de DevOps como:

- **Contenedores**
- **Orquestación de Contenedores**
- **Dockerfile**
- **Volúmenes**
- **Redes de Contenedores**
- **Troubleshooting**

Estos conocimientos son directamente transferibles a la configuración de entornos para otras aplicaciones y tecnologías.

---

### 🧑‍💻 Autor

Este proyecto fue creado paso a paso por [salmamax](https://github.com/salmamax) como parte de un ejercicio práctico de aprendizaje de DevOps.

---

### 📝 Licencia

Uso libre para fines educativos.
