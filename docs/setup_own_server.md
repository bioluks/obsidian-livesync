# Setup a CouchDB server

## Table of Contents
- [Configure](#configure)
- [Run](#run)
  - [Docker CLI](#docker-cli)
  - [Docker Compose](#docker-compose)
- [Access from a mobile device](#access-from-a-mobile-device)
  - [Testing from a mobile](#testing-from-a-mobile)
  - [Setting up your domain](#setting-up-your-domain)
- [Reverse Proxies](#reverse-proxies)
  - [Traefik](#traefik)
---

## Configure

The easiest way to set up a CouchDB instance is using the official [docker image](https://hub.docker.com/_/couchdb).

Some initial configuration is required. Create a `local.ini` to use Self-hosted LiveSync as follows ([CouchDB has to be version 3.2 or higher](https://docs.couchdb.org/en/latest/config/http.html#chttpd/enable_cors), if lower `enable_cors = true` has to be under section `[httpd]` ):

```ini
[couchdb]
single_node=true
max_document_size = 50000000

[chttpd]
require_valid_user = true
max_http_request_size = 4294967296
enable_cors = true

[chttpd_auth]
require_valid_user = true
authentication_redirect = /_utils/session.html

[httpd]
WWW-Authenticate = Basic realm="couchdb"
bind_address = 0.0.0.0

[cors]
origins = app://obsidian.md, capacitor://localhost, http://localhost
credentials = true
headers = accept, authorization, content-type, origin, referer
methods = GET,PUT,POST,HEAD,DELETE
max_age = 3600
```

## Run

### Docker CLI

You can launch CouchDB using your `local.ini` like this:
```
$ docker run --rm -it -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=password -v /path/to/local.ini:/opt/couchdb/etc/local.ini -p 5984:5984 couchdb
```
*Remember to replace the path with the path to your local.ini*

Run in detached mode:
```
$ docker run -d --restart always -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=password -v /path/to/local.ini:/opt/couchdb/etc/local.ini -p 5984:5984 couchdb
```
*Remember to replace the path with the path to your local.ini*

### Docker Compose
Create a directory, place your `local.ini` within it, and create a `docker-compose.yml` alongside it. Make sure to have write permissions for `local.ini` and the about to be created `data` folder after the container start. The directory structure should look similar to this:
```
obsidian-livesync
├── docker-compose.yml
└── local.ini
```

A good place to start for `docker-compose.yml`:
```yaml
version: "2.1"
services:
  couchdb:
    image: couchdb
    container_name: obsidian-livesync
    user: 1000:1000
    environment:
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=password
    volumes:
      - ./data:/opt/couchdb/data
      - ./local.ini:/opt/couchdb/etc/local.ini
    ports:
      - 5984:5984
    restart: unless-stopped
```

And finally launch the container
```
# -d will launch detached so the container runs in background
docker compose up -d
```

## Access from a mobile device
If you want to access Self-hosted LiveSync from mobile devices, you need a valid SSL certificate.

### Testing from a mobile
In the testing phase, [localhost.run](https://localhost.run/) or something like services is very useful.

example using localhost.run:
```
$ ssh -R 80:localhost:5984 nokey@localhost.run
Warning: Permanently added the RSA host key for IP address '35.171.254.69' to the list of known hosts.

===============================================================================
Welcome to localhost.run!

Follow your favourite reverse tunnel at [https://twitter.com/localhost_run].

**You need a SSH key to access this service.**
If you get a permission denied follow Gitlab's most excellent howto:
https://docs.gitlab.com/ee/ssh/
*Only rsa and ed25519 keys are supported*

To set up and manage custom domains go to https://admin.localhost.run/

More details on custom domains (and how to enable subdomains of your custom
domain) at https://localhost.run/docs/custom-domains

To explore using localhost.run visit the documentation site:
https://localhost.run/docs/

===============================================================================


** your connection id is xxxxxxxxxxxxxxxxxxxxxxxxxxxx, please mention it if you send me a message about an issue. **

xxxxxxxx.localhost.run tunneled with tls termination, https://xxxxxxxx.localhost.run
Connection to localhost.run closed by remote host.
Connection to localhost.run closed.
```

https://xxxxxxxx.localhost.run is the temporary server address.

### Setting up your domain

Set the A record of your domain to point to your server, and host reverse proxy as you like.  
Note: Mounting CouchDB on the top directory is not recommended.  
Using Caddy is a handy way to serve the server with SSL automatically.

I have published [docker-compose.yml and ini files](https://github.com/vrtmrz/self-hosted-livesync-server) that launch Caddy and CouchDB at once. If you are using Traefik you can check the [Reverse Proxies](#reverse-proxies) section below.

And, be sure to check the server log and be careful of malicious access.


## Reverse Proxies

### Traefik

If you are using Traefik, this [docker-compose.yml](https://github.com/vrtmrz/obsidian-livesync/blob/main/docker-compose.traefik.yml) file (also pasted below) has all the right CORS parameters set. It assumes you have an external network called `proxy`.

```yaml
version: "2.1"
services:
  couchdb:
    image: couchdb:latest
    container_name: obsidian-livesync
    user: 1000:1000
    environment:
      - COUCHDB_USER=username
      - COUCHDB_PASSWORD=password
    volumes:
      - ./data:/opt/couchdb/data
      - ./local.ini:/opt/couchdb/etc/local.ini
    # Ports not needed when already passed to Traefik
    #ports:
    #  - 5984:5984
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      # The Traefik Network
      - "traefik.docker.network=proxy"
      # Don't forget to replace 'obsidian-livesync.example.org' with your own domain
      - "traefik.http.routers.obsidian-livesync.rule=Host(`obsidian-livesync.example.org`)"
      # The 'websecure' entryPoint is basically your HTTPS entrypoint. Check the next code snippet if you are encountering problems only; you probably have a working traefik configuration if this is not your first container you are reverse proxying.
      - "traefik.http.routers.obsidian-livesync.entrypoints=websecure"
      - "traefik.http.routers.obsidian-livesync.service=obsidian-livesync"
      - "traefik.http.services.obsidian-livesync.loadbalancer.server.port=5984"
      - "traefik.http.routers.obsidian-livesync.tls=true"
      # Replace the string 'letsencrypt' with your own certificate resolver
      - "traefik.http.routers.obsidian-livesync.tls.certresolver=letsencrypt"
      - "traefik.http.routers.obsidian-livesync.middlewares=obsidiancors"
      # The part needed for CORS to work on Traefik 2.x starts here
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolallowmethods=GET,PUT,POST,HEAD,DELETE"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolallowheaders=accept,authorization,content-type,origin,referer"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolalloworiginlist=app://obsidian.md,capacitor://localhost,http://localhost"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolmaxage=3600"
      - "traefik.http.middlewares.obsidiancors.headers.addvaryheader=true"
      - "traefik.http.middlewares.obsidiancors.headers.accessControlAllowCredentials=true"

networks:
  proxy:
    external: true
```

Partial `traefik.yml` config file mentioned in above:
```yml
...

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: "websecure"
          scheme: "https"
  websecure:
    address: ":443"

...
```