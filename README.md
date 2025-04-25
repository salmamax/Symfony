# **🐳 Entorno de Desarrollo Symfony con Docker**

Este proyecto proporciona un entorno de desarrollo local para aplicaciones Symfony utilizando Docker y Docker Compose. Es una guía práctica para entender cómo orquestar servicios con contenedores, lo cual es fundamental en DevOps.

## **📦 Estructura del proyecto**

Mantendremos una estructura limpia y organizada:

symfony-docker/  
│  
├── docker-compose.yml         \# Define todos los servicios y cómo se conectan.  
│  
├── php/                       \# Carpeta para archivos relacionados con el contenedor PHP.  
│   └── Dockerfile             \# Instrucciones para construir la imagen PHP.  
│  
├── nginx/                     \# Carpeta para archivos relacionados con el contenedor Nginx.  
│   └── default.conf           \# Configuración del servidor web Nginx.  
│  
└── (el código Symfony aquí)   \# Aquí residirán los archivos de tu proyecto Symfony.

## **⚙️ Servicios incluidos**

| Servicio | Imagen | Función | Notas |
| :---- | :---- | :---- | :---- |
| php | Custom (8.3) | Contenedor con PHP 8.3, extensiones y Composer | Construido desde ./php/Dockerfile |
| db | mysql:5.7 | Contenedor con MySQL 5.7 y volumen persistente | Compatible con más arquitecturas de CPU |
| nginx | nginx:alpine | Servidor web que expone Symfony en el puerto 8080 | Configurado por ./nginx/default.conf |

## **📌 Requisitos previos**

Asegúrate de tener instalado en tu sistema operativo (Ubuntu):

* **Docker:** Sigue las instrucciones oficiales para tu distribución.  
* **Docker Compose:** Viene incluido con las instalaciones recientes de Docker Desktop o puede requerir instalación aparte en Linux.

## **🔨 Paso a paso para levantar el entorno**

Sigue estos pasos en tu terminal, desde el directorio raíz del proyecto (\~/Proyectos/symfony-docker):

### **1\. Crea la estructura inicial**

Si aún no lo has hecho, crea las carpetas y los archivos de configuración básicos:

mkdir \-p php nginx  
touch php/Dockerfile nginx/default.conf docker-compose.yml

(Asegúrate de estar en el directorio \~/Proyectos/symfony-docker antes de ejecutar esto).

### **2\. Dockerfile para PHP 8.3 (**php/Dockerfile**)**

Edita el archivo php/Dockerfile y pega el siguiente contenido. Este Dockerfile define cómo construir la imagen de nuestro contenedor PHP, incluyendo las extensiones necesarias y Composer.

\# Usamos la imagen oficial de PHP 8.3 con FPM.  
FROM php:8.3-fpm

\# Instalamos extensiones de PHP necesarias para Symfony y otras comunes.  
\# Corregimos el nombre del comando docker-php-ext-install (con guion, no guion bajo).  
RUN apt-get update && apt-get install \-y \\  
    libpq-dev \\  
    libicu-dev \\  
    libxml2-dev \\  
    libzip-dev \\  
    libonig-dev \\  
    libpng-dev \\  
    libjpeg-dev \\  
    libfreetype6-dev \\  
    libwebp-dev \\  
    git \\  
    curl \\  
    && docker-php-ext-install \-j$(nproc) pdo\_mysql intl xml zip exif gd opcache

\# Limpiamos la caché de apt.  
RUN apt-get clean && rm \-rf /var/lib/apt/lists/\*

\# Instalamos Composer globalmente.  
COPY \--from=composer:latest /usr/bin/composer /usr/local/bin/composer

\# Establecemos el directorio de trabajo.  
WORKDIR /var/www/symfony

\# El contenedor PHP-FPM escucha por defecto en el puerto 9000\.  
EXPOSE 9000

\# Comando por defecto al iniciar el contenedor.  
CMD \["php-fpm"\]

### **3\. Configuración de Nginx (**nginx/default.conf**)**

Edita el archivo nginx/default.conf y pega la configuración para que Nginx sirva la aplicación Symfony.

