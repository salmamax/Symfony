# 🐘 Symfony 7 + Docker + Nginx + MySQL + Hot Reload

Este proyecto proporciona un entorno de desarrollo completo para Symfony 7 usando Docker y Docker Compose. El objetivo es disponer de una infraestructura lista para trabajar cómodamente en modo desarrollo con soporte para hot-reload y persistencia de datos.

---

## 📦 Estructura del proyecto

```
symfony-docker/
│
├── docker-compose.yml
│
├── php/
│   └── Dockerfile
│
├── nginx/
│   └── default.conf
│
└── (el código Symfony se instala directamente aquí)
```

---

## ⚙️ Servicios incluidos

| Servicio | Función |
|---------|--------|
| `php`   | Contenedor con PHP 8.3, extensiones necesarias y Composer |
| `db`    | Contenedor con MySQL 8.0 y volumen persistente |
| `nginx` | Servidor web que expone Symfony en el puerto `8080` |

---

## 🔨 Paso a paso para levantar el entorno

### 1. Crea la estructura

```bash
mkdir -p symfony-docker/php symfony-docker/nginx
cd symfony-docker
touch php/Dockerfile nginx/default.conf docker-compose.yml
```

### 2. Dockerfile para PHP 8.3 (`php/Dockerfile`)

```Dockerfile
FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
    libicu-dev \
    libxml2-dev \
    libzip-dev \
    unzip \
    git \
    zip \
    curl \
    && docker-php-ext-install intl pdo_mysql xml zip

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/symfony
```

### 3. Configuración de Nginx (`nginx/default.conf`)

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

### 4. docker-compose.yml

```yaml
version: '3.8'

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
    image: mysql:8.0
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

## 🚀 Instalación de Symfony

1. Levanta los contenedores:

```bash
docker-compose up -d --build
```

2. Entra al contenedor de PHP:

```bash
docker-compose exec php bash
```

3. **Crea el proyecto Symfony (muy importante hacerlo dentro del contenedor)**:

```bash
composer create-project symfony/skeleton:^7.0 .
```

4. Instala paquetes útiles:

```bash
composer require twig symfony/asset
composer require symfony/maker-bundle --dev
```

---

## ✅ Acceder a la aplicación

- Visita en tu navegador:  
  👉 http://localhost:8080

---

## 🔁 Hot Reload

- Al editar archivos `.php` o `.twig`, los cambios se ven al instante al recargar el navegador.
- Funciona automáticamente gracias al volumen: `.:/var/www/symfony`

---

## 🧪 Crear controlador de ejemplo

```bash
docker-compose exec php bash
php bin/console make:controller InicioController --no-test
```

Y accede a:  
👉 http://localhost:8080/inicio

---

## ❗Errores comunes y soluciones

### ❌ **Error: “File not found” en http://localhost:8080**
Este error ocurre cuando Nginx no encuentra la carpeta `public/`, que es donde Symfony coloca el archivo `index.php`.

✔️ Soluciones:

1. Asegúrate de que **has creado el proyecto Symfony dentro del contenedor**, no en tu sistema local:

```bash
docker-compose exec php bash
composer create-project symfony/skeleton:^7.0 .
```

2. Verifica que la carpeta `public/` existe en la raíz del proyecto.

3. Si ya creaste el proyecto, **reinicia los contenedores** para que Nginx lo detecte correctamente:

```bash
docker-compose down
docker-compose up -d --build
```

4. Vuelve a acceder a:  
👉 http://localhost:8080

---

### ❌ **Error: “Project directory is not empty”**
Significa que estás intentando crear un proyecto Symfony donde ya hay archivos.

✔️ Solución:

```bash
rm -rf ./* .[^.]*
composer create-project symfony/skeleton:^7.0 .
```

En caso de que se os borren todo lo creado anteriormente.
Simplemente crear las carpetas y archivos nuevamente.

Nuevamente

```bash
docker-compose down
docker-compose up -d --build
```

---

### ❌ **Error: Class "TestCase" not found**
Ocurre si Symfony intenta crear un test y no tienes PHPUnit instalado.

✔️ Solución 1 (rápida):
```bash
php bin/console make:controller Nombre --no-test
```

✔️ Solución 2 (completa):
```bash
composer require --dev phpunit/phpunit
```

---

### ❌ **Error al usar `composer require annotations`**
No es compatible con Symfony 7 porque ya no se usa `framework-extra-bundle`.

✔️ Solución:  
No lo instales. Usa rutas con atributos modernos:

```php
#[Route('/inicio', name: 'inicio')]
```

---

### ❌ **Hot-reload no funciona**
Verifica que el volumen está correctamente montado:

```yaml
volumes:
  - .:/var/www/symfony
```

También asegúrate de no tener errores de permisos:

```bash
chmod -R 777 var public
```

---

## 📌 Requisitos previos

- Docker
- Docker Compose
- Composer (opcional, se instala dentro del contenedor)

---

## 🧑‍💻 Autor

Proyecto creado paso a paso por Jose Sotos ([@SokiSotos](https://github.com/SokiSotos)) como parte del curso ASIR 🖥️

---

## 📝 Licencia

Uso libre para fines educativos ✨
