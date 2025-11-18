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

## **Levantamiento del Proyecto en Docker**

<img width="1823" height="695" alt="Captura de pantalla 2025-11-18 102842" src="https://github.com/user-attachments/assets/30544d61-dbfd-42d4-b946-f54b05ccc6bf" />


<img width="1888" height="258" alt="Captura de pantalla 2025-11-18 102912" src="https://github.com/user-attachments/assets/e486d15f-1bb6-4d99-9b7a-c1e1dc0c713a" />

## **Aplicación web**

<img width="1897" height="873" alt="Captura de pantalla 2025-11-18 103022" src="https://github.com/user-attachments/assets/7b44b4b6-27d0-4acc-8947-12f89f454cce" />


<img width="1507" height="968" alt="Captura de pantalla 2025-11-18 103052" src="https://github.com/user-attachments/assets/c4323160-15e1-454a-846c-51aa1ca2e74b" />


## **Pruebas desde la terminal**

<img width="1911" height="467" alt="Captura de pantalla 2025-11-18 103242" src="https://github.com/user-attachments/assets/e81265c7-565d-41f5-ba4c-fd0e89d2766e" />



# Despliegue en la Nube con AWS EC2

## 1. **Configuración de la Instancia EC2 en AWS**

### Paso 1.1: Crear Instancia EC2
- Acceder a AWS Console → EC2 → Launch Instance
- **AMI:** Ubuntu Server 22.04 LTS
- **Tipo de Instancia:** t2.micro (Free Tier)
- **Par de claves:** Crear nuevo (flaskapp-key.pem)
- **Security Group:** Configurar reglas de entrada

### Paso 1.2: Configurar Security Groups
```bash
Reglas de Entrada (Inbound):
- SSH (22)    - 0.0.0.0/0
- HTTP (80)   - 0.0.0.0/0  
- HTTPS (443) - 0.0.0.0/0
```

### Paso 1.3: Obtener IP Pública
- En AWS Console: EC2 → Instances → Copiar "IPv4 Public IP"
- Ejemplo: `3.134.108.89`

## 2. **Conectar a la Instancia EC2**

### Paso 2.1: Preparar conexión SSH
```bash
# Dar permisos al archivo .pem
chmod 400 flaskapp.pem

# Conectar a la instancia
ssh -i "flaskapp.pem" ubuntu@3.134.108.89
```

## 3. **Configuración del Servidor EC2**

### Paso 3.1: Actualizar sistema
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git
```

### Paso 3.2: Instalar Docker
```bash
# Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Agregar usuario al grupo docker
sudo usermod -aG docker ubuntu

# Verificar instalación
docker --version
```

### Paso 3.3: Instalar Docker Compose
```bash
# Descargar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Dar permisos de ejecución
sudo chmod +x /usr/local/bin/docker-compose

# Verificar instalación
docker-compose --version
```

## 4. **Preparar la Aplicación en EC2**

### Paso 4.1: Crear estructura de directorios
```bash
mkdir -p /home/ubuntu/app
cd /home/ubuntu/app
```

### Paso 4.2: Transferir archivos desde máquina local
**En máquina LOCAL:**
```bash
# Comprimir proyecto
tar -czf flaskapp.tar.gz Dockerfile docker-compose.yml flaskapp/

# Transferir a EC2
scp -i "flaskapp.pem" flaskapp.tar.gz ubuntu@3.134.108.89:/home/ubuntu/app/
```

**En EC2:**
```bash
# Descomprimir
tar -xzf flaskapp.tar.gz
ls -la  # Verificar archivos
```

## 5. **Configuración Crítica para Producción**

### Problema Identificado y Solucionado:
**Error:** La aplicación redirigía a XAMPP en lugar de mostrar Flask
**Causa:** `ServerName localhost` en configuración de Apache
**Solución:** Cambiar a `ServerName _` y `ServerAlias *`

### Dockerfile Corregido
:
```dockerfile
# Configuración de Apache CORREGIDA
RUN printf '%s\n' \
    '# Virtual Host para HTTP - Redirección a HTTPS' \
    '<VirtualHost *:80>' \
    '    ServerName _' \
    '    ServerAlias *' \
    '    Redirect permanent / https://_/' \
    '    ErrorLog /var/log/apache2/flaskapp_error.log' \
    '    CustomLog /var/log/apache2/flaskapp_access.log combined' \
    '</VirtualHost>' \
    # ... resto de configuración
    > /etc/apache2/sites-available/flaskapp.conf