server {  
    listen 80;  
    server\_name localhost;

    root /var/www/symfony/public;  
    index index.php;

    location / {  
        try\_files $uri /index.php$is\_args$args;  
    }

    location \~ \\.php$ {  
        fastcgi\_pass php:9000;  
        fastcgi\_index index.php;  
        include fastcgi\_params;  
        fastcgi\_param SCRIPT\_FILENAME $document\_root$fastcgi\_script\_name;  
    }

    location \~ /\\.ht {  
        deny all;  
    }  
}

### **4\. docker-compose.yml**

Edita el archivo docker-compose.yml y define los tres servicios (php, db, nginx), sus imágenes, volúmenes, redes y dependencias. Es crucial usar la imagen mysql:5.7 para la compatibilidad de CPU.

\# Puedes eliminar la línea 'version: "3.8"' para evitar advertencias de obsolescencia.

services:  
  php:  
    build:  
      context: ./php \# Construye desde el Dockerfile en ./php  
    volumes:  
      \# Monta el directorio actual (código Symfony, archivos Docker) en el contenedor  
      \- .:/var/www/symfony  
    networks:  
      \- symfony  
    depends\_on:  
      \- db \# Asegura que la base de datos inicie antes que PHP

  db:  
    image: mysql:5.7 \# Usamos la versión 5.7 por compatibilidad de CPU  
    restart: always  
    environment:  
      \# Credenciales de la base de datos (¡cambia en producción\!)  
      MYSQL\_ROOT\_PASSWORD: root  
      MYSQL\_DATABASE: symfony  
      MYSQL\_USER: symfony  
      MYSQL\_PASSWORD: symfony  
    volumes:  
      \# Volumen persistente para los datos de MySQL  
      \- db\_data:/var/lib/mysql  
    networks:  
      \- symfony

  nginx:  
    image: nginx:alpine \# Imagen ligera de Nginx  
    ports:  
      \# Mapea el puerto 80 del contenedor al puerto 8080 de tu máquina local  
      \- "8080:80"  
    volumes:  
      \# Monta el directorio del código y la configuración de Nginx  
      \- .:/var/www/symfony  
      \- ./nginx/default.conf:/etc/nginx/conf.d/default.conf  
    networks:  
      \- symfony  
    depends\_on:  
      \- php \# Asegura que PHP inicie antes que Nginx

\# Definición del volumen persistente para la base de datos  
volumes:  
  db\_data:

\# Definición de la red interna para los servicios  
networks:  
  symfony:

### **5\. Limpiar y levantar el entorno Docker**

Si has tenido intentos anteriores o archivos de Symfony en tu directorio local, es crucial limpiar antes de levantar el entorno para evitar conflictos, especialmente con el volumen montado.

Asegúrate de que tu directorio local \~/Proyectos/symfony-docker contenga **solo** los archivos de configuración de Docker (docker-compose.yml, carpetas php/ y nginx/) antes de continuar. Si tienes archivos de Symfony (como bin/, vendor/, composer.json), elimínalos manualmente desde tu terminal local (puedes necesitar sudo rm \-r para las carpetas si los permisos dan problemas).

Una vez limpio el directorio local, detén y elimina cualquier contenedor, red o volumen residual del intento anterior:

docker-compose down

Ahora, construye las imágenes (si es necesario) y levanta todos los servicios definidos en docker-compose.yml:

docker-compose up \-d \--build

El flag \--build asegura que la imagen PHP se construya con la configuración de nuestro Dockerfile. El flag \-d inicia los contenedores en segundo plano.

Verifica que los contenedores estén funcionando:

docker-compose ps

Deberías ver los servicios db, php y nginx en estado Up.

### **6\. Instalar Symfony DENTRO del contenedor PHP (Método recomendado)**

Es **muy importante** instalar el proyecto Symfony utilizando Composer **dentro del contenedor PHP**, ya que este tiene el entorno PHP y Composer configurados correctamente. Para evitar problemas con el volumen montado y el error "Project directory is not empty", instalaremos Symfony en un subdirectorio temporal y luego moveremos los archivos a la raíz del volumen montado.

Primero, entra al contenedor PHP:

