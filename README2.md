Esta configuración está diseñada para monitorizar los logs de la máquina host donde se ejecutan los tres servicios principales. La usaremos como punto de partida, de modo que más adelante podamos ampliarla para incluir otras máquinas.

Para la monitorización de un sistema de logging, hemos utilizado **Loki** como un servidor de almacenamiento, **Promtail** como agente de recolección y **Grafana** para la visualización y alertas.

Adicionalmente, se ha implementado **Nginx como un reverse proxy** delante de Loki. Su función es actuar como una capa de seguridad, gestionando una **autenticación básica (usuario y contraseña)** para todas las peticiones antes de que lleguen a Loki, asegurando así el acceso al sistema de logs.

Habitualmente, hemos diagnosticado problemas con la lectura directa de `/var/log/syslog`, por lo que implementaremos la siguiente arquitectura:

- El servicio **rsyslog** del sistema operativo, centraliza todos los logs en un único archivo. Exactamente en  `/var/log/all-logs.log`. Promtail se configura para leer únicamente este archivo, lo etiquetará y los enviará a Loki.

---

# **INSTALACIÓN Y CONFIGURACIÓN DE LOS SERVICIOS**

### [Loki]

Instalación del Loki

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.8/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki
```

Creación de usuarios y directorios.

```bash
sudo useradd --system --no-create-home --shell /bin/false loki
sudo mkdir -p /etc/loki
sudo mkdir -p /var/loki/{boltdb-shipper-active,boltdb-shipper-cache,chunks,compactor,wal}
sudo chown -R loki:loki /etc/loki /var/loki
```

Archivo de configuración  **(`/etc/loki/loki-config.yml`)**::

```yaml
auth_enabled: true

server:
  http_listen_port: 3101
  grpc_listen_port: 9097

ingester:
  wal:
    enabled: true
    dir: /var/loki/wal
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 30m
  chunk_retain_period: 1m
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /var/loki/boltdb-shipper-active
    cache_location: /var/loki/boltdb-shipper-cache
    cache_ttl: 48h
    shared_store: filesystem
  filesystem:
    directory: /var/loki/chunks

compactor:
  working_directory: /var/loki/compactor
  shared_store: filesystem
  retention_enabled: true

table_manager:
  retention_deletes_enabled: true
  retention_period: 72h

limits_config:
  ingestion_rate_mb: 30
  ingestion_burst_size_mb: 60
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
```

Configuración del servicio  **(`/etc/systemd/system/loki.service`)**:

```bash
[Unit]
Description=Loki log aggregation system
After=network.target

[Service]
User=loki
Group=loki
Type=simple
ExecStart=/usr/local/bin/loki -config.file /etc/loki/loki-config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Activación del servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
```

### ***[Promtail]***

Configuración del **rsyslog**

Crear el archivo (`/etc/rsyslog.d/99-promtail.conf`) con el siguiente contenido y reinciiar el servicio:

```bash
*.* /var/log/all-logs.log
sudo systemctl restart rsyslog
```

Instalación del Promtail

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.8/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```

Creación de usuarios y directorios.

```bash
sudo useradd --system --no-create-home --shell /bin/false promtail
sudo usermod -a -G adm promtail
sudo mkdir -p /etc/promtail/secret /var/lib/promtail
sudo chown -R promtail:promtail /etc/promtail/secret /var/lib/promtail
sudo touch /etc/promtail/secret/loki_user /etc/promtail/secret/passwd
sudo chmod 700 /etc/promtail/secret && sudo chmod 600 /etc/promtail/secret/loki_user /etc/promtail/secret/loki_passwd
```

Credenciales

```bash
sudo vim /etc/promtail/secret/loki_user
[usuari]

sudo vim /etc/promtail/secret/loki_passwd
[contraseña]
```

Archivo de configuración (`/etc/promtail/promtail-config.yml`):

```yaml
server:
  http_listen_port: 3102
  grpc_listen_port: 9096

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: "http://127.0.0.1:3100/loki/api/v1/push"
    basic_auth:
      username: "[usuario]"
      password_file: /etc/promtail/secret/loki_passwd

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          host: ansible_pau
          job: systemlogs
          __path__: /var/log/all-logs.log
    pipeline_stages:
      - regex:
          expression: ':\s+(?i)(?P<level>emergency|alert|critical|crit|error|exception|fatal|warn|warning)'
      - labels:
          level:
```

