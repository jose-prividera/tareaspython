# Runbook de Deploy — Hostinger VPS
**Proyecto:** Front-API / LaravelCiaf
**Entorno:** Producción — VPS Hostinger + MySQL 8
**Tiempo estimado:** 10-20 minutos
**Última actualización:** 2026-03-17

---

## Índice
1. [Pre-requisitos](#1-pre-requisitos)
2. [Checklist pre-deploy](#2-checklist-pre-deploy)
3. [Proceso de deploy](#3-proceso-de-deploy)
4. [Verificación post-deploy](#4-verificación-post-deploy)
5. [Rollback](#5-rollback)
6. [Casos especiales](#6-casos-especiales)
7. [Contactos de emergencia](#7-contactos-de-emergencia)

---

## 1. Pre-requisitos

### Accesos necesarios
- [ ] SSH al VPS Hostinger (pedir credenciales al dev principal)
- [ ] Acceso al panel Hostinger hPanel (para reiniciar servicios si es necesario)
- [ ] Acceso al repositorio Git (rama `main` o rama a deployar)

### Software en el servidor (verificar una vez)
```bash
php --version        # debe ser 8.2+
composer --version   # debe estar instalado globalmente
node --version       # debe ser 18+
npm --version
mysql --version      # debe ser 8.0+
git --version
```

### Directorios del proyecto en el servidor
```
/home/usuario/public_html/    # o la ruta que usa Hostinger para el dominio
```
> Confirmar ruta exacta con el dev principal antes del primer deploy.

---

## 2. Checklist Pre-Deploy

**El dev principal debe completar esto ANTES de avisar al devops:**

- [ ] Rama `main` (o rama de release) tiene todos los cambios finales
- [ ] `php artisan test` pasa al 100% en local
- [ ] Si hay nuevas migraciones: se ejecutaron en local con `php artisan migrate:fresh` sin errores
- [ ] Si hay nuevas dependencias composer: `composer.lock` está commiteado
- [ ] Si hay nuevas dependencias npm: `package-lock.json` está commiteado
- [ ] Si hay nuevos `.env` variables: están documentadas en `.env.example` y comunicadas al devops
- [ ] El `CHANGELOG` o mensaje de PR describe brevemente qué cambió

---

## 3. Proceso de Deploy

### Paso 1: Conectar al servidor

```bash
ssh usuario@ip-del-servidor-hostinger
# Ingresar contraseña o usar key SSH
```

### Paso 2: Navegar al directorio del proyecto

```bash
cd /home/usuario/public_html
# Verificar que estamos en el directorio correcto
ls -la
# Debe mostrar: artisan, composer.json, package.json, etc.
```

### Paso 3: Activar modo mantenimiento

```bash
php artisan down --message="Actualización en progreso. Volvemos en minutos." --retry=60
```

> Esto muestra una página de mantenimiento a los usuarios mientras se deploya.

### Paso 4: Traer los últimos cambios

```bash
git fetch origin
git status
# Verificar en qué rama estamos
git branch
# Si el deploy es desde main:
git pull origin main
# Si el deploy es desde una rama específica (ej: ciaf_v1.0.0):
git pull origin ciaf_v1.0.0
```

> **Si git pull falla por archivos locales modificados:**
> ```bash
> git stash          # guarda cambios locales temporalmente
> git pull origin main
> git stash drop     # descarta los cambios locales (NO usar git stash pop en producción)
> ```

### Paso 5: Instalar dependencias PHP

```bash
composer install --no-dev --optimize-autoloader
```

> `--no-dev` excluye dependencias de desarrollo (PHPUnit, etc.)
> `--optimize-autoloader` mejora la performance del autoloading

### Paso 6: Instalar y compilar assets frontend

```bash
npm ci                # más estricto que npm install, usa package-lock.json exacto
npm run build         # compila para producción
```

### Paso 7: Ejecutar migraciones de base de datos

```bash
# ¡IMPORTANTE! Verificar primero qué migraciones se van a ejecutar:
php artisan migrate --pretend

# Si el output muestra migraciones esperadas (las del PR), ejecutar:
php artisan migrate --force
```

> `--force` es necesario en producción (Laravel lo requiere para evitar accidentes).
> Si `--pretend` muestra algo inesperado, NO ejecutar y contactar al dev principal.

### Paso 8: Limpiar y optimizar caches

```bash
php artisan config:clear
php artisan cache:clear
php artisan view:clear
php artisan route:clear
php artisan config:cache    # genera cache optimizado de config
php artisan route:cache     # genera cache de rutas
php artisan view:cache      # precompila vistas Blade
```

### Paso 9: Reiniciar la cola de trabajos (si usa Queue)

```bash
php artisan queue:restart
```

### Paso 10: Levantar el sitio

```bash
php artisan up
```

---

## 4. Verificación Post-Deploy

Ejecutar estos pasos inmediatamente después del deploy:

```bash
# 1. Verificar que la app responde
curl -I https://tu-dominio.com
# Esperado: HTTP/1.1 200 OK

# 2. Verificar logs en busca de errores inmediatos
tail -n 50 storage/logs/laravel.log

# 3. Verificar que las migraciones se ejecutaron
php artisan migrate:status
# Todas deben mostrar [X] Ran

# 4. Verificar permisos de storage
ls -la storage/logs/
ls -la storage/app/
# El usuario del servidor web (www-data o similar) debe tener escritura
```

### Checklist visual (abrir en browser):
- [ ] Login funciona con `admin@example.com` / `password` (solo en staging — nunca en prod real)
- [ ] Dashboard del Programador carga sin errores
- [ ] El wizard de carga de workflows abre correctamente
- [ ] El historial de ejecuciones muestra datos

---

## 5. Rollback

Si algo falla después del deploy, seguir este proceso:

### Rollback rápido (código)

```bash
# Activar mantenimiento inmediatamente
php artisan down

# Volver al commit anterior
git log --oneline -5       # ver los últimos commits
git checkout HASH_ANTERIOR # reemplazar HASH_ANTERIOR con el commit previo al deploy

# Reinstalar dependencias del estado anterior
composer install --no-dev --optimize-autoloader
npm ci && npm run build

# Limpiar caches
php artisan config:clear && php artisan cache:clear && php artisan view:clear && php artisan route:clear
php artisan config:cache && php artisan route:cache

# Levantar el sitio
php artisan up
```

### Rollback de migraciones (solo si es absolutamente necesario)

```bash
# Ver las migraciones ejecutadas en este deploy
php artisan migrate:status

# Revertir la última migración (o las N últimas)
php artisan migrate:rollback --step=1   # revertir 1 migración
php artisan migrate:rollback --step=3   # revertir 3 migraciones
```

> **ADVERTENCIA:** El rollback de migraciones puede causar pérdida de datos si las tablas tienen registros.
> Contactar al dev principal antes de ejecutar `migrate:rollback` si hay dudas.

---

## 6. Casos Especiales

### Si es el primer deploy (setup inicial)

```bash
# Clonar el repositorio
git clone https://github.com/Noodle1981/Front-Api.git .

# Configurar entorno
cp .env.example .env
php artisan key:generate

# Editar .env con los valores de producción:
# - DB_DATABASE, DB_USERNAME, DB_PASSWORD
# - APP_URL=https://tu-dominio.com
# - PYTHON_API_URL=http://ip-servidor-python:5000
# - PYTHON_API_USE_MOCK=false  (o true si Python no está listo)

# Instalar y configurar
composer install --no-dev --optimize-autoloader
npm ci && npm run build
php artisan migrate --force
php artisan db:seed --force  # seeders básicos de roles y permisos

# Permisos de storage
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache  # ajustar usuario según Hostinger

# Compilar caches
php artisan config:cache && php artisan route:cache && php artisan view:cache

php artisan up
```

### Si hay errores de permisos en storage

```bash
chmod -R 775 storage
chmod -R 775 bootstrap/cache
# Si Hostinger usa un usuario específico (consultar en hPanel):
chown -R USUARIO_HOSTINGER:USUARIO_HOSTINGER storage bootstrap/cache
```

### Si el archivo .env no tiene las nuevas variables

```bash
# Ver qué variables nuevas hay en .env.example
diff .env .env.example
# Agregar manualmente las que falten con sus valores de producción
nano .env
php artisan config:clear && php artisan config:cache
```

### Si composer install falla por memory limit

```bash
php -d memory_limit=-1 /usr/bin/composer install --no-dev --optimize-autoloader
```

---

## 7. Contactos de Emergencia

| Situación | Contactar a |
|-----------|-------------|
| Bug crítico en producción post-deploy | Dev principal |
| Problemas de acceso SSH o hPanel | Hostinger Support |
| Python API no responde | Dev principal (verificar `PYTHON_API_USE_MOCK=true` como fix temporal) |
| Base de datos no conecta | Verificar credenciales en `.env` y estado del servicio MySQL en hPanel |

---

## Apéndice: Variables de Entorno de Producción

Variables que DEBEN estar configuradas en el `.env` de producción:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://tu-dominio.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=nombre_bd_produccion
DB_USERNAME=usuario_bd
DB_PASSWORD=password_bd_seguro

# Python API de conciliación
PYTHON_API_URL=http://ip-servidor-python:5000
PYTHON_API_USE_MOCK=false          # IMPORTANTE: false en producción

# Email (para alertas del sistema)
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=email@dominio.com
MAIL_PASSWORD=app_password_aqui
MAIL_ENCRYPTION=tls

# Sessions y cache
SESSION_DRIVER=file
CACHE_STORE=file
QUEUE_CONNECTION=sync

# Workflows
WORKFLOWS_CONCILIATION_TYPE_ID=1
```
