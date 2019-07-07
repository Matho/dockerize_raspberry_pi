# How to run Rails apps with Docker on Raspberry Pi 4

Read time: 30min 
* Disclaimer: Althought this tutorial is quite detailed, you need to understand, what you are writing in terminal. Some of the commands
may be omited. I'm very new to Docker, so instructions could be written more clearly.


## Where and what to buy 
On 24.6.2019 I have bought Raspberry Pi 4. It is my first raspberry pi and I wanted to create own server. Own server 
connected to UPS. No hosting in datacentre.

I put order at https://rpishop.cz/, which is official distributor of rpi in Czech and Slovak republic. 
Although there is high demand of rpi, my rpi was delivered in a week, which is fast.

I have bought the 4GB variant of rpi. 
The rpi you can find at https://rpishop.cz/raspberry-pi-4b/1598-raspberry-pi-4-model-b-4gb-ram.html.
You need also order the USB-c cable https://rpishop.cz/raspberry-pi-4b/1595-OFI045.html. I recommend to order also 
rpi cover https://rpishop.cz/krabicky/1611-zonepi-krabicka-pro-raspberry-pi-4b-galaxy.html , the HDMI reduction https://rpishop.cz/kabely-a-konektory/1608-hdmi-microhdmi-redukce-cerna-15cm.html
and the passive cooler https://rpishop.cz/chlazeni/294-chladic-pro-raspberry-pi.html .

If you don't want to damage SD card when your rpi will lost power supply, I recommend to buy also UPS (uninterruptible power supply).
I have bought CyberPower BRICS LCD Series BR1000ELCD in Alza - https://www.alza.sk/cyberpower-brics-lcd-series-br1000elcd-d4055352.htm
The big advantage is, that it is able to turn off beep sound, when the rpi is powered on battery. Also I have bought sd card,
64GB https://www.alza.sk/sandisk-microsdxc-64gb-ultra-uhs-i-v30-u3-sd-adapter-d5498575.htm  Now you have everything to start.

