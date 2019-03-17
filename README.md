# Howto setup gitlab 

An approach to setup a [dockerized gitlab-ce](https://hub.docker.com/r/gitlab/gitlab-ce/) with git, registry and pages accessible via HTTPS. Using [Let's Encrypt](https://letsencrypt.org/) certificates and the [Traefik Proxy](https://traefik.io/). 

## Goal

The goal is:

* to have a gitlab-ce instance running on a single node. 
* to have all built-in services like git, docker registry and gitlab pages accessible via HTTPS, only
* to have all required certifcates issued and renewed automatically 

### Some Remarks

I already tried serveral times to achieve this goal, last time using a wild combination of `HA Proxy`, [nginx-proxy](https://github.com/jwilder/nginx-proxy) and [letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion).
It was complicated and wildcard certificates for gitlab pages were not possible. This time, the Traefik proxy made all the difference! 

Also, many thanks to all the authors of how-tos I visited while compiling this document. Unfortunately, I did not keep a log to mention you as you would deserve.  

## The Approach

To meet our goals, we use the following approach:

* Gitlab runs on docker, using the official [gitlab-ce docker image](https://hub.docker.com/r/gitlab/gitlab-ce/). 
* All gitlab services use HTTP, only accessible from within the host-local docker network. This keeps gitlab configuration as simple as possible.
* Public facing HTTPS termination is handled by a [Traefik Proxy](https://traefik.io/), all requests are forwarded to the appropriate github service, based on domain
* Certificates are coming from [Let's Encrypt](https://letsencrypt.org/), automatically requested and renewed by Traefik
* Everything is put into two `docker-compose.yml` files to keep things flexible

## Prerequisites

To have a chance to succeed, you need the following before you continue:

* Two second-level domains. We use use `foo.com` and `foo-pages.com` here as an example
* The two domains must be managed by [a DNS provider supported by Trafik](https://docs.traefik.io/configuration/acme/#provider). 
* Root access to a host with a fixed IP address and `docker-compose` installed 
* Some experience with DNS, docker, networking and unix and gitlab administration. Unfortunately, I cannot go into all the nifty details here, but this guide should be a good starting point.

## Step by Step Guide

### 1. Setup your DNS

We assume 
* `gitlab.foo.com` will be our gitlab/git server
* `registry.foo.com` will be our docker registry
* `*.foo-pages.com` will host our gitlab page domains

Therefore, you need to setup your DNS A records accordingly: All have to point to your servers IP address. 

### 2. Setup your DNS API

If you have a DNS provider that is mentioned in https://docs.traefik.io/configuration/acme/#provider, you have some API that is secured by a token or other means of authorization. The values to need are also shown in the mentioned list.

Your milage may vary here, and I cannot really help with that. If you have your DNS zones on azure, have a look at https://noobient.com/2018/04/10/free-wildcard-certificates-using-azure-dns-lets/

### 3. Create local docker network

We will use two docker-compose files, because this allows to plug-in other services, e.g. two web servers that host the websites at `foo.com` and `foo-pages.com` 

To have both docker-compose stacks connected, we have to create a dedicated docker network on the host machine. Ours is called `traefik-backend` in this example:


```sh
$> docker create traefik-backend
```
### 3. Deploy Traefik 

#### Docker Compose

Lets have a look at the following `docker-compose.yml`:

```yaml
version: '2'

services:
  traefik:
    image: traefik:alpine
    restart: always
    ports:
      - 80:80
#      - 8080:8080              # traefik web UI, for debugging only!
      - 443:443
    networks:
      - traefik-backend         # set the docker network
    environment:                
      - AZURE_CLIENT_ID         # Define the following values in        
      - AZURE_CLIENT_SECRET     # an '.env' file right beside this
      - AZURE_SUBSCRIPTION_ID   # compose file if you use Azure
      - AZURE_TENANT_ID         # To be adjusted for another provider
      - AZURE_RESOURCE_GROUP
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/etc/traefik/traefik.toml
      - ./acme.json:/acme.json
    container_name: traefik

networks:
  traefik-backend:              # refer to the network
    external: true

```

The AZURE environment variables are expected to be set on the host or using an `.env`. Have a look at https://docs.docker.com/compose/environment-variables/ for details, there are many ways to do it. 

Of course, you can also set values in the docker-compose file directly. Just make sure you do not push them into a public repository :-)

#### Example `.env` file: 

```
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_SUBSCRIPTION_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_RESOURCE_GROUP=xxxxxxxxxxxx
```

#### Traefik Configuration `traefik.toml` 
As you see, we mount a `traefik.toml` file into the docker container. This is were traefik is configured:

```toml
logLevel = "info"
defaultEntryPoints = ["https","http"]

# Enable if you need an access log: 
[accessLog]

# WEB interface of Traefik - it will show web page with overview of frontend and backend configurations
# Should not be used in production. If you enable it, ensure  port 8080 is also exported in the docker-compose.yml

#[web]
#address = ":8080"

# Listen on port 80 and 443. 
# 80 redirects to 443, only https is used
[entryPoints]

  [entryPoints.http]
    address = ":80"

    [entryPoints.http.redirect]
    entryPoint = "https"

  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]

# Use docker backend
[docker]
  endpoint = "unix:///var/run/docker.sock"
  watch = true
  exposedByDefault = false
  network = "traefik-backend" # be to use the correct network here

# Let's Encrypt stuff below this point:
[acme]
  email = "admin@foo.com"
  storage = "/acme.json"
  acmeLogging = true       # Helps ifthere are issues with certificates
  entryPoint = "https"

  [acme.dnsChallenge]
    provider = "azure"     # You have to change this to your DNS provider
    delayBeforeCheck = 0

  [[acme.domains]]
    main = "*.foo-pages.com"
#    sans = ["flowerwcs.com"]     # uncomment if you want to host 
                                  # https://foo-pages.com on this server
  [[acme.domains]]
    main = "gitlab.foo.com"       # again, adjust if you also want to
    sans = ["registry.foo.com"]   # host https://foo.com on this server
```

#### Check if Traefik is running

If you run 

```sh
$> docker-compose up
```

you should see that traefik is starting up and is requesting certifiates for configured domains from Let's Encrypt.

If you access one of you domains, you should be redirected to HTTPS and get a 404 error. In the traefik log, you will see that no backend is configured. That's OK for now.

### 4. Deploy Gitlab

Now to the Gitlab Behemoth....

Currently, I keep everything configured in a single docker-compose file, no external configuration files or environment vars are required. 

All date is stored below `/srv` on the host system. 

```yaml
# GITLAB compose file
version: '3'

services:

  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    restart: always
    expose:
      - 5000 # gitlab port
      - 5001 # registry port
      - 5002 # pages port
    labels:
      # Traefik Magic: Define three backends and linked domains
      traefik.enable: true
      traefik.gitlab.port: 5000
      traefik.gitlab.frontend.rule: 'Host:gitlab.foo.com'
      traefik.registry.port: 5001
      traefik.registry.frontend.rule: 'Host:registry.foo.com'
      traefik.pages.port: 5002
      traefik.pages.frontend.rule: 'HostRegexp: {subdomain:[a-z0-9]+}.foo-pages.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.foo.com'
        registry_external_url 'https://registry.foo.com'
        pages_external_url 'https://foo-pages.com'
        nginx['listen_port'] = 5000
        nginx['listen_https'] = false
        nginx['proxy_set_headers'] = {"X-Forwarded-Proto" => "https","X-Forwarded-Ssl" => "on"}
        gitlab_rails['registry_enabled'] = true
        registry['enable'] = true
        registry_nginx['enable'] = true
        registry_nginx['listen_port'] = 5001
        registry_nginx['listen_https'] = false
        registry_nginx['proxy_set_headers'] = {"X-Forwarded-Proto" => "https","X-Forwarded-Ssl" => "on"}
        gitlab_rails['rack_attack_git_basic_auth'] = {
           'enabled' => false,
        }
        pages_nginx['listen_port'] = 5002
        pages_nginx['listen_https'] = false
        pages_nginx['proxy_set_headers'] = {"X-Forwarded-Proto" => "https","X-Forwarded-Ssl" => "on"}
        gitlab_pages['inplace_chroot'] = true
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "YOUR_SMTP_SERVER"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "YOUR_SMTP_USER"
        gitlab_rails['smtp_password'] = "YOUR_SMTP_PWD"
        gitlab_rails['smtp_domain'] = "YOUR_SMTP_DOMAIN"
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['smtp_openssl_verify_mode'] = 'none'
        gitlab_rails['gitlab_email_from'] = 'gitlab@foo.com'
        gitlab_rails['gitlab_email_reply_to'] = 'noreply@foo.com'
        gitlab_rails['backup_keep_time'] = 604800
        postgresql['shared_buffers'] = "256MB"
        unicorn['worker_processes'] = 4
        prometheus_monitoring['enable'] = false
    networks:
      - traefik-backend
    volumes:
      - '/srv/gitlab/config:/etc/gitlab'
      - '/srv/gitlab/logs:/var/log/gitlab'
      - '/srv/gitlab/data:/var/opt/gitlab'

networks:
  traefik-backend:
    external: true
```

The configuration above also include SMTP settings to gitlab can send notification mails. You can remove everything containing `smtp` from the compose file, or adjust to your environment.

The domains are referenced in two locations in the file:
1. In the traefik related configuration (`labels` section)
1. In the gitlab configuration itself (`GITLAB_OMNIBUS_CONFIG`)

Of course, you have to adjust both to your domains.

When you run Traefik and the gitlab container together, gitlab should become available via your configured domains.

Please be patient, Gitlab needs some time to start up and Trafik seems to need some time to recognize the connected backend. I had to wait quite some time before everything was up and running (less than a minute, I guess)

That's it - have fun! :-)

## Where to go from here

You still need to setup backups. And maybe you don't like the docker socket being mounted into the public traefik container. 

There is surely still room for improvement, please send your comments and ideas (or pull requests) if you would like to contribute.