```

## 6. **Despliegue con Docker Compose**

### Paso 6.1: Construir y ejecutar
```bash
cd /home/ubuntu/app

# Construir imágenes
docker-compose build

# Ejecutar en segundo plano
docker-compose up -d
```

### Paso 6.2: Verificar despliegue
```bash
# Verificar contenedores activos
docker-compose ps

# Verificar configuración de Apache
docker-compose exec flaskapp apache2ctl -S

# Probar aplicación localmente
curl -k https://localhost/api/users
```

## 7. **Configuración de Seguridad**

### Paso 7.1: Configurar firewall
```bash
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw status
```

## 8. **Verificación de Acceso Remoto**

### Paso 8.1: Probar desde máquina local
```bash
# Probar HTTP (debe redirigir a HTTPS)
curl -I http://3.134.108.89

# Probar HTTPS
curl -k https://3.134.108.89

# Probar API
curl -k https://3.134.108.89/api/users
```

### Paso 8.2: Probar desde navegador
- Abrir: `https://3.134.108.89`
- Aceptar certificado autofirmado


# Capturas del Despliegue


##**Reglas de Seguridad de la Instancia**

<img width="1596" height="706" alt="Captura de pantalla 2025-11-18 104839" src="https://github.com/user-attachments/assets/81105119-7bee-41f1-b9c3-40aa4b237290" />

<img width="1581" height="392" alt="Captura de pantalla 2025-11-18 105003" src="https://github.com/user-attachments/assets/ca1b0522-4f54-4ee3-b3b3-65b4d095c6cd" />


##**Conexión a la instancia mediante SSH**

<img width="990" height="247" alt="Captura de pantalla 2025-11-18 105026" src="https://github.com/user-attachments/assets/56fcbaad-ea59-49f2-a28b-7fb4d63f8464" />

##**Estructura del Proyecto web**

<img width="862" height="146" alt="Captura de pantalla 2025-11-18 110715" src="https://github.com/user-attachments/assets/d98411e7-b1b4-41c1-8ed0-3bc64a9f2432" />



##**Aplicación web desde la nube de la instancia EC2**

<img width="1919" height="692" alt="Captura de pantalla 2025-11-18 105105" src="https://github.com/user-attachments/assets/d0920d45-f98f-4ac3-9a09-6e020cf612d8" />

<img width="1296" height="893" alt="Captura de pantalla 2025-11-18 105218" src="https://github.com/user-attachments/assets/b0591296-b4a2-41de-8f80-ed19a3dd07ad" />



# **Monitoreo con Prometheus y Node Exporter**

##  **Pasos Completos Realizados**

### **1. Instalación de Node Exporter**

#### **Creación de usuario y directorios:**
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

#### **Descarga e instalación:**
```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm -rf node_exporter-1.7.0.linux-amd64*
```

#### **Configuración del servicio systemd:**
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

**Contenido del servicio:**
```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

#### **Inicio y habilitación:**
```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

---

### **2. Instalación de Prometheus**

#### **Preparación del entorno:**
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

#### **Descarga e instalación:**
```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-amd64.tar.gz
tar xzf prometheus-2.47.0.linux-amd64.tar.gz
sudo cp prometheus-2.47.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.47.0.linux-amd64/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo cp -r prometheus-2.47.0.linux-amd64/consoles /etc/prometheus/
sudo cp -r prometheus-2.47.0.linux-amd64/console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus/consoles /etc/prometheus/console_libraries
```

---

### **3. Configuración de prometheus.yml**

```bash
sudo nano /etc/prometheus/prometheus.yml
```

**Configuración final:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