- **basic_auth:** Configuración para autenticar Promtail frente a Loki.
- **username:** Usuario utilizado en la autenticación.
- **password_file:** Ruta del archivo que contiene la contraseña.
    - **Nota:** Tanto **secret** como **loki_passwd** deben contar con los permisos adecuados de lectura y escritura, así como los del usuario y grupo correspondientes.

Configuración del servicio  **(`/etc/systemd/system/promtail.service`)**:

```bash
[Unit]
Description=Promtail agent for Loki
After=network.target

[Service]
User=promtail
Group=promtail
Type=simple
ExecStart=/usr/local/bin/promtail -config.file /etc/promtail/promtail-config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Activación del servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
```

### ***[Nginx]***

Instalación del Nginx::

```bash
sudo apt-get install nginx apache2-utils
```

Creación de un nuevo archivo, que contendrá el usuario y contraseña hasheados:

```bash
sudo htpasswd -c /etc/nginx/.httpasswd [user]
```

Configuración del Nginx como Reverse Proxy (`/etc/nginx/sites-available/loki`):

```bash
server {

    listen 3100;
    server_name AnsiblePau;

    location / {
        auth_basic "Area Restringida de Loki";
        auth_basic_user_file /etc/nginx/.htpasswd;
	
				proxy_http_version 1.1;
	    	proxy_set_header Upgrade $http_upgrade;
	    	proxy_set_header Connection "Upgrade";

        proxy_set_header X-Scope-OrgID admin;
        proxy_pass http://127.0.0.1:3101;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Activación y reinicio:

```bash
sudo ln -s /etc/nginx/sites-available/loki /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

Finalizamos este apartado accediendo a **Grafana**. Debemos asegurarnos de que en los **Data Sources** esté añadido **Loki**.

Una vez localizado, hacemos clic sobre Loki y procedemos a configurarlo para recibir los logs:

- **URL:** Dirección y puerto del servicio donde se encuentra Loki.
- **Authentication:**
    - **Basic Authentication**
    - **User:** user_prova (usuario definido en el archivo .yml de Promtail).
    - **Password_file:** Ruta del archivo que contiene la contraseña.

Finalmente, seleccionamos **Save & Test** para verificar la configuración.

# CONFIGURACIÓN DE LAS ALERTAS

Antes de nada, configuraremos la **Webhook URL de Slack** para poder integrarla en Grafana.

