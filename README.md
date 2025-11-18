# 3re_Parcial_Servicios-Telematicos

# Empaquetado y Despliegue Local con Docker

## 1. **Clonar el repositorio, copiarlo en /var/www/flaskapp y preparar el entorno**

```bash
# Acceder al servidor y navegar al directorio de trabajo
cd /var/www

# Estructura inicial del proyecto Flask existente
flaskapp/
├── config.py
├── flaskapp.wsgi
├── web/
│   └── views.py
└── users/
    ├── models/
    │   ├── db.py
    │   ├── user_model.py
    │   └── product_model.py
    └── controllers/
        ├── user_controller.py
        └── product_controller.py
```

## 2. **Configurar Apache con HTTPS y redirección HTTP→HTTPS**

### Generar certificado SSL autofirmado:
```bash
sudo mkdir -p /etc/ssl/private /etc/ssl/certs
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/apache-selfsigned.key \
    -out /etc/ssl/certs/apache-selfsigned.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=192.168.50.4"
```

### Configurar Virtual Host de Apache (`/etc/apache2/sites-available/flaskapp.conf`):
```apache
# Redirección HTTP → HTTPS
<VirtualHost *:80>
    ServerName 192.168.50.4
    Redirect permanent / https://192.168.50.4/
</VirtualHost>

# Configuración HTTPS
<VirtualHost *:443>
    ServerName 192.168.50.4
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
    
    # Configuración WSGI para Flask
    WSGIDaemonProcess flaskapp user=www-data group=www-data threads=5
    WSGIScriptAlias / /var/www/flaskapp/flaskapp.wsgi
    
    <Directory /var/www/flaskapp>
        WSGIProcessGroup flaskapp
        Require all granted
    </Directory>
</VirtualHost>
```

### Habilitar configuración:
```bash
sudo a2enmod ssl rewrite
sudo a2ensite flaskapp.conf
sudo systemctl restart apache2
```

## 3. **Crear Dockerfile**

```dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    apache2 \
    libapache2-mod-wsgi-py3 \
    python3 \
    python3-pip \
    openssl \
    && a2enmod ssl \
    && a2enmod rewrite \
    && rm -rf /var/lib/apt/lists/*

# Generar certificado SSL
RUN mkdir -p /etc/ssl/private /etc/ssl/certs
RUN openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/apache-selfsigned.key \
    -out /etc/ssl/certs/apache-selfsigned.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"

# Configuración de Apache para Docker
RUN cat > /etc/apache2/sites-available/flaskapp.conf << 'EOF'
<VirtualHost *:80>
    ServerName localhost
    ServerAlias *
    Redirect permanent / https://localhost/
</VirtualHost>

<VirtualHost *:443>
    ServerName localhost
    ServerAlias *
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
    
    WSGIDaemonProcess flaskapp user=www-data group=www-data threads=5
    WSGIScriptAlias / /var/www/flaskapp/flaskapp.wsgi
    
    <Directory /var/www/flaskapp>
        WSGIProcessGroup flaskapp
        Require all granted
    </Directory>
</VirtualHost>
EOF

# Habilitar sitio y copiar aplicación
RUN a2ensite flaskapp.conf && a2dissite 000-default.conf
COPY ./flaskapp /var/www/flaskapp

# Instalar dependencias Python
RUN pip3 install --upgrade pip
RUN pip3 install flask==2.3.3 flask-sqlalchemy==3.0.5 PyMySQL==1.0.2 cryptography==41.0.7

# Ajustar permisos
RUN chown -R www-data:www-data /var/www/flaskapp

EXPOSE 80 443
CMD ["apache2ctl", "-D", "FOREGROUND"]
```

## 4. **Crear docker-compose.yml**

```yaml
services:
  db:
    image: mysql:8.0
    container_name: mysql-db
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: myflaskapp
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - flaskapp-network
    restart: unless-stopped

  flaskapp:
    build: .
    ports:
      - "80:80"
      - "443:443"
    container_name: flaskapp-ssl-container
    volumes:
      - ./flaskapp:/var/www/flaskapp
    environment:
      - MYSQL_HOST=db
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=myflaskapp
    depends_on:
      - db
    restart: unless-stopped
    networks:
      - flaskapp-network

networks:
  flaskapp-network:
    driver: bridge

volumes:
  db_data:
```

## 5. **Configurar la aplicación Flask para Docker**

### Actualizar `config.py`:
```python
import os

class Config:
    MYSQL_HOST = os.environ.get('MYSQL_HOST', 'db')
    MYSQL_USER = os.environ.get('MYSQL_USER', 'root')
    MYSQL_PASSWORD = os.environ.get('MYSQL_PASSWORD', 'root')
    MYSQL_DB = os.environ.get('MYSQL_DB', 'myflaskapp')
    SQLALCHEMY_DATABASE_URI = f'mysql+pymysql://{MYSQL_USER}:{MYSQL_PASSWORD}@{MYSQL_HOST}/{MYSQL_DB}'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

### Crear `requirements.txt`:
```
Flask==2.3.3
Flask-SQLAlchemy==3.0.5
PyMySQL==1.0.2
cryptography==41.0.7
Werkzeug==2.3.7
```

## 6. **Desplegar y Verificar**

### Construir y ejecutar contenedores:
```bash
docker-compose down -v
docker-compose build --no-cache
docker-compose up -d
```

### Verificar funcionamiento:
```bash
# Verificar contenedores
docker-compose ps

# Probar HTTPS
curl -k https://localhost

# Verificar redirección HTTP → HTTPS
curl -I http://localhost

# Probar API
curl -k https://localhost/api/users

# Verificar base de datos
docker-compose exec db mysql -u root -proot -e "USE myflaskapp; SHOW TABLES;"
```

### Verificar en navegador:
- Acceder a: `https://192.168.50.4` 
- HTTP redirige automáticamente a HTTPS

## **Estructura Final del Proyecto**
```
/var/www/
├── Dockerfile
├── docker-compose.yml
├── flaskapp/
│   ├── config.py
│   ├── flaskapp.wsgi
│   ├── requirements.txt
│   ├── web/
│   │   └── views.py
│   └── users/
│       ├── models/
│       └── controllers/

```


# Capturas del Despliegue

<img width="1823" height="695" alt="Captura de pantalla 2025-11-18 102842" src="https://github.com/user-attachments/assets/30544d61-dbfd-42d4-b946-f54b05ccc6bf" />