I recommend to start with downloading NOOBS. Noobs (https://www.raspberrypi.org/downloads/noobs/) is ' New Out Of the Box Software'. It is 
easy installer of rpi OS. Click download Zip and download the installer. Unzip it to your SD card, to the root.
Before it, format your SD card with FAT system. Don't afraid, rpi will not use fat system, the fat system is required
only for install process. Put the card to rpi and start it. Don't forget to put also ethernet cable. On install welcome screen,
select Raspbian Full (recommended) system to install. Click install and follow installation screen. 

If you passed OS installation, you could continue with apps installation. Connect monitor to hdmi port. 
Note: Currently there is bug (or unfinished drivers) and the HW acceleration in Chromium do not work. Chromium runs
only on sw acceleration for now, which is quite slow. So the video playback in Chromium is very slow. See https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=244398&p=1490752&hilit=video+slow#p1490752 for 
more info. 

## Firewall & ssh login

We will continue with firewall setttings. Run \
`sudo apt-get install ufw` \

Deny all incoming traffic \
`sudo ufw default deny incoming`

Allow required ports \
for ssh: `sudo ufw allow 7777` \
for monit: `sudo ufw allow 2812` \
for docker gitlab registry: `sudo ufw allow 5005` \
for https: `sudo ufw allow 443` \

Start ssh daemon after restart: \
`sudo systemctl enable ssh.service`

and start it for current session: \
`sudo systemctl start ssh.service `

You should be able to login to your pi: \
`ssh pi@10.0.2.2` 

If you don't know your pi IP, write `ifconfig` in pi's terminal and search for eth0.

Copy your (your notebook you are using) ssh key to clipboard and set to pi's authorized keys. 
This will allow you to password-less login via ssh keys: \
`mkdir ~/.ssh` \
`vim.tiny ~/.ssh/authorized_keys` \

On your notebook, copy output of `cat ~/.ssh/id_rsa.pub` and pass it to rpi's authorized_keys. 

Use default 22 port is not secure, so we will change port to 7777. Open: `vim.tiny /etc/ssh/sshd_config` 
and change config to `PasswordAuthentication no`. At the begining of file change port to `Port 7777`. 
Now restart ssh to take changes `sudo systemctl restart ssh.service`. Logout from your terminal session (or better to create new one)
via `ssh pi@10.0.2.2 -p 7777`. You should be logged in.

## Swap configuration
It is good to allow some swap file. Go to: \
`sudo vim.tiny /etc/dphys-swapfile`

and set this configs: (alternatively use your own, this is for 4 GB) \
`CONF_SWAPSIZE=4096`    
`CONF_MAXSWAP=4096`

Do restart of service: \
`sudo /etc/init.d/dphys-swapfile restart`

Run top command and see the swap settigs: \
`top`

## Router configs
I have my rpi behind 2 routers. I recommend to set static IP (for wan) for the second router, to have after
restart with the one ip. Also I recommend to setup port forwarding in the first and second router.
The ports you will need to have forwarded in both routers: \
`443`  
`5000`   
`22`  
`5005`  
`80`  
`2812`  
`7777`

Also change the remote administration port for first router to some another port, do not listen on port 80.
Also set DHCP start ip in both of routers to some bigger values, to do not try to assign the statically setup ips.

## Static IP config for eth0

Open: \
`sudo vim.tiny /etc/dhcpcd.conf`

and set the eth0 config: 
>interface eth0 
static ip_address=10.0.2.2/24  
static routers=10.0.2.1  
static domain_name_servers=217.119.121.225 8.8.8.8

Set up also ipv6 address. 

## Postgres installation
We will use Postgres for our rails apps. We will not include postgres in our docker containers, we will share
Postgres with our rails apps.

To install, do: \
> sudo apt install postgresql libpq-dev postgresql-client postgresql-client-common
sudo pg_ctlcluster 11 main start  
sudo systemctl enable postgresql@11-main.service  
sudo systemctl enable postgresql.service  


To create pi user: \
> createuser pi -P --interactive

Edit listen config: \
> sudo vim.tiny /etc/postgresql/11/main/postgresql.conf

Change to:
> listen_addresses = '*'

Open pg_hba: \
> sudo vim.tiny /etc/postgresql/11/main/pg_hba.conf

add: \
> host    all     all     0.0.0.0/0       md5

If you want to connect to postgres from outside the server, open the 5432 port in ufw. The better use case
is to create ssh tuneling to given port. If you want to use pgadmin3 to connect to remote db, it will fail, 
because our db is version 11. Use alternative tool, like Datagrip from JetBrains. 

  
## Monit monitoring
Monit is server monitoring tool. It can ping your servers or monitor processed (like databases), filesystems ...

To install monit:
> sudo apt-get install monit  
sudo systemctl enable monit  
sudo ufw allow 2812  
sudo vim.tiny /etc/monit/monitrc  

In this file set directives to: \
>  set httpd port 2812 and  
 allow admin:password   # login credentials to <your-ip>:2812

It is up to you to set monitoring of your processes. I recommend to set ping monitoring by editing your monit 
config on another server.

To make change, restart monit: \
> sudo systemctl restart monit.service

## Docker installation
The tutorial for Debian is here - https://docs.docker.com/install/linux/docker-ce/debian/
There is known bugs with installation docker on rpi 4. See https://github.com/docker/for-linux/issues/709 for more info.
There is workaround to install docker with: \
> curl -sL get.docker.com | sed 's/9)/10)/' | sh

Docker postinstall is located at: `https://docs.docker.com/install/linux/linux-postinstall/`
> sudo usermod -aG docker $USER  
logout  
login

Test docker installation via command: \
> docker run hello-world


## Gitlab installation
We need docker registry to host docker images for our rails app(s). Docker hub is limited to only 1 image. So
we will install Gitlab. Note: You do not need to install Gitlab to have only docker registry feature. You can found docker
registry image on docker hub to have non-gui docker registry host.

The gitlab documentation is located at https://docs.gitlab.com/omnibus/docker/

Try to run this docker command: \
> sudo docker run --detach \
--hostname gitlab.matho.sk \
--publish 443:443 --publish 80:80 --publish 22:22 --publish 5005:5005 --publish 5000:5000 \
--name gitlab \
--restart always \
--volume /srv/gitlab/config:/etc/gitlab \
--volume /srv/gitlab/logs:/var/log/gitlab \
--volume /srv/gitlab/data:/var/opt/gitlab \
ulm0/gitlab:latest

Don't forget to point www and non-www domain names to your public IP (set DNS A record). 

You can list containers in system via:
> sudo docker container ls -a

You can stop gitlab container via: \
> sudo docker stop gitlab

You can see logs from container via: \
> sudo docker logs gitlab

You can get bash shell inside container via: \ 
> sudo docker exec -it gitlab bash

Edit gitlab config: \
> sudo docker exec -it gitlab vim.basic /etc/gitlab/gitlab.rb

and locate and replace following configs: \
> external_url 'https://gitlab.matho.sk/' \
gitlab_rails['gitlab_email_enabled'] = true  
gitlab_rails['gitlab_email_from'] = 'noreply@matho.sk'  
gitlab_rails['gitlab_email_display_name'] = 'gitlab.matho.sk'  
gitlab_rails['gitlab_email_reply_to'] = 'martin.markech@matho.sk'  
gitlab_rails['gitlab_email_subject_suffix'] = ''
gitlab_rails['incoming_email_enabled'] = false  
gitlab_rails['gitlab_shell_ssh_port'] = 22  
gitlab_rails['smtp_enable'] = true  
gitlab_rails['smtp_address'] = "smtp.websupport.sk"  
gitlab_rails['smtp_port'] = 25  
gitlab_rails['smtp_user_name'] = "no-reply@matho.sk"  
gitlab_rails['smtp_password'] = "password"  
gitlab_rails['smtp_domain'] = "matho.sk"  
gitlab_rails['smtp_authentication'] = "login"  
gitlab_rails['smtp_enable_starttls_auto'] = true  
registry_external_url 'https://registry.gitlab.matho.sk:5005'   
gitlab_rails['registry_enabled'] = true  
gitlab_rails['registry_host'] = "registry.gitlab.matho.sk"  
gitlab_rails['registry_port'] = "5005"  
gitlab_rails['registry_path'] = "/var/opt/gitlab/gitlab-rails/shared/registry"   
letsencrypt['enable'] = true   
letsencrypt['contact_emails'] = ['matomarkech@gmail.com']  

This config will ensure, that gitlab cloning will be do via 22 port, ssh loging via 7777 port. Docker registry
will run on 5005 port. 

Wait 5 minutes while the certificates will generate. 
When you have edited the gitlab config, restart docker container via
`sudo docker restart gitlab`

If it starts correctly, you should see gitlab.matho.sk running. Give it some time to start (5 minutes). 

## Gitlab through nginx_proxy
We plan to host multiple docker containers. E.g: one for nginx_proxy, the second for gitlab and third for gymplaci rails app.
To be able run multiple apps on 80 port, we start nginx_proxy, and do proxy for 80 port. For more info
see https://hub.docker.com/r/jwilder/nginx-proxy/. The problem is, this image is not ready for raspberry pi. It didn't
work for me. So I tried to use fork of library with supports for rpi:

Stop gitlab container  
> sudo docker stop gitlab

>docker run -d -p 80:80 -p 443:443 \
--name nginx_proxy \
-v /var/run/docker.sock:/tmp/docker.sock:ro \
-v /srv/gitlab/config/ssl:/etc/nginx/certs \
budry/jwilder-nginx-proxy-arm	

Run new gitlab container: \
> sudo docker run --detach \
--hostname gitlab.matho.sk \
--publish 443:443 --publish 80:80 --publish 22:22 --publish 5005:5005 --publish 5000:5000 \
--name gitlab2 \
--restart always \
--volume /srv/gitlab/config:/etc/gitlab \
--volume /srv/gitlab/logs:/var/log/gitlab \
--volume /srv/gitlab/data:/var/opt/gitlab \
-e VIRTUAL_HOST=gitlab.matho.sk,registry.gitlab.matho.sk \
-e VIRTUAL_PROTO=https \
-e VIRTUAL_PORT=443 \
ulm0/gitlab:latest

Note the new -e flags, VIRTUAL_HOST, VIRTUAL_PROTO, VIRTUAL_PORT. These flags are required for nginx_proxy. Based
on that, nginx_proxy generates nginx conf with proxy pass to given hosts.
Open `https://gitlab.matho.sk` and the gitlab should be started. Note: gitlab starting took about 4 minutes on my rpi.
Give it some time to start. It will show 502 error page when it is loading.

Is the host runnig? You can go next. If it failed to start, check the nginx_proxy logs, or gitlab logs.
> sudo docker logs gitlab2

## Gitlab docker registry
When you create gitlab project, you can see the Registry menu item in project. It means, that you can
push docker images to gitlab. It is docker image repository. 

Go to your gitlab settings page, and in menu select access tokens. Create personal access token. Select read_repository, write_repository
access. And try to login to docker registry: (replace personal_access_token with your value)
> docker login registry.gitlab.matho.sk:5005 -u root -p personal_access_token

If it fail, try to check another options in personal access token page. If this command passed, you are able to push 
docker images to registry


## Prepare your rails app for docker
I have two rails apps, which I'm going to host on my raspberry pi 4. It is Refinery CMS - based projects running on 
ruby 2.2.2. Yes, it is quite old ruby version, but I didn't find time to migrate it to new ruby/rails yet.

First, create new project in Gitlab for this rails app. We will push  the rails project to this repository and we 
will add new remote for this project.

### Dockerize gymplaci
At first, I recommend to move your credentials hardcoded in your rails app to env variables. So add gem to your rails Gemfile:
> gem 'dotenv-rails'

Run bundle
> bundle install

Now, you can create env files like
> .env.development  
.env.production

and store the env variable inside it. Do not commit it to project, it should stay secret.

Add `.dockerignore` file in root of your rails project:
> Dockerfile  
.byebug_history  
.rspec  
README.md  
bin/build.docker.sh  
bin/deploy.docker.sh  
doc/*  
log/*  
tmp/*  
.git/*  
.idea/*  
coverage/*  
public/uploads/*  
public/system/*  
docker-compose.*  
node_modules/*  
!.env.production  
!.env.example

Add to `.gitignore` file:
> .env.*

Add `Dockerfile` file in root:
```
FROM hypriot/rpi-ruby:2.2.2   

MAINTAINER Matho "martin.markech@matho.sk" \  

RUN apt-get update && apt-get install -y \  
    curl \
    vim \
    git \
    build-essential \
    libgmp-dev \
    libpq-dev \
    postgresql-client \
    locales \
    nginx \
    cron \
    bash \
    imagemagick

WORKDIR /app  

ARG BUNDLE_CODE__MATHO__SK
ARG RAILS_ENV

ADD ./Gemfile ./Gemfile
ADD ./Gemfile.lock ./Gemfile.lock

RUN bundle install --deployment --clean --path vendor/bundle --without development test

ADD . .

ADD config/etc/nginx/conf.d/nginx.docker.conf /etc/nginx/conf.d/default.conf
RUN rm /etc/nginx/sites-enabled/default

ADD .env.development .env.development
ADD .env.production .env.production
RUN bundle exec rake assets:precompile

EXPOSE 80

CMD bin/run.docker.sh
```

Note: The `hypriot/rpi-ruby:2.2.2` is required only because of my 2.2.2 ruby for rpi. If you need to use
another version of ruby, find some project with given ruby for rpi. 

Add `bin/run.docker.sh` file:
```
#!/bin/bash
set -x
set -e
set -o pipefail

./bin/run.symlinks.docker.sh

bundle exec rake db:migrate

service nginx start

bundle exec puma -C config/puma.rb
```

Ensure, you have puma gem also in production gem group, or better said, you don't have it available only for development env.

Add file `bin/run.symlinks.docker.sh` : 
```
#!/bin/bash
  
# Create symlinks (use absolute paths)  
for folder in 'tmp/cache' 'log' 'public/uploads' 'public/system'; do  
  rm -rf "/app/$folder"  
  mkdir -p "/app/shared/$folder"  
  ln -sf "/app/shared/$folder" "/app/$folder"  
done  

mkdir -p /app/shared/nginx/cache/dragonfly
```

Modify `config/database.yml` to: 
```
 default: &default  
    adapter: postgresql  
    encoding: unicode  
    pool: <%= ENV['RAILS_MAX_THREADS'].to_i * 30 %>  
    host: <%= ENV.fetch("POSTGRES_HOST") { 'postgres' } %>  
    database: <%= ENV.fetch("POSTGRES_DB") { 'db' } %>  
    username: <%= p ENV.fetch("POSTGRES_USER"); ENV.fetch("POSTGRES_USER") { 'postgres' } %>  
    password: <%= ENV.fetch("POSTGRES_PASSWORD") { 'postgres' } %>  
    port: <%= ENV.fetch("POSTGRES_PORT") { 5432 } %>  
  
  development:  
    <<: *default  
  
  staging:  
    <<: *default  
  
  test:  
    <<: *default  
    database: <%= ENV.fetch("POSTGRES_TEST_DB") { 'db_test' } %><%= ENV['TEST_ENV_NUMBER'] %>  
    
  production:  
    <<: *default 
```

Add file `config/etc/nginx/conf.d/nginx.docker.conf` : 
```
upstream app {
  server unix:/app/puma.sock fail_timeout=0;
}

proxy_cache_path /app/shared/nginx/cache/dragonfly levels=2:2 keys_zone=dragonfly:100m inactive=30d max_size=1g;
server {
  listen 80 default_server;
  root /app/public;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
    add_header Vary Accept-Encoding;
  }

  try_files $uri/index.html $uri $uri.html @app;
  location @app {
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $server_name;
    proxy_pass_request_headers      on;

    proxy_redirect off;
    proxy_pass http://app;

    proxy_connect_timeout       1800;
    proxy_send_timeout          1800;
    proxy_read_timeout          1800;
    send_timeout                1800;

    proxy_buffer_size   128k;
    proxy_buffers   4 256k;
    proxy_busy_buffers_size   256k;

    gzip             on;
    gzip_min_length  1000;
    gzip_proxied     expired no-cache no-store private auth;
    gzip_types       text/plain text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_disable     "MSIE [1-6]\.";
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;

  client_body_timeout 12;
  client_header_timeout 12;
  keepalive_timeout 20;
  send_timeout 10;

  client_body_buffer_size 10K;
  client_header_buffer_size 1k;
  large_client_header_buffers 4 32k;

  server_tokens off;
}
```

Add `config/puma.rb` 
```
workers ENV.fetch("RAILS_WORKERS") { 2 }

threads_min = ENV.fetch("RAILS_MIN_THREADS") { 5 }
threads_max = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_min, threads_max

# Specifies the `environment` that Puma will run in
environment ENV.fetch("RAILS_ENV") { "development" }

# Executed in Docker container
if File.exists?('/.dockerenv')
  app_dir = File.expand_path("../..", __FILE__)

  stdout_redirect "#{app_dir}/log/puma.stdout.log", "#{app_dir}/log/puma.stderr.log", true

  bind "unix://#{app_dir}/puma.sock"
else
  # Listen only on port
  port ENV.fetch("PORT") { 3000 }
end

# Allow puma to be restarted by `rails restart` command.
plugin :tmp_restart
```

Add `docker-compose.production.yml` 
```
version: "2"
services:
  app:
    image: "registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest"
    restart: always
    env_file: .env.production
    ports:
      - "8082:80"
    network_mode: bridge
    volumes:
      - ./shared:/app/shared
    tty: true
    environment:
      - HOSTNAME=gymplaci.sk
      - VIRTUAL_HOST=gymplaci.sk,www.gymplaci.sk
      - RAILS_ENV=production

```
To make the project installed on rpi, I needed to make following changes 
* migrate to newer json gem: `gem 'json', '>= 1.8.6'`
* migrate therubyracer to newer version: `gem 'therubyracer', '0.12.0', :platforms => :ruby`
* move libv version to `gem 'libv8', '3.16.14.19'`
* `bundle update json`
* `bundle install`
* I had problems with asset precompilation, I remove .css from `config.assets.precompile` 
* push the changes to remote 

It is not possible to build docker image on your ubuntu (developer's system). It is able build docker image
only on rpi host (architecture). So create some temporary folder, like `~/delete/gymplaci.sk`. 

Clone project on rpi: \
`git clone git@gitlab.matho.sk:mathove-projekty/gymplaci.sk.git`

Add secret settings file: `.env.production`
```
SMTP_ADDRESS=smtp.websupport.sk
SMTP_DOMAIN=gymplaci.sk
SMTP_USERNAME=no-reply@gymplaci.sk
SMTP_PASSWORD=

POSTGRES_HOST=localhost
POSTGRES_DB=gymplaci_production
POSTGRES_USER=
POSTGRES_PASSWORD=

RAILS_WORKERS=2
RAILS_MIN_THREADS=5
RAILS_MAX_THREADS=5
```

### Postgres import
To run docker image, you need to have at least created postgres database. We will backup whole database (all database tables) with one command: 
```
pg_dumpall -U postgres -h localhost -p 5432 --file=2019_07_06_pg_cluster.sql
```
Then download generated file via scp tool to your folder on rpi. I recommend to generate ssh keys on rpi host and setup it on your remote to be
able log with ssh keys to your vps server.

When you have downloaded the file , restore the whole pg via:
```
pg_dumpall -U postgres -h localhost -p 5432 --file=2019_07_06_pg_cluster.sql
```

It will ask for password for each importing database. So don't think the inserted password is incorrect, it only asks
each time it imports database.

You have imported db, you can continue in docker image building

### Building docker image

You are ready to build your docker image: 
```
docker build --build-arg RAILS_ENV=production -t registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk .
```
If it pass, you can push the image to repository: 
```
docker login registry.gitlab.matho.sk:5005 -u root -p yourpersonalaccesstoken
docker push registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk
```

Try to run docker image. Note: stop previously started docker image, if it is running.
```
docker run --detach \
 --hostname gymplaci.sk \
 --publish 8082:80 \
 --name gymplaci.sk \
 --restart always \
 --volume /data/gymplaci.sk:/app/shared \
 -e VIRTUAL_HOST=gymplaci.sk,www.gymplaci.sk \
 -e RAILS_ENV=production \
registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest
```

Did it starts correctly? Could you visit `http://gymplaci.sk` in your browser? If yes, continue

## Docker compose
For now, you have started each container manually. When you start it manually, the network mode bridge was applied.
Now we want to create one docker compose file, where all services will be written. All containers you will be
able to start via 1 command.

Create new directory, e.g.:
```
mkdir ~/docker
cd ~/docker
```
Copy paste following file: `docker-compose-server.yml`  \
```
services:
  gymplaci.sk:
    image: "registry.gitlab.matho.sk:5005/mathove-projekty/gymplaci.sk:latest"
    restart: always
    env_file: .env.production.gymplaci
    ports:
      - "8082:80"
    network_mode: bridge
    volumes:
      - /data/gymplaci.sk:/app/shared
    tty: true
    environment:
      - HOSTNAME=gymplaci.sk
      - VIRTUAL_HOST=gymplaci.sk,www.gymplaci.sk
      - RAILS_ENV=production

  gitlab:
    image: "ulm0/gitlab:latest"
    restart: always
    ports:
      - "4431:443"
      - "8081:80"
      - "22:22"
      - "5005:5005"
      - "5000:5000"
    network_mode: bridge
    volumes:
      - /srv/gitlab/config:/etc/gitlab
      - /srv/gitlab/logs:/var/log/gitlab
      - /srv/gitlab/data:/var/opt/gitlab
    tty: true
    environment:
      - HOSTNAME=gitlab.matho.sk
      - VIRTUAL_HOST=gitlab.matho.sk,registry.gitlab.matho.sk
      - VIRTUAL_PROTO=https
      - VIRTUAL_PORT=443

  nginx_proxy:
    image: "budry/jwilder-nginx-proxy-arm"
    restart: always
    ports:
      - "80:80"
      - "443:443"
    network_mode: bridge
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - /srv/gitlab/config/ssl:/etc/nginx/certs
    tty: true
    environment:
      - HOSTNAME=nginx_proxy
    depends_on:
      - gymplaci.sk
      - gitlab

```

Copy `.env.production` file content from your gymplaci project to `.env.production.gymplaci`
This script will allow to run gymplaci project on 8082 port , gitlab on 8081 port ang nginx_proxy will proxy
the calls for the following ports. Note the `network_mode: bridge` instruction. This will ensure, that all 3 containers
will be run into one network. If you start commands manually via docker run, the bridge network mode will be used
automatically. But when you start it via docker compose, the new network will be created. So, do not forget or delete this
instruction.

If you want to print, which containers are in which network, you can use: (instead of \< NETWORK ID \>use network id from ls command) 
```
docker network ls
docker network inspect <NETWORK ID>
``` 

Stop and remove all containers. Don't afraid, your data will be saved outside of your docker containers 
```
sudo docker stop container_name
sudo docker rm container_name
```

Go to your ~/docker folder
```
cd ~/docker
```

Wait! You do not install docker-compose yet! So install it via: 
```
sudo apt-get install docker-compose

```

and start docker compose 
```
sudo docker-compose -f docker-compose-server.yml up -d
```

This command will start all 3 services in detached mode. Also, this containers will auto start on system reboot.

Point to `http://gymplaci.sk` project. It is running, but you do not see any images on website, am I right?
Yes, because you need download the public folder content.

Allow to login to your vps droplet from raspberry pi user. (from rpi `root` user). 
Then go to folder and run image downloading

```
cd /data/gymplaci.sk/public/system/
rsync -avz -e "ssh -p 7777" user@yourserverdomain.eu:/home/user/www/shared/system/ .
```
This will download your images.

And it is all! Enjoy running your own server!