**Aplicar permisos:**
```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

---

### **4. Configuración de Alertas Básicas**

#### **Creación de reglas de alerta:**
```bash
sudo nano /etc/prometheus/alert_rules.yml
```

**Reglas configuradas:**
```yaml
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on instance {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes."

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on instance {{ $labels.instance }}"
          description: "Memory usage is above 80% for 5 minutes."

      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk space is below 20% on mount point {{ $labels.mountpoint }}"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Service {{ $labels.job }} on {{ $labels.instance }} has been down for more than 1 minute."
```

**Aplicar permisos:**
```bash
sudo chown prometheus:prometheus /etc/prometheus/alert_rules.yml
```

---

### **5. Servicio Systemd para Prometheus**

```bash
sudo nano /etc/systemd/system/prometheus.service
```

**Configuración del servicio:**
```ini
[Unit]
Description=Prometheus Time Series Collection and Processing Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090
Restart=always

[Install]
WantedBy=multi-user.target
```

---

### **6. Inicio y Configuración Final**

#### **Permisos y inicio de servicios:**
```bash
sudo chown -R prometheus:prometheus /var/lib/prometheus
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

#### **Configuración de firewall:**
```bash
sudo ufw allow 9090/tcp  # Prometheus
sudo ufw allow 9100/tcp  # Node Exporter
```

#### **Verificación:**
```bash
sudo systemctl status prometheus
sudo systemctl status node_exporter
curl http://localhost:9100/metrics | head -10
curl http://localhost:9090/api/v1/targets
```

---

##  **Documentación de las 3 Métricas Específicas**

### **1. `node_cpu_seconds_total`**
- **Descripción**: Tiempo total de CPU en cada modo (idle, user, system)
- **Utilidad**: Monitorear uso de CPU, detectar cuellos de botella
- **Consulta**: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

### **2. `node_memory_MemAvailable_bytes`**  
- **Descripción**: Memoria RAM disponible para aplicaciones
- **Utilidad**: Prevenir swapping, planificar upgrades de memoria
- **Consulta**: `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100`

### **3. `node_filesystem_avail_bytes`**
- **Descripción**: Espacio disponible en sistemas de archivos
- **Utilidad**: Evitar que el sistema se quede sin espacio
- **Consulta**: `(node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes`

---

## **Verificación Final**

### **Comandos de validación:**
```bash
# Servicios activos
sudo systemctl is-active prometheus
sudo systemctl is-active node_exporter

# Puertos escuchando
sudo netstat -tulpn | grep -E '(9090|9100)'

# Targets en Prometheus
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job, instance, health}'

# Métricas disponibles
curl -s http://localhost:9100/metrics | wc -l
```


# Capturas del Despliegue


##**Comprobando que los Servicios están funcionando**

<img width="1907" height="518" alt="Captura de pantalla 2025-11-18 110821" src="https://github.com/user-attachments/assets/422e387b-282b-49ea-9c51-b04a75009bb9" />

<img width="1919" height="488" alt="Captura de pantalla 2025-11-18 111058" src="https://github.com/user-attachments/assets/ae6d6f44-eb4f-4eb7-8cdb-37397ee529f8" />

<img width="1523" height="289" alt="Captura de pantalla 2025-11-18 111145" src="https://github.com/user-attachments/assets/09781736-4293-453b-9a1f-bf6d1bb44ef6" />


##**Apartado web**

<img width="1916" height="621" alt="Captura de pantalla 2025-11-18 111335" src="https://github.com/user-attachments/assets/3ef9eca2-29ab-44f2-9042-ae857b75b11e" />

<img width="1913" height="547" alt="Captura de pantalla 2025-11-18 111359" src="https://github.com/user-attachments/assets/a8c60094-90af-4856-9782-9f62b6807773" />




# **Visualización con Grafana**

## **1. Instalación de Grafana**

### **Instalación en Ubuntu/Debian:**
```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar dependencias
sudo apt install -y software-properties-common apt-transport-https

# Agregar repositorio de Grafana
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Instalar Grafana
sudo apt update
sudo apt install -y grafana

# Habilitar e iniciar servicio
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

### **Configurar firewall:**
```bash
sudo ufw allow 3000/tcp
sudo ufw status
```

### **Acceso inicial:**
- **URL**: `http://TU_IP_EC2:3000`
- **Usuario**: `admin`
- **Contraseña**: `admin` (cambiar en primer login)

---

## 🔗 **2. Configurar Prometheus como Fuente de Datos**

### **Desde la interfaz web:**
1. **Login** en Grafana (`http://IP:3000`)
2. **Configuration** → **Data Sources** → **Add data source**
3. **Seleccionar "Prometheus"**

### **Configuración:**
```yaml
# En la interfaz de Grafana:
Name: Prometheus
URL: http://localhost:9090
Access: Server (default)

# Click: Save & Test
# Debe mostrar: "Data source is working"
```


---

## 📊 **3. Crear Dashboard Personalizado**

### **Panel 1: Uso de CPU y Memoria (Time Series)**

#### **Crear nuevo dashboard:**
1. **Create** → **Dashboard** → **Add new panel**



## 📥 **4. Importar Dashboard Preconfigurado**

### **Dashboard recomendado: Node Exporter Full**
- **ID**: `1860` 
- **Fuente**: [Grafana Dashboard Library](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)

### **Pasos para importar:**
1. **Dashboards** → **New** → **Import**
2. **Poner ID**: `1860`
3. **Click**: Load
4. **Seleccionar data source**: Prometheus
5. **Click**: Import

### **Verificación del dashboard importado:**
- Debe mostrar múltiples paneles automáticamente
- CPU, memoria, disco, red, load average
- Todos los paneles deben tener datos

---

##  **Comandos de Verificación**

### **Verificar servicios:**
```bash
# Grafana
sudo systemctl status grafana-server

# Prometheus
sudo systemctl status prometheus

# Node Exporter
sudo systemctl status node_exporter

# Puertos abiertos
sudo netstat -tulpn | grep -E '(3000|9090|9100)'
```

### **Probar conectividad:**
```bash
# Grafana
curl -I http://localhost:3000

# Prometheus
curl -I http://localhost:9090

# Node Exporter
curl -I http://localhost:9100
```

Capturas deñ Despliegue

##**Comprobando funcionamiento del servicio**

<img width="1908" height="540" alt="Captura de pantalla 2025-11-18 113000" src="https://github.com/user-attachments/assets/0a2cc735-2511-4ccd-944c-598f686c4361" />

<img width="1915" height="492" alt="Captura de pantalla 2025-11-18 113043" src="https://github.com/user-attachments/assets/ae81470b-031f-4d09-9ed8-d40097b020f7" />



##**Panel del Login**

<img width="1455" height="974" alt="Captura de pantalla 2025-11-18 113146" src="https://github.com/user-attachments/assets/be3989a1-1e33-41cc-a728-e7b99bbee3e6" />



##**Configuración de Grafana**

<img width="1919" height="608" alt="Captura de pantalla 2025-11-18 115047" src="https://github.com/user-attachments/assets/53b9fa50-1602-4780-afbc-cdde083e61dd" />

<img width="1488" height="724" alt="Captura de pantalla 2025-11-18 115127" src="https://github.com/user-attachments/assets/20267224-f89a-4a36-9126-13b87c5b3b59" />

<img width="1540" height="485" alt="Captura de pantalla 2025-11-18 115204" src="https://github.com/user-attachments/assets/f33c0177-32f0-4abd-b3b6-92037237940a" />


##**Panel del Dashboard Personalizado**


<img width="1435" height="627" alt="Captura de pantalla 2025-11-18 113557" src="https://github.com/user-attachments/assets/e78e6ca9-418a-42d0-a8e2-0bb0e1ca2d11" />



##**Panel del Dashboard Preconfigurado**

##**ID**: 1860

<img width="1515" height="810" alt="Captura de pantalla 2025-11-18 114830" src="https://github.com/user-attachments/assets/5fae6fa7-9a7b-406c-a743-07557e398503" />

<img width="1512" height="574" alt="Captura de pantalla 2025-11-18 114912" src="https://github.com/user-attachments/assets/e3a5c4af-5b17-412b-a106-73d336e390c3" />


