docker-compose exec php bash

Una vez dentro del contenedor (root@...:/var/www/symfony\#), crea un subdirectorio temporal:

mkdir symfony\_temp

Navega a ese subdirectorio:

cd symfony\_temp

Ahora, **dentro del subdirectorio temporal**, ejecuta el comando para crear el proyecto Symfony:

composer create-project symfony/skeleton:^7.0 .

El punto (.) instala Symfony en /var/www/symfony/symfony\_temp.

Una vez terminada la instalación, navega de regreso a la raíz del directorio montado (/var/www/symfony):

cd ..

Ahora, mueve todos los contenidos del subdirectorio temporal a la raíz del directorio montado (incluyendo archivos ocultos):

find symfony\_temp/ \-mindepth 1 \-maxdepth 1 \-exec mv {} . \\;

Finalmente, elimina el subdirectorio temporal vacío:

rmdir symfony\_temp

Sal del contenedor:

exit

Ahora, tu directorio local \~/Proyectos/symfony-docker debería contener los archivos de configuración de Docker Y los archivos del proyecto Symfony 7 en la raíz.

### **7\. Acceder a la aplicación**

Abre tu navegador web y visita:

👉 **http://localhost:8080**

Deberías ver la página de bienvenida de Symfony. Si la ves, significa que Nginx está sirviendo correctamente la aplicación desde el contenedor PHP.

## **🚀 Desarrollando con Symfony**

Ahora que tu entorno está listo, puedes empezar a construir tu aplicación:

* **Conexión a la Base de Datos:** Edita el archivo .env en la raíz de tu proyecto Symfony para configurar la conexión a la base de datos MySQL. Usa el nombre del servicio db como host:  
  \# .env  
  \# ... otras variables ...  
  DATABASE\_URL="mysql://symfony:symfony@db:3306/symfony?serverVersion=5.7\&charset=utf8mb4"  
  \# ...

* **Comandos de Consola:** Cualquier comando de consola de Symfony (como php bin/console make:..., doctrine:..., etc.) debe ejecutarse **dentro del contenedor PHP**:  
  docker-compose exec php bash  
  \# Dentro del contenedor:  
  php bin/console make:controller NombreController  
  \# ... otros comandos ...  
  exit

* **Instalar Dependencias:** Si necesitas añadir una nueva librería con Composer, hazlo **dentro del contenedor PHP**:  
  docker-compose exec php bash  
  \# Dentro del contenedor:  
  composer require nombre/paquete  
  composer require \--dev nombre/paquete-dev  
  exit

* **Hot-Reload:** Los cambios en los archivos de código en tu máquina local se reflejarán instantáneamente en la aplicación gracias al volumen montado.

## **🔍 Solución de problemas comunes**

Aquí hay algunos problemas que podrías encontrar y cómo solucionarlos, basándonos en nuestra experiencia:

### **❌ Error en la construcción del Dockerfile:** docker-php-ext\_install: not found

* **Causa:** Un typo en el nombre del comando docker-php-ext-install en tu Dockerfile.  
* **Solución:** Edita php/Dockerfile y corrige docker-php-ext\_install a docker-php-ext-install (cambia el guion bajo \_ por un guion \-). Luego, reconstruye la imagen PHP con docker-compose up \-d \--build.

### **❌ Contenedor** db **reiniciándose con** Fatal glibc error: CPU does not support x86-64-v2

* **Causa:** La imagen por defecto de MySQL 8.0 requiere instrucciones de CPU que tu procesador no soporta. Es una incompatibilidad de hardware.  
* **Solución:** Edita docker-compose.yml y cambia la imagen del servicio db de mysql:8.0 a mysql:5.7. Luego, baja y sube los contenedores (docker-compose down seguido de docker-compose up \-d \--build). MySQL 5.7 tiene una compatibilidad más amplia.

### **❌ Error al instalar Symfony:** "Project directory is not empty"

* **Causa:** El directorio donde Composer intenta crear el proyecto (/var/www/symfony, tu directorio local montado) contiene archivos previos (como los archivos de configuración de Docker o instalaciones anteriores de Symfony).  
* **Solución:** La mejor manera de manejar esto es seguir el **Paso 6** ("Instalar Symfony DENTRO del contenedor PHP (Método recomendado)") utilizando el subdirectorio temporal. Esto evita la necesidad de borrar manualmente la raíz del directorio montado antes de la instalación principal. Si por alguna razón el directorio montado contiene archivos que no se eliminaron con el método del subdirectorio, puedes intentar eliminarlos manualmente **dentro del contenedor** con rm (con cuidado de no borrar archivos importantes).

### **❌ Error al generar tests con** make:controller**:** "Class "PHPUnit\\Framework.TestCase" not found"

* **Causa:** Intentaste generar tests unitarios para el controlador (respondiendo "yes" a la pregunta interactiva o usando un flag si existiera) sin tener la librería PHPUnit instalada en el entorno del contenedor PHP.  
* **Solución:**  
  1. **Opción Rápida (no generar tests):** Al ejecutar php bin/console make:controller, responde no a la pregunta Do you want to generate PHPUnit tests? \[Experimental\] (yes/no) \[no\]:. La opción \--no-test no existe en esta versión del Maker Bundle.  
  2. **Opción Completa (instalar PHPUnit):** Si planeas escribir tests, instala PHPUnit como dependencia de desarrollo dentro del contenedor PHP: composer require \--dev phpunit/phpunit. Luego podrás generar tests sin problemas.

### **❌ Permisos denegados al eliminar archivos en la máquina local**

* **Causa:** Los archivos en el volumen montado fueron creados por el usuario root dentro del contenedor, y tu usuario local de Ubuntu no tiene permisos para modificarlos o eliminarlos.  
* **Solución:** En algunos casos, puedes usar sudo rm \-r en tu terminal local para forzar la eliminación (con cuidado). Sin embargo, nuestra experiencia sugiere que esto puede ser inconsistente. La mejor forma de **evitar** encontrarse en esta situación durante la instalación de Symfony es utilizar el **método del subdirectorio temporal** descrito en el **Paso 6**, ya que evita la necesidad de borrar la raíz del volumen montado antes de la instalación principal. Si necesitas cambiar permisos en archivos después de que son creados por el contenedor, puedes intentar usar chmod o chown dentro del contenedor PHP (ya que eres root allí) en los directorios o archivos específicos (por ejemplo, para los directorios var/ o public/ si hay problemas de escritura).

### **❌ Hot-reload no funciona (los cambios en el código no se ven)**

* **Causa:** Problemas con el volumen montado o permisos que impiden que el contenedor PHP o Nginx vean los cambios en los archivos locales.  
* **Solución:**  
  1. Verifica en tu docker-compose.yml que el volumen \- .:/var/www/symfony esté correctamente definido para los servicios php y nginx.  
  2. Asegúrate de que no haya errores de permisos que impidan al usuario dentro del contenedor leer o escribir en los archivos del volumen. Puedes intentar darle permisos amplios a ciertos directorios (como var/ y public/) desde tu terminal local con chmod (por ejemplo, sudo chmod \-R 777 var public con precaución, ya que 777 es muy permisivo).

## **🌱 Conceptos de DevOps aprendidos**

Al completar esta guía, has aplicado y entendido conceptos clave de DevOps como:

* **Contenedores:** Aislamiento y empaquetado de aplicaciones y servicios.  
* **Orquestación de Contenedores:** Gestión de múltiples contenedores interconectados (docker-compose).  
* **Dockerfile:** Definición reproducible de entornos de aplicación.  
* **Volúmenes:** Persistencia de datos y compartición de archivos entre el host y los contenedores.  
* **Redes de Contenedores:** Permitir la comunicación segura entre servicios.  
* **Troubleshooting:** Diagnóstico y resolución de problemas en entornos distribuidos.

Estos conocimientos son directamente transferibles a la configuración de entornos para otras aplicaciones y tecnologías utilizando las mismas herramientas.

## **🧑‍💻 Autor**

Este proyecto fue creado paso a paso con la asistencia de Gemini como parte de un ejercicio práctico de aprendizaje de DevOps.

## **📝 Licencia**

Uso libre para fines educativos.