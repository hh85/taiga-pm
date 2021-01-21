# Taiga PM Installation Documentation
Full documentation https://taigaio.github.io/taiga-doc/dist/setup-production.html

# Follow the full documentation for Steps 1 and 2
### After which follow the revisions below closely in comparison to the full docs because I had to adjust some things to account for versions being a bit behind at the time when I did the installation steps.

# 3. Prerequisites

## 3.1. Installing Dependencies
### Essential packages:
```bash
sudo apt-get update
sudo apt-get install -y build-essential binutils-doc autoconf flex bison libjpeg-dev
sudo apt-get install -y libfreetype6-dev zlib1g-dev libzmq3-dev libgdbm-dev libncurses5-dev
sudo apt-get install -y automake libtool curl git tmux gettext
sudo apt-get install -y nginx
sudo apt-get install -y rabbitmq-server redis-server  # Optional: taiga-events or async tasks
```


### The taiga-back module depends on PostgreSQL (> 9.4) as its database:
```bash
sudo apt-get install -y postgresql-9.5 postgresql-contrib-9.5
sudo apt-get install -y postgresql-doc-9.5 postgresql-server-dev-9.5
```
### If Having Issues finding PostgreSQL, do the following:
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt-get update
```

#### Then retry installing PostgreSQL



### Python 3 must be installed along with a few third-party libraries:
```bash
sudo apt-get install -y python3 python3-pip python3-dev virtualenvwrapper
sudo apt-get install -y libxml2-dev libxslt-dev
sudo apt-get install -y libssl-dev libffi-dev
```



### Restart the shell or type bash and press Enter to reload the shell environment with the new virtualenvwrapper variables and functions.

**This step is mandatory before continuing with the deployment!**




### If taiga user not already available, Create a user with root privileges named taiga:
```bash
sudo adduser taiga
sudo adduser taiga sudo
sudo su taiga
cd ~
```

## 3.2. Configuring Dependencies
### Configure PostgreSQL with the initial user and database:
```bash
sudo -u postgres createuser taiga
sudo -u postgres createdb taiga -O taiga --encoding='utf-8' --locale=en_US.utf8 --template=template0
```


### Create a user named taiga, and a virtualhost for RabbitMQ (Optional: taiga-events or async tasks)
```bash
sudo rabbitmqctl add_user taiga PASSWORD_FOR_EVENTS
sudo rabbitmqctl add_vhost taiga
sudo rabbitmqctl set_permissions -p taiga taiga ".*" ".*" ".*"
```

# 4. Backend Setup

### Download the code:
```bash
cd ~
git clone https://github.com/taigaio/taiga-back.git taiga-back
cd taiga-back
git checkout stable
```


### Create a new virtualenv named taiga:
```bash
mkvirtualenv -p /usr/bin/python3 taiga
```


### Install all Python dependencies:
```bash
pip install -r requirements.txt
```

### Execute all migrations to populate the database with basic necessary initial data:
```bash
python manage.py migrate --noinput
python manage.py loaddata initial_user
python manage.py loaddata initial_project_templates
python manage.py compilemessages
python manage.py collectstatic --noinput
```

#### The above migrations create an administrator account. The login credentials are the following:
```bash
username: admin
password: 123123
```
**Attention! Change the administrator account password to a strong password before enabling public access to your Taiga instance. If you miss this action, unauthorized persons can log in to your Taiga instance equipped with administrator privileges.**

**OPTIONAL: If you would like to have some example data loaded into Taiga, execute the following command to populate the database with sample projects and random data (useful for demos):**
```bash
python manage.py sample_data
```



### To finish the setup of taiga-back, create the initial configuration file for proper static/media file resolution, optionally with email sending support:
#### Copy-paste the following config into ~/taiga-back/settings/local.py and update it with your own details:
```python
from .common import *

MEDIA_URL = "http://pm.adplexus.com/media/"
STATIC_URL = "http://pm.adplexus.com/static/"
SITES["front"]["scheme"] = "http"
SITES["front"]["domain"] = "pm.adplexus.com"

SECRET_KEY = "k3E9$2x&ZB3jPTM#1D9s5&HPb"

DEBUG = False
PUBLIC_REGISTER_ENABLED = True

DEFAULT_FROM_EMAIL = "no-reply@adplexus.com"
SERVER_EMAIL = DEFAULT_FROM_EMAIL

#CELERY_ENABLED = True

EVENTS_PUSH_BACKEND = "taiga.events.backends.rabbitmq.EventsPushBackend"
EVENTS_PUSH_BACKEND_OPTIONS = {"url": "amqp://taiga:PASSWORD_FOR_EVENTS@localhost:5672/taiga"}

# Uncomment and populate with proper connection parameters
# to enable email sending. `EMAIL_HOST_USER` should end by @<domain>.<tld>
#EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
#EMAIL_USE_TLS = False
#EMAIL_HOST = "localhost"
#EMAIL_HOST_USER = ""
#EMAIL_HOST_PASSWORD = ""
#EMAIL_PORT = 25

# Uncomment and populate with proper connection parameters
# to enable GitHub login/sign-in.
#GITHUB_API_CLIENT_ID = "yourgithubclientid"
#GITHUB_API_CLIENT_SECRET = "yourgithubclientsecret"
```

## 4.1. Verification

(Optional step) To make sure that everything works, execute the following commands to run the backend in development mode for a quick test:
```bash
workon taiga
python manage.py runserver
```

**Open your browser at http://localhost:8000/api/v1/.**

If your configuration is correct, you will see a JSON representation of REST API endpoints.

At this stage, the backend has been deployed successfully, but to run the Python backend in production, a WSGI server must be configured first. The WSGI server configuration is explained later in this documentation.
