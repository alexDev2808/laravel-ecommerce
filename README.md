# Guía técnica: Laravel en Fedora 43 con Apache (httpd)

**Entorno objetivo:**  
- Fedora Linux 43  
- Escritorio GNOME  
- Shell zsh  
- Servidor web Apache (httpd)  
- Laravel en `/home/{username}/Documents/laravel/ecommerce`

Esta guía explica paso a paso cómo:

- Instalar PHP, Composer y Apache en Fedora 43  
- Crear y configurar un proyecto Laravel  
- Configurar Apache (httpd) con VirtualHost  
- Ajustar permisos y SELinux para que todo funcione  
- Solucionar errores típicos de permisos (SQLite, logs, MySQL)

---

## 1. Instalación de requisitos

```zsh
sudo dnf install php php-cli php-common php-mbstring php-xml php-json php-zip php-pdo php-mysqlnd php-sqlite3 php-gd php-opcache php-intl php-bcmath
sudo dnf install composer httpd
sudo systemctl enable --now httpd
```

---

## 2. Crear el proyecto Laravel

```zsh
mkdir -p ~/Documents/laravel
cd ~/Documents/laravel
composer create-project laravel/laravel ecommerce
```

Ruta final del proyecto:

```text
/home/{username}/Documents/laravel/ecommerce
```

---

## 3. Configurar base de datos SQLite

Crear el archivo de base de datos:

```zsh
touch /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
```

Editar `.env` en la raíz del proyecto y configurar SQLite:

```env
DB_CONNECTION=sqlite
DB_DATABASE=/home/{username}/Documents/laravel/ecommerce/database/database.sqlite
```

Limpiar configuración en Laravel:

```zsh
cd /home/{username}/Documents/laravel/ecommerce
php artisan config:clear
php artisan cache:clear
```

---

## 4. Configurar Apache (VirtualHost)

Crear archivo de configuración:

```zsh
sudo nano /etc/httpd/conf.d/ecommerce.conf
```

Contenido (ajusta el usuario si es necesario):

```apache
<VirtualHost *:80>
    ServerName ecommerce-app.local
    DocumentRoot /home/{username}/Documents/laravel/ecommerce/public

    <Directory /home/{username}/Documents/laravel/ecommerce/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/ecommerce-error.log
    CustomLog /var/log/httpd/ecommerce-access.log combined
</VirtualHost>
```

Editar `/etc/hosts` para registrar el dominio local:

```zsh
sudo nano /etc/hosts
```

Añadir al final:

```text
127.0.0.1   ecommerce-app.local
```

---

## 5. Permisos UNIX para Laravel

```zsh
# Permitir a Apache atravesar directorios
chmod 711 /home/{username}
chmod 755 /home/{username}/Documents
chmod 755 /home/{username}/Documents/laravel
chmod 755 /home/{username}/Documents/laravel/ecommerce

# Dar grupo apache a carpetas críticas
sudo chgrp -R apache /home/{username}/Documents/laravel/ecommerce/storage
sudo chgrp -R apache /home/{username}/Documents/laravel/ecommerce/bootstrap/cache
sudo chgrp -R apache /home/{username}/Documents/laravel/ecommerce/database

# Permisos de escritura para el grupo
sudo chmod -R 775 /home/{username}/Documents/laravel/ecommerce/storage
sudo chmod -R 775 /home/{username}/Documents/laravel/ecommerce/bootstrap/cache
sudo chmod -R 775 /home/{username}/Documents/laravel/ecommerce/database

# Asegurar el archivo SQLite
sudo touch /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
sudo chmod 664 /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
sudo chgrp apache /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
```

---

## 6. SELinux: permitir acceso a Laravel y MySQL

Habilitar lectura de contenido de usuario y home dirs:

```zsh
sudo setsebool -P httpd_read_user_content 1
sudo setsebool -P httpd_enable_homedirs 1
```

Permitir a Apache conectarse a MySQL y otros servicios de red:

```zsh
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_can_network_connect 1
```

Aplicar contextos SELinux correctos a Laravel:

```zsh
sudo chcon -R -t httpd_sys_content_t /home/{username}/Documents/laravel/ecommerce/public
sudo chcon -R -t httpd_sys_rw_content_t /home/{username}/Documents/laravel/ecommerce/storage
sudo chcon -R -t httpd_sys_rw_content_t /home/{username}/Documents/laravel/ecommerce/bootstrap/cache
sudo chcon -R -t httpd_sys_rw_content_t /home/{username}/Documents/laravel/ecommerce/database
```

---

## 7. Claves de app, migraciones y limpieza de caché

```zsh
cd /home/{username}/Documents/laravel/ecommerce
php artisan key:generate
php artisan migrate
php artisan config:clear
php artisan cache:clear
php artisan view:clear
php artisan config:cache
```

Reiniciar Apache:

```zsh
sudo systemctl restart httpd
```

Abrir en el navegador:

```text
http://ecommerce-app.local
```

---

## 8. Errores típicos y soluciones rápidas

### 8.1 `attempt to write a readonly database`

Causa: SQLite sin permisos de escritura.

Solución (ya incluida arriba):

```zsh
sudo chmod 664 /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
sudo chgrp apache /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
sudo chcon -t httpd_sys_rw_content_t /home/{username}/Documents/laravel/ecommerce/database/database.sqlite
```

### 8.2 `The stream or file "storage/logs/laravel.log" could not be opened`

Causa: Apache no puede escribir en `storage/logs`.

Solución:

```zsh
sudo chgrp -R apache /home/{username}/Documents/laravel/ecommerce/storage
sudo chmod -R 775 /home/{username}/Documents/laravel/ecommerce/storage
sudo chcon -R -t httpd_sys_rw_content_t /home/{username}/Documents/laravel/ecommerce/storage
```

### 8.3 `SQLSTATE[HY000] [2002] Permission denied`

Causa: SELinux bloquea conexión MySQL.

Solución:

```zsh
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_can_network_connect 1
```

---

## 9. Cheat sheet (resumen rápido)

```bash
# Crear proyecto
cd ~/Documents/laravel
composer create-project laravel/laravel ecommerce

# VirtualHost
sudo nano /etc/httpd/conf.d/ecommerce.conf

# Hosts
sudo nano /etc/hosts
# 127.0.0.1 ecommerce-app.local

# Permisos
chmod 711 /home/{username}
chmod 755 /home/{username}/Documents /home/{username}/Documents/laravel /home/{username}/Documents/laravel/ecommerce
sudo chgrp -R apache ecommerce/storage ecommerce/bootstrap/cache ecommerce/database
sudo chmod -R 775 ecommerce/storage ecommerce/bootstrap/cache ecommerce/database

# SELinux
sudo setsebool -P httpd_read_user_content 1 httpd_enable_homedirs 1 httpd_can_network_connect_db 1 httpd_can_network_connect 1
sudo chcon -R -t httpd_sys_content_t ecommerce/public
sudo chcon -R -t httpd_sys_rw_content_t ecommerce/storage ecommerce/bootstrap/cache ecommerce/database

# Laravel
php artisan key:generate
php artisan migrate
php artisan config:clear && php artisan cache:clear && php artisan view:clear && php artisan config:cache

# Reiniciar Apache
sudo systemctl restart httpd
```