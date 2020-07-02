# Mattermost installation on CentOS 7

From a perspective in the years 2020 and onwards, we could say CentOS 7 is providing quite an old technical stack which means all the hardened features that have been developed since that OS version has been out are not available easily or even not available at all.

This guide aims to bring an installation as stable as possible with some compromises taken on the long term support mantra this OS tries to provide in order to bring some advanced features and the latest version of Mattermost available.

## Installation

First thing first, let's add the Extra Packages for Enterprise Linux repo:
```
# yum -y install epel-release
```

By default, that package should already be installed:
```
[...]
Package epel-release-7-12.noarch already installed and latest version
Nothing to do
```

We will be using adding [a well maintained community maintained Mattermost repository](https://gitlab.com/harbottle/harbottle-main/-/tree/master/). The latter is not as hardened as the one provided on Arch Linux due to unmet dependency preriquisites, but it is perfectly suitable for a production environment.

Just confirm the install process:
```
# yum -y install https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm
```

## Database setup

The MariaDB version provided with CentOS 7 (MariaDB 5.5) is a bit outdated. It seems it doesn't support full index on InnoDB, we had the following error message: "The used table type doesn't support FULLTEXT indexes". We [switched](https://mariadb.com/resources/blog/installing-mariadb-10-on-centos-7-rhel-7/) to the upstream MariaDB version (10.x).
```
# curl -LOC - https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
# chmod +x mariadb_repo_setup
# ./mariadb_repo_setup
# yum install MariaDB-server
```

Start the MariaDB service to perform the initial secure installation of MariaDB. Just follow the on-screen instructions.
```
# systemctl start mariadb.service
# mysql_secure_installation
```

Connect as root to the database, create the database, a dedicated user for Mattermost and grant privileges to it.
```
$ mysql -u root -p
```

Inside the MariaDB shell:
```
CREATE DATABASE mattermostdb;
CREATE USER mmuser IDENTIFIED BY 'mmuser_password';
GRANT ALL ON mattermostdb.* TO mmuser;
```

Ensure the MariaDB server is running at boot:
```
# systemctl enable mariadb.service
```

## Configuring Mattermost

The Mattermost configuration happens in the file `/etc/mattermost/config.json`.

Let's specify the correct `DriverName` and `DataSource` .
```
[...]
"SqlSettings": {
    "DriverName": "mysql",
    "DataSource": "<database_username>:<database_password>@tcp(localhost:3306)/<database_name>?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
[...]
```

Let's change the default language to French:

```
[...]
"LocalizationSettings": {
    "DefaultServerLocale": "fr",
    "DefaultClientLocale": "fr",
    "AvailableLocales": ""
},
[...]
```

Start and enable the Mattermost service:
```
# systemctl start mattermost
# systemctl enable mattermost
```

The service is by default listening on all interfaces only in IPv6. This means configuring a firewall will be needed, otherwise, Mattermost will still be reachable on :8065 to the public, something we don't want. To confirm the situation, we can check it with `ss`:
```
$ ss -tuanp | grep LISTEN | grep 8065
tcp    LISTEN     0      128    [::]:8065               [::]:*                   users:(("mattermost",pid=1115,fd=19))
```

To check out if Mattermost is answering without any issue, we can use `curl`:
```
$ curl -g 'http://[::1]:8065'
```

## Nginx reverse proxy with SSL frontend

### Nginx http installation

Let's install nginx:
```
# yum install nginx.x86_64
```

Contrary to more recent versions of CentOS, virtual hosts are configured in `/etc/nginx/conf.d/` and not using the apache2 inspired directory structure (`/etc/nginx/sites-available/` and `/etc/nginx/sites-enabled/`). If needed, these folders can be created easily and sourced from `/etc/nginx/nginx.conf`. Right now, the location of these virtual hosts is defined by this statement:
```
include /etc/nginx/conf.d/*.conf;
```

Create a file `/etc/nginx/conf.d/mattermost.conf` with the following content:
```
upstream backend {
   server [::1]:8065;
   keepalive 32;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mattermost_cache:10m max_size=3g inactive=120m use_temp_path=off;

server {
   listen 80;
   server_name <your_subdomain.example.com>;

   location ~ /api/v[0-9]+/(users/)?websocket$ {
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       client_max_body_size 50M;
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       client_body_timeout 60;
       send_timeout 300;
       lingering_timeout 5;
       proxy_connect_timeout 90;
       proxy_send_timeout 300;
       proxy_read_timeout 90s;
       proxy_pass http://backend;
   }

   location / {
       client_max_body_size 50M;
       proxy_set_header Connection "";
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       proxy_read_timeout 600s;
       proxy_cache mattermost_cache;
       proxy_cache_revalidate on;
       proxy_cache_min_uses 2;
       proxy_cache_use_stale timeout;
       proxy_cache_lock on;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }
}
```

Start ant enable nginx:
```
# systemctl start nginx
# systemctl enable nginx
```

Please make sure you are aware of the [firewalld basic concepts](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-firewalld-on-centos-7#basic-concepts-in-firewalld) first.

Check the zone which is currently defined and adapt the command hereafter accordingly:
```
# firewall-cmd --get-default-zone
public
```

Authorise http and https:
```
# firewall-cmd --zone=public --permanent --add-service=http --add-service=https
```

Check if Mattermost is reachable from the outside using your web browser:
```
http://your_subdomain.example.com
```


### Enable https

Install certbot and the nginx plugin for certbot
```
# yum install certbot certbot-nginx
```

In order to avoid rate limits, let's do a test first (materialized by the `--dry-run` parameter).
```
# certbot --nginx certonly --dry-run
```

Cerbot should detect your subdomain automatically and ask you to specify an email address where to send reminders in the event the certificate is expiring soon:
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx
Starting new HTTPS connection (1): acme-staging-v02.api.letsencrypt.org

Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: your_subdomain.example.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
```
If everything works smoothly, repeat that step without the `--dry-run` parameter. If you receive an output similar to the following, this means the certificate has been flawlessly issued:
```
Renewing an existing certificate
Performing the following challenges:
http-01 challenge for your_subdomain.example.com
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/your_subdomain.example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/your_subdomain.example.com/privkey.pem
   Your cert will expire on 2020-09-21. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Replace the `mattermost.conf` nginx file as follow:
```
upstream backend {
   server [::1]:8065;
   keepalive 32;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mattermost_cache:10m max_size=3g inactive=120m use_temp_path=off;


server {
   listen 80 http2 default_server;
   listen [::]:80 http2;
   server_name your_subdomain.example.com;
   return 301 https://$server_name$request_uri;
}

server {
   listen 443 ssl http2;
   listen [::]:443 ssl http2;
   server_name your_subdomain.example.com;

   ssl_certificate /etc/letsencrypt/live/your_subdomain.example.com/fullchain.pem;
   ssl_certificate_key /etc/letsencrypt/live/your_subdomain.example.com/privkey.pem;
   ssl_session_timeout 1d;
   ssl_protocols TLSv1.2;
   ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
   ssl_prefer_server_ciphers on;
   ssl_session_cache shared:SSL:50m;
   # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
   add_header Strict-Transport-Security max-age=15768000;
   # OCSP Stapling ---
   # fetch OCSP records from URL in ssl_certificate and cache them
   ssl_stapling on;
   ssl_stapling_verify on;

   location ~ /api/v[0-9]+/(users/)?websocket$ {
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       client_max_body_size 50M;
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       client_body_timeout 60;
       send_timeout 300;
       lingering_timeout 5;
       proxy_connect_timeout 90;
       proxy_send_timeout 300;
       proxy_read_timeout 90s;
       proxy_pass http://backend;
   }

   location / {
       client_max_body_size 50M;
       proxy_set_header Connection "";
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       proxy_read_timeout 600s;
       proxy_cache mattermost_cache;
       proxy_cache_revalidate on;
       proxy_cache_min_uses 2;
       proxy_cache_use_stale timeout;
       proxy_cache_lock on;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }
}
```

Note: Reusing the `$server_name` variable for the directory of the certificate was not working. Using the full path in plain text is needed.

Check whether the configuration works with:
```
# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If everything is okay, reload nginx:
```
# systemctl reload nginx
```

Check if Mattermost is reachable in https and ensure that the access via the port `:8065` is being denied.

Check the https connection robustness on [SSL Labs](https://www.ssllabs.com/ssltest/), but don't forget to check the following checkbox on the webpage to avoid the website to be promoted to the recently checked section of SSL Labs.
```
[x] Do not show the results on the boards
```

Setup [auto renewal](https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-centos-7#step-4-%E2%80%94-setting-up-auto-renewal) of the certificate using a crontab:
```
# crontab -e
0 0,12 * * * python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew
```

Note: The random statement we defined in the cron is to avoid the task from running at exactly midday or midnight. Quite useful when the cloud provider is doing backups, to avoid an I/O increase which could make the SSL renewal fail.

## Specific Mattermost configuration

* `System Console` > `Integrations` > `GIF (Beta)` > `Enable GIF Picker`: `true`
* `System Console` > `Site Configuration` > `Posts` > `Enable Link Previews`: `true`
* `System Console` > `Environment` > `SMTP` > configure all the needed fields.

## Plugins to consider

* [Autolink](https://github.com/mattermost/mattermost-plugin-autolink#configuration-management)
* [Collabora Mattermost](https://www.collaboraoffice.com/integrations/mattermost-plugin/)