Dentro de la página de la API de Slack, accedemos a **Incoming Webhooks** en la sección **Features** -[**Acceso directo**](https://api.slack.com/)-.

Creamos un nuevo Webhook seleccionando el canal en el que queremos que el bot envíe las alertas. Una vez creado, se generará una URL que debemos guardar, ya que la utilizaremos más adelante en Grafana.

Dentro de la sección de Alerting, podremos ver 3 apartados a configurar para nuestra alerta:

- **Alert rules:** Definimos la confición que debe de cumplirse antes de que se active una regla de alerta.
- **Contact Points:** Configuramos quien recibe las notificaciones y como se envían.
- **Notification Policies:** Configuramos como se enrutan las instancias de alerta de activación de contacto.

### [Alert rules]

Clicaremos en “New alert rules” para empezar a rellenar lo siguiente:

1. **Enter alert rule name**
    - **Name Alert:** nombre de la alerta

1. **Define query and alert condition**
    
    ```bash
    count_over_time({host="ansible_pau"}
      |~ "(?i)error|fail|failed"
      != "loki"
      != "grafana"
      !~ "(?i)failed password|authentication failure"
    [10m])
    ```
    
    - **Add expression:**
        - **Input:** A
        - **IS ABOVE:** 2
        
2. **Set evaluation behavior**
    - **Folder:** Nombre de la carpeta donde se almacenará la alerta
    - **Evaluation group and interval: Descripción de nuestra alerta**
    - **Pending period:** None (Ya que la consulta tiene una memoria de 10 minutos, no necesitamos un periodo de espera largo)
    - **Configure no data and error handling:** Normal y Error

1. **Configure labels and notifications**
- **Contact Point:** Escogemos el canal de Slack donde queremos que vayan a parar los avisos
    
    Habilitamos override grouping
    
- **Group By:** Muting, aertname, host, hob.

Guardamos la configuración, y además, veremos en nuestra alerta en el apartado Grafana. Si clicamos al lápiz, para editar la evaluación del grupo, al tener una Query muy buena, pondremos que cada 1m nuestra consulta de la regla de nuestra alerta.

### **[Contact Points:]**

Creamos un nuevo punto de contacto:

- **Name:** Nombre para nuestro punto de contacto
- I**ntegration:** Slack
    - **Webhook URL:** [URL webhook del Slack]
    - **Optional Slack settings**
        - **Title:**
            
            ```go
            `*[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }} en {{ .CommonLabels.host }}*`
            ```
            
        
        - Text Body
            
            ```go
            `{{ range .Alerts }}
            {{ $query := (printf "{host=\"%s\"} |~ \"(?i)error|fail|failed\" != \"loki\" != \"grafana\"" .Labels.host) | urlquery }}`
            
            `{{ if eq .Status "firing" }}🚨 *ALERTA* 🚨{{ else }}✅ *ALERTA RESUELTA* ✅{{ end }}`
            
            `*Alerta:* {{ .Labels.alertname }}*Host:* {{ .Labels.host }}*Severidad:* {{ .Labels.severity }}`
            
            `<[http://192.168.68.70:3000/alerting/list|⚙️](http://192.168.68.70:3000/alerting/list%7C%E2%9A%99%EF%B8%8F) Revisar Alertas en Grafana> | <{{ .SilenceURL }}| 🔕 Silenciar>`
            
            `{{ end }}`
            ```
            

### [Alert rules]

Editaremos la **“Default Policy”** que tenemos configurada para las alertas y seleccionaremos nuestro **“Contact Point”**:

- **Group By:** grafana_folder, alertname
- **Group wait:** 30s → Espera 30 segundos antes de enviar la primera notificación. (Útil para agrupar varias alertas que se disparan al mismo tiempo).
- **Group interval:** 5m → Si ya existe una alerta activa, espera 5 minutos antes de notificar nuevas alertas del mismo grupo.
- **Repeat interval:** 4h → Si una alerta sigue activa, espera 4 horas antes de enviar un recordatorio.

# **PRUEBAS**

A la hora de realizar esto, bajaremos el tiempo de las alertas a los más bajo posible para que puedan saltar al Slack y hacer pruebas.

Para generar logs de error ficticios y probar las alertas, se puede usar este bucle:

```jsx
pau@AnsiblePau:~$ for i in {1..5}; do echo "$(date) AnsiblePau ERROR_VERIFICACION: Prueba $i" | sudo tee -a /var/log/all-logs.log; sleep 1; done
```

Para comprobar los logs de error de nuestra máquina, debemos de ejecutar en el “Data source” de Loki lo siguiente (como código):

```jsx
{host="ansible_pau"} |~ "(?i)error|fail|failed" != "loki" != "grafana"
```

Esto generará errores ficticios. Se mostrarán en la entrada de logs, pero también podríamos ver otros errores, dependiendo de cómo esté configurado el sistema. Es decir:

1. Filtrar los logs de las útlimas 8h
    
    ![image.png](attachment:2b0b114d-d2c0-49c3-8d39-7e8f7e4e3dc7:faa8fa3e-c01c-4bbc-8d84-64b45ff8d520.png)
    

1. Aquí vemos todos los logs en tiempo real, sin intervalo de actualización. Por lo tanto, al añadir un nuevo error ficticio, lo veremos inmediatamente, sin necesidad de recargar o ejecutar nuevamente la consulta.
    
    ![image.png](attachment:7c922618-494c-4852-93ec-1ce3213585b5:image.png)
    

# EJEMPLO - SOLUCIÓN DE UN ERROR PERSISTENTE

Si salta error en el servicio **snap.firmware-updater**, lo podemos desinstalar con el **snap** sin problema. 

Básicamente, este usa el paquete de SNAPS, en el cual, comprueba si hay firmware nuevo para los componentes del hardware (BIOS/UEFI). Estos errores saltan, ja que este servicio en una máquina virtual, no es relevante. 

Trata de buscar actualizaciones hacia componentes que no tiene acceso al estar en una máquina virtual y no tener un “acceso físico” al componente en si.

Además, esto ya viene gestionado por el VMware Tools o también se puede gestionar desde el propio ESXI.

```jsx
# Comprovar cuantos servicios hay del firmware
snap service firmware-updater

# Detener y deshabilitar los servicios que se han listado con el comando anterior
sudo snap stop --disable [servicio]

# Si listamos i no se llega a desactivar, lo podemos eliminar directamente
sudo snap remove firmware-updater
```
