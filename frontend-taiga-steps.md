# 5. Frontend
## Download the code from GitHub:
```bash
cd ~
git clone https://github.com/taigaio/taiga-front-dist.git taiga-front-dist
cd taiga-front-dist
git checkout stable
```


## Copy the example config file:
```bash
cp ~/taiga-front-dist/dist/conf.example.json ~/taiga-front-dist/dist/conf.json
```


## Edit the example configuration following the pattern below (replace with your own details):
```js
{
	"api": "http://example.com/api/v1/",
	"eventsUrl": "ws://example.com/events",
	"debug": "true",
	"publicRegisterEnabled": true,
	"feedbackEnabled": true,
	"privacyPolicyUrl": null,
	"termsOfServiceUrl": null,
	"GDPRUrl": null,
	"maxUploadFileSize": null,
	"contribPlugins": []
}
```

# 6. Events Setup

## The taiga-events module depends on rabbitmq-server as its message broker.

## Download the code:
```bash
cd ~
git clone https://github.com/taigaio/taiga-events.git taiga-events
cd taiga-events
```

## Install Node.js
```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
```


## Install the required JavaScript dependencies:
```bash
npm install
npm audit
```


## Create config.json file based on the provided example.
```bash
cp config.example.json config.json
```


## Update it with your RabbitMQ URL and your unique secret key. Your final config.json should look similar to the following example:
```json
{
    "url": "amqp://taiga:PASSWORD_FOR_EVENTS@localhost:5672/taiga",
    "secret": "theveryultratopsecretkey",
    "webSocketServer": {
        "port": 8888
    }
}
```


## The secret value in config.json must be the same as the SECRET_KEY in ~/taiga-back/settings/local.py!
```
k3E9$2x&ZB3jPTM#1D9s5&HPb
```

## Add taiga-events to systemd configuration.

## Copy-paste the code below into `/etc/systemd/system/taiga_events.service`
```ini
[Unit]
Description=taiga_events
After=network.target

[Service]
User=taiga
WorkingDirectory=/home/taiga/taiga-events
ExecStart=/bin/bash -c "node_modules/coffeescript/bin/coffee index.coffee"
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```


## Reload the systemd configurations:
```bash
sudo systemctl daemon-reload
sudo systemctl start taiga_events
sudo systemctl enable taiga_events
```


## To verify that the service is running, execute the following command:
```bash
sudo systemctl status taiga
```


# 7. Start and Expose Taiga

Before moving further, make sure you installed taiga-back and taiga-front-dist, however, having installed them is insufficient to run Taiga.

taiga-back should run under an application server, which in turn, should be executed and monitored by a process manager. For this task we will use gunicorn and systemd respectively.

Both taiga-front-dist and taiga-back must be exposed to the outside using a proxy/static-file web server. For this purpose, Taiga uses NGINX

## 7.1. systemd and gunicorn

**systemd** is the process supervisor used by Ubuntu, and Taiga uses it to run **gunicorn**.
**systemd** is not only for executing processes, but it also has utils for monitoring them, collecting logs, and restarting processes if something goes wrong, and for starting processes on system boot.

### Create a new systemd file at `/etc/systemd/system/taiga.service` to run taiga-back:
```ini
[Unit]
Description=taiga_back
After=network.target

[Service]
User=taiga
Environment=PYTHONUNBUFFERED=true
WorkingDirectory=/home/taiga/taiga-back
ExecStart=/home/taiga/.virtualenvs/taiga/bin/gunicorn --workers 4 --timeout 60 -b 127.0.0.1:8001 taiga.wsgi
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```


### Reload the systemd daemon and start the taiga service:
```bash
sudo systemctl daemon-reload
sudo systemctl start taiga
sudo systemctl enable taiga
```

### To verify that the service is running, execute the following command:
```bash
sudo systemctl status taiga
```

## 7.2. NGINX



### NGINX is used as a static file web server to serve taiga-front-dist and send proxy requests to taiga-back.
### Remove the default NGINX config file to avoid collision with Taiga:

```bash
sudo rm /etc/nginx/sites-enabled/default
```


### Create the logs folder (mandatory)
```bash
mkdir -p ~/logs
```

### To configure a new NGINX virtualhost for Taiga, create and edit the `/etc/nginx/conf.d/taiga.conf` file, as follows:

```nginx
server {
    listen 80 default_server;
    server_name pm.adplexus.com;  #  See http://nginx.org/en/docs/http/server_names.html

    large_client_header_buffers 4 32k;
    client_max_body_size 50M;
    charset utf-8;

    access_log /home/taiga/logs/nginx.access.log;
    error_log /home/taiga/logs/nginx.error.log;

    # Frontend
    location / {
        root /home/taiga/taiga-front-dist/dist/;
        try_files $uri $uri/ /index.html;
    }

    # Backend
    location /api {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8001/api;
        proxy_redirect off;
    }

    # Admin access (/admin/)
    location /admin {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8001$request_uri;
        proxy_redirect off;
    }

    # Static files
    location /static {
        alias /home/taiga/taiga-back/static;
    }

    # Media files
    location /media {
        alias /home/taiga/taiga-back/media;
    }

    # Events
    location /events {
        proxy_pass http://127.0.0.1:8888/events;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
```


### Execute the following command to verify the NGINX configuration and to track any error in the service:
```bash
sudo nginx -t
```



### Finally, restart the nginx service:
```bash
sudo systemctl restart nginx
```

Now you should have the service up and running on: http://pm.adplexus.com/
