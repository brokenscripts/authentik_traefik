# Authentik 2024+ and Traefik 3.x  

**Ensure all `CHANGEME` and `domain.tld` values are changed to match YOUR environment!**  

Important changes: Traefik 2.x write up has been renamed from the `main` branch to `traefik2`.  Traefik 3.x and Authentik 2024.x now reside on the `traefik3` branch, which will be the default branch.  

--- 

# Minimal setup  
After having a discussion with a few folks, some want just a "simple" compose to test this out.  Here you go, no traefik, just a barebones compose.yaml to run as it sits and have a working authentik.  

## Prepare the environment
```bash
## Create all required folders
mkdir -p ./authentik/{redis,postgresql,media,custom-templates}
## Pull all images before running this file
grep "image:" compose.yaml | cut -d":" -f2 | xargs -I {} docker pull "{}"
## Set the value of PUID and PGID to current user if not already set using bash interpolation
export PUID="${PUID:-$(id -u)}"
export PGID="${PGID:-$(id -g)}"
docker compose up -d
```

<details>  

<summary>minimal compose.yaml</summary>  

```yaml
# compose.yaml
# -----------------------
# Preparation
# -----------------------
## Create all required folders
# mkdir -p ./authentik/{redis,postgresql,media,custom-templates}
## Pull all images before running this file
# grep "image:" compose.yaml | cut -d":" -f2 | xargs -I {} docker pull "{}"
## Set the value of PUID and PGID to current user if not already set using bash interpolation
# export PUID="${PUID:-$(id -u)}"
# export PGID="${PGID:-$(id -g)}"
# docker compose up -d


# authentik_server      | {"error":"authentik starting","event":"failed to proxy to backend","level":"warning","logger":"authentik.router","timestamp":"2024-12-15T14:23:08Z"}
#
# navigate to http://{IP}/if/flow/initial-setup/


name: authentik # Project Name

networks:
  authentik-backend:
    name: authentik-backend
  t3_proxy:
    name: t3_proxy
    driver: bridge
    ipam:
      config:
        - subnet: 10.255.224.0/20  # 10.255.224.1 - 10.255.239.254
          # ip_range specifies the "dhcp scope" for containers
          ip_range: 10.255.224.0/21  # 10.255.224.1 - 10.255.231.254
          # stack IPs will be in 10.255.232.x subnet


services:
  authentik_postgresql:
    image: docker.io/library/postgres:16-alpine
    container_name: authentik_postgresql
    shm_size: 128mb
    restart: unless-stopped
    user: ${PUID}:${PGID}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    networks:
      - authentik-backend
    volumes:
      - "./authentik/postgresql:/var/lib/postgresql/data"
    environment:
      - POSTGRES_PASSWORD=postgresql_password
      - POSTGRES_USER=authentik_db_user
      - POSTGRES_DB=authentik

  authentik_redis:
    image: docker.io/library/redis:alpine
    container_name: authentik_redis
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    networks:
      - authentik-backend
    volumes:
      - "./authentik/redis:/data"

  # Use the embedded outpost (2021.8.1+) instead of the separate Forward Auth / Proxy Provider container
  authentik_server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik_server
    restart: unless-stopped
    command: server
    user: ${PUID}:${PGID}
    depends_on:
      - authentik_postgresql
      - authentik_redis
    networks:
      - t3_proxy
      - authentik-backend
    ports:
      - 9000:9000
    environment:
      - AUTHENTIK_REDIS__HOST=authentik_redis
      - AUTHENTIK_POSTGRESQL__HOST=authentik_postgresql
      - AUTHENTIK_POSTGRESQL__NAME=authentik
      - AUTHENTIK_POSTGRESQL__USER=authentik_db_user
      - AUTHENTIK_POSTGRESQL__PASSWORD=postgresql_password
      - AUTHENTIK_LOG_LEVEL=warning
      - AUTHENTIK_SECRET_KEY=authentik_secret_key__authentik_secret_key
    volumes:
      - ./authentik/media:/media"
      - "./authentik/custom-templates:/templates"

  authentik_worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik_worker
    restart: unless-stopped
    # Removing `user: root` also prevents the worker from fixing the permissions
    # on the mounted folders, so when removing this make sure the folders have the correct UID/GID
    # (1000:1000 by default)
    # user: root
    user: ${PUID}:${PGID}
    command: worker
    depends_on:
      - authentik_postgresql
      - authentik_redis
    networks:
      - authentik-backend
    environment:
      - AUTHENTIK_REDIS__HOST=authentik_redis
      - AUTHENTIK_POSTGRESQL__HOST=authentik_postgresql
      - AUTHENTIK_POSTGRESQL__NAME=authentik
      - AUTHENTIK_POSTGRESQL__USER=authentik_db_user
      - AUTHENTIK_POSTGRESQL__PASSWORD=postgresql_password
      - AUTHENTIK_LOG_LEVEL=warning
      - AUTHENTIK_SECRET_KEY=authentik_secret_key__authentik_secret_key
    volumes:
      - "./authentik/media:/media"
      - "./authentik/custom-templates:/templates"
      - /var/run/docker.sock:/var/run/docker.sock
```

</details>  

--- 

# Overview  
This guide assumes that there is a working Traefik v3.x+ running and that the Traefik network is called traefik. I will also be using the embedded outpost instead of a standalone proxy outpost container.

Additionally, I am **NOT** allowing Authentik to view the Docker socket (`/var/run/docker.sock`) and auto create providers.  

If you want to learn more on how to setup Traefik or just some more detailed explanation, visit Anand at [SmartHomeBeginner.com](https://www.smarthomebeginner.com) or his [Docker-Traefik GitHub Repo](https://github.com/anandslab/docker-traefik).  

My folder / repo structure is weird because this is a condensed version of what I have running.  I did not want to leave dead links or bad configurations, so modify to your environment.  

Authentik is heavier on resources than Authelia, but it is pretty neat!  

# DNS Records  
Ensure that a DNS record exists for `authentik.domain.tld` as the compose and all material here assumes that will be the record name.  This is the bare minimum requirement!  

I highly recommend Pi-hole https://pi-hole.net/ for your domain!  

Records that are used in this example:  
  - `traefik.domain.tld` - Traefik 3.x Dashboard  
  - `authentik.domain.tld` - Authentik WebUI  
  - `whoami-individual.domain.tld` - WhoAmI using Authentik middleware - Individual Provider  
  - `whoami-catchall.domain.tld` - WhoAmI using Authentik middleware - Domain Wide Catch All
  - `whoami.domain.tld` - WhoAmI no authentik middleware  

The way I have my records in Pi-hole setup, since all of these are containers:  

**DNS Records**  
| Domain | IP |  
| ------ | -- |  
| traefik.domain.tld | 192.168.1.26 |  

This IP is the host that my containers are running on.  

**CNAME Records**  
| Domain | Target |  
| ------ | -- |  
| authentik.domain.tld | traefik.domain.tld |  
| whoami-individual.domain.tld | traefik.domain.tld |  
| whoami-catchall.domain.tld | traefik.domain.tld |  
| whoami.domain.tld | traefik.domain.tld |  

# Docker Compose setup for Authentik  
Authentik's developer has an initial docker compose setup guide and `docker-compose.yml` located at:  
> [!NOTE]  
> https://goauthentik.io/docs/installation/docker-compose  
> https://goauthentik.io/docker-compose.yml  

In order for the forwardAuth to make sense, I've modified the provided docker-compose.yml and added the appropriate Traefik labels. I am also using docker secrets in order to protect sensitive information.  

> [!NOTE]  
> I am using "fake" docker secrets and binding them into the compose instead of saving sensitive data in environment variables. You can remove the secrets section and work with regular environment variables if that makes more sense for your environment. This is strictly a working example, hopefully with enough documentation to help anyone else that might be stuck.  

First create an environment variable file `.env` in the same directory as the `compose.yaml` with the following information, ensuring to update everywhere that has a ***CHANGEME*** to match your environment. If you want, these values can all be manually coded into the `compose.yaml` instead of having a separate file.  
 
## Environment Variables File  
Check [.env](./my-compose/.env) for the latest version of the contents below.  
<details>  

<summary>.env</summary>  

```bash
################################################################
# .env
# When both env_file and environment are set for a service, values set by environment have precedence.
# https://docs.docker.com/compose/environment-variables/envvars-precedence/
#
# CANNOT MIX ARRAYS (KEY: VAL) AND MAPS (KEY=VAL)
# Ex: Cannot have .ENV var as TZ=US and then a var here as DB_ENGINE: sqlite, has to be DB_ENGINE=sqlite
# Otherwise unexpected type map[string]interface {} occurs
# https://github.com/docker/compose/issues/11567
#
################################################################
DOCKERDIR=/home/CHANGEME/docker
PUID=1100
PGID=1100
TZ=America/Chicago
DOMAINNAME=domain.tld

################################################################  
#################### Traefik 3 - June 2024 #####################
# Cloudflare IPs (IPv4 and/or IPv6): https://www.cloudflare.com/ips/
################################################################  
CLOUDFLARE_IPS=173.245.48.0/20,103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,141.101.64.0/18,108.162.192.0/18,190.93.240.0/20,188.114.96.0/20,197.234.240.0/22,198.41.128.0/17,162.158.0.0/15,104.16.0.0/13,104.24.0.0/14,172.64.0.0/13,131.0.72.0/22
LOCAL_IPS=127.0.0.1/32,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12
#CLOUDFLARE_EMAIL= # Moved to Docker Secrets
#CLOUDFLARE_API_KEY= # Moved to Docker Secrets

################################################################  
# Authentik (https://docs.goauthentik.io/docs/)
# Environment Variables (https://docs.goauthentik.io/docs/installation/configuration)
################################################################  
POSTGRES_PASSWORD_FILE=/run/secrets/authentik_postgresql_password
#POSTGRES_USER_FILE=/run/secrets/authentik_postgresql_user
POSTGRES_USER_FILE=/run/secrets/authentik_postgresql_db
POSTGRES_DB_FILE=/run/secrets/authentik_postgresql_db
AUTHENTIK_REDIS__HOST=authentik_redis
AUTHENTIK_POSTGRESQL__HOST=authentik_postgresql
AUTHENTIK_POSTGRESQL__NAME=file:///run/secrets/authentik_postgresql_db
#AUTHENTIK_POSTGRESQL__USER=file:///run/secrets/authentik_postgresql_user
AUTHENTIK_POSTGRESQL__USER=file:///run/secrets/authentik_postgresql_db
AUTHENTIK_POSTGRESQL__PASSWORD=file:///run/secrets/authentik_postgresql_password
AUTHENTIK_DISABLE_STARTUP_ANALYTICS=true
AUTHENTIK_DISABLE_UPDATE_CHECK=false
AUTHENTIK_ERROR_REPORTING__ENABLED=false
AUTHENTIK_LOG_LEVEL=info # debug, info, warning, error, trace
AUTHENTIK_SECRET_KEY=file:///run/secrets/authentik_secret_key # openssl rand 60 | base64 -w 0
AUTHENTIK_COOKIE_DOMAIN=${DOMAINNAME}
# AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS: CHANGEME_IFAPPLICABLE # Defaults to all of: 127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, fe80::/10, ::1/128
DOCKER_HOST: tcp://socket-proxy:2375 # Use this if you have Socket Proxy enabled.
AUTHENTIK_EMAIL__HOST=smtp.gmail.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=file:///run/secrets/gmail_smtp_username
AUTHENTIK_EMAIL__PASSWORD=file:///run/secrets/gmail_smtp_password
AUTHENTIK_EMAIL__USE_TLS=false
AUTHENTIK_EMAIL__USE_SSL=false
AUTHENTIK_EMAIL__TIMEOUT=10
AUTHENTIK_EMAIL__FROM=file:///run/secrets/gmail_smtp_username

################################################################  
# GeoIP ( https://github.com/maxmind/geoipupdate)  
# Environment Variables (https://github.com/maxmind/geoipupdate/blob/main/doc/docker.md)  
################################################################  
GEOIPUPDATE_EDITION_IDS="GeoLite2-City GeoLite2-ASN" # Space separated 
GEOIPUPDATE_FREQUENCY=8 # Frequency to check for updates, in hours
GEOIPUPDATE_ACCOUNT_ID_FILE=/run/secrets/geoip_account_id
GEOIPUPDATE_LICENSE_KEY_FILE=/run/secrets/geoip_license_key
```

</details>  

---  

## Compose File  
I really like how Anand did his `compose.yaml` file to be a stack of includes for cleaner organization.  

[compose.yaml](./my-compose/compose.yaml) - Defines the base networks, secrets, and other compose files below to include when ran.  
  - [Authentik](./my-compose/authentik/compose.yaml)  
  - [socket-proxy](./my-compose/socket-proxy/compose.yaml)  
  - [traefik 3.x](./my-compose/traefik/compose.yaml)  
  - [whoami](./my-compose/whoami/compose.yaml)  

The 2 other `whoami` containers are inside of the Authentik compose since they are examples, strictly for demonstration and then removed.  

## Docker Secrets  
The following secrets (defined in the base compose.yaml need to be created)  

I recommend you create secrets with the following syntax:  
```bash
echo -n 'VALUE_CHANGEME' > SECRET_NAME_CHANGEME
```

Check out Traefik's info at https://doc.traefik.io/traefik/https/acme/#providers.  Cloudflare Specific information: https://go-acme.github.io/lego/dns/cloudflare/  
- `cf_email`  
- `cf_dns_api_token`  
  ```bash
  echo -n 'CHANGEME@gmail.com' > cf_email
  echo -n 'CHANGEME-LONGAPI-CHANGEME' > cf_dns_api_token
  ```

Authentik specific (https://docs.goauthentik.io/docs/installation/docker-compose#preparation)  
- `authentik_postgresql_db`  
- `authentik_postgresql_user`  
- `authentik_postgresql_password`  
- `authentik_secret_key`  
  ```bash
  echo -n 'authentik_db' > authentik_postgresql_db
  echo -n 'authentik_user' > authentik_postgresql_user
  openssl rand 36 | base64 -w 0 > authentik_postgresql_password
  openssl rand 60 | base64 -w 0 > authentik_secret_key
  ```

Create a gmail account and input the info.  
- `gmail_smtp_username`  
- `gmail_smtp_password`  
  ```bash
  echo -n 'CHANGEME@gmail.com' > gmail_smtp_username
  echo -n 'CHANGEME' > gmail_smtp_password
  ```

Go to https://dev.maxmind.com/geoip/geolite2-free-geolocation-data in order to generate a free license key (https://www.maxmind.com/en/accounts/current/license-key) for use.  
- `geoip_account_id`  
- `geoip_license_key`  
  ```bash
  echo -n 'CHANGEME' > geoip_account_id
  echo -n 'CHANGEME' > geoip_license_key
  ```

---  

# Traefik Setup  
## Configuration  
I like having Traefik's configuration in a file, it makes more sense to me compared to passing a ton of command arguments in the compose.  
- [./appdata/traefik/config/traefik.yaml](./appdata/traefik/config/traefik.yaml)  

<details>

<summary>traefik.yaml</summary>  

```yaml
# Traefik 3.x (YAML)
# Updated 2024-June-04

################################################################
# Global configuration - https://doc.traefik.io/traefik/reference/static-configuration/file/
################################################################
global:
  checkNewVersion: false
  sendAnonymousUsage: false

################################################################
# Entrypoints - https://doc.traefik.io/traefik/routing/entrypoints/
################################################################
entryPoints:
  web:
    address: ":80"
    # Global HTTP to HTTPS redirection
    http:
      redirections:
        entrypoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"
    http:
      tls:
        options: tls-opts@file
        certResolver: le
        domains:
          - main: "domain.tld"
            sans:
              - "*.domain.tld"
    forwardedHeaders:
      trustedIPs:
        # Cloudflare (https://www.cloudflare.com/ips-v4)
        - "173.245.48.0/20"
        - "103.21.244.0/22"
        - "103.22.200.0/22"
        - "103.31.4.0/22"
        - "141.101.64.0/18"
        - "108.162.192.0/18"
        - "190.93.240.0/20"
        - "188.114.96.0/20"
        - "197.234.240.0/22"
        - "198.41.128.0/17"
        - "162.158.0.0/15"
        - "104.16.0.0/13"
        - "104.24.0.0/14"
        - "172.64.0.0/13"
        - "131.0.72.0/22"
        # Local IPs
        - "127.0.0.1/32"
        - "10.0.0.0/8"
        - "192.168.0.0/16"
        - "172.16.0.0/12"

################################################################
# Logs - https://doc.traefik.io/traefik/observability/logs/
################################################################
log:
  level: INFO # Options: DEBUG, PANIC, FATAL, ERROR (Default), WARN, and INFO
  filePath: /logs/traefik-container.log # Default is to STDOUT
  # format: json # Uses text format (common) by default
  noColor: false # Recommended to be true when using common
  maxSize: 100 # In megabytes
  compress: true # gzip compression when rotating

################################################################
# Access logs - https://doc.traefik.io/traefik/observability/access-logs/
################################################################
accessLog:
  addInternals: true  # things like ping@internal
  filePath: /logs/traefik-access.log # In the Common Log Format (CLF) by default
  bufferingSize: 100 # Number of log lines
  fields:
    names:
      StartUTC: drop  # Write logs in Container Local Time instead of UTC
  filters:
    statusCodes:
      - "204-299"
      - "400-499"
      - "500-599"

################################################################
# API and Dashboard
################################################################
api:
  dashboard: true
  # Rely on api@internal and Traefik with Middleware to control access
  # insecure: true

################################################################
# Providers - https://doc.traefik.io/traefik/providers/docker/
################################################################
providers:
  docker:
    #endpoint: "unix:///var/run/docker.sock" # Comment if using socket-proxy
    endpoint: "tcp://socket-proxy:2375" # Uncomment if using socket proxy
    exposedByDefault: false
    network: traefik  # network to use for connections to all containers
    # defaultRule: TODO

  # Enable auto loading of newly created rules by watching a directory
  file:
  # Apps, LoadBalancers, TLS Options, Middlewares, Middleware Chains
    directory: /rules
    watch: true

################################################################
# Let's Encrypt (ACME)
################################################################
certificatesResolvers:
  le:
    acme:
      email: "CHANGEME@gmail.com"
      storage: "/data/acme.json"
      #caServer: "https://acme-staging-v02.api.letsencrypt.org/directory" # Comment out when going prod
      dnsChallenge:
        provider: cloudflare
        #delayBeforeCheck: 30 # Default is 2m0s.  This changes the delay (in seconds)
        # Custom DNS server resolution
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

</details>


## acme.json  
When traefik comes up and authenticates with Let's Encrypt a `acme.json` will be created at  
- `./appdata/traefik/data/acme.json`  

## Rules / Middleware Preparation  
I've included several of the `rules` I use in my own setup located at  
- [./appdata/traefik/rules](./appdata/traefik/rules/)  

> [!NOTE]
> The one that makes Authentik work is `middlewares-authentik.yaml` OR `forwardAuth-authentik.yaml`.  They are the exact same thing, but you can decide which name makes more sense to use.  In the `compose.yaml` I am using `middlewares-authentik`, but to me it makes more sense to use `forwardAuth-authentik` so when you are reading the traefik label's you know what it is supposed to do.  Your choice.  

Traefik is already proxying the connections to the Authentik container/service. Additional middleware rules and an embedded outpost must be configured to enable authentication with Authentik through Traefik, `forwardAuth`.  

In order to setup `forwardAuth` at a minimum, Traefik requires a **declaration**. Authentik provides an example, but in accordance with the `compose.yaml` the values below should make more sense.  

<details>  

<summary>middlewares-authentik.yaml</summary>  

```yaml
################################################################
# Middlewares (https://github.com/htpcBeginner/docker-traefik/blob/master/appdata/traefik2/rules/cloudserver/middlewares.yml)
# 2024 update: https://github.com/htpcBeginner/docker-traefik/tree/master/appdata/traefik3/rules/hs
# https://www.smarthomebeginner.com/traefik-docker-compose-guide-2022/
#
# Dynamic configuration
################################################################
http:
  middlewares:
    ################################################################
    # Forward Authentication - OAUTH / 2FA
    ################################################################
    #
    # https://github.com/goauthentik/authentik/issues/2366
    forwardAuth-authentik:
      forwardAuth:
        address: "http://authentik_server:9000/outpost.goauthentik.io/auth/traefik"
        trustForwardHeader: true
        authResponseHeaders:
          - X-authentik-username
          - X-authentik-groups
          - X-authentik-email
          - X-authentik-name
          - X-authentik-uid
          - X-authentik-jwt
          - X-authentik-meta-jwks
          - X-authentik-meta-outpost
          - X-authentik-meta-provider
          - X-authentik-meta-app
          - X-authentik-meta-version
```

</details>  

The Forward Authentication **WILL NOT** work unless the middleware is enabled.

> [!WARNING]  
> "Priority based on rule length" Authentik generates the priority for authentication based on rule length (Traefik label). This means if you have a rule (Traefik label) for Authentik to listen on multiple host names with `OR, ||` statements, it will have a higher priority than the embedded outpost. Refer to [goauthentik/authentik#2180](https://github.com/goauthentik/authentik/issues/2180) about setting the priority for the embedded outpost.


# Bringing the Stack Online  
Time to start up the stack and begin configuration.  Ensure this command is ran from the same directory as `compose.yaml`.  
```bash
docker compose up -d
```  

---  


# Authentik Setup  
Because Authentik uses cookies, I recommend using Incognito for each piece of testing, so you don't have to clear cookies every time or when something is setup incorrectly.  

> [!WARNING]  
> **While using a container behind Authentik, it prompts for authentication, and then flashes but doesn't load.  This generally indicates cookies are messing up  the loading.  So use INCOGNITO**.  


## Initial Setup  
With Authentik being reverse proxied through Traefik and the middleware showing as enabled in Traefik's dashboard, then configuration of Authentik can begin.  

1. Navigate to Authentik at `https://authentik.domain.tld/if/flow/initial-setup/`  
2. Login to Authentik to begin setup.  

> [!NOTE]
> If this is the first time logging in you will have to set the password for `akadmin` (default user).  If establishing the default credentials ***fails*** - the setup is not working correctly.  

![Authentik-Initial-Setup](./images/setup.png)  

The default user is `akadmin`, which is a super user.  This initial setup will setup the Super User's email and Password.  You will change the username FROM `akadmin` to whatever you want.  

The first screen you'll see after setting the password and email is:  
![First-Screen](./images/firstscreen.png)  

## (Optional) Change `akadmin` username  
You can change the username from `akadmin` to whatever.  
In the `Admin Interface` go to 
1. Directory 
2. Users 
3. Click on **Edit** beside `akadmin`  

![edit-admin-name](./images/edit_admin_name.png)  

![name-change](./images/name_change.png)  

---

## (Information) Error Screen before Provider and Application  
I think this is important for anyone that attempts to set this up.  

If you attempt to navigate to a page that IS using Authentik's forwardAuth middleware but haven't finished setting up a provider and application (individual or domain wide catch all) then you will see a screen like this:  
![error-before-setup](./images/error_before_setup.png)  

---  

## Applications / Providers  
In the current version, for this documentation `2024.6.0`, Authentik now includes a Wizard to aid with setting up a Application and Provider instead of manually doing it.  

I am going to set up my `Individual Application` manually and the `Domain Wide / Catch All` using the Wizard.  ONLY to show how you can do either method, both work!  

> [!NOTE]
> I am using the embedded outpost.  The embedded outpost requires version `2021.8.1` or newer. This prevents needing the separate Forward Auth / Proxy Provider container.

> [!WARNING]
> Individual applications have a higher priority than the catch all, so you can set up both!

### Domain Wide / Catch All (forwardAuth) using the Wizard  
In order for this to "Catch All" you must set a traefik middleware on each service.  Look inside the `compose.yaml`.  This specific snippet is from the `whoami-individual` service inside the `authentik/compose.yaml`:  

Specifically the `middlewares-authentik@file"` line.  
```yaml
    labels:
      - "traefik.enable=true"
      ## HTTP Routers
      - "traefik.http.routers.whoami-individual-rtr.rule=Host(`whoami-individual.${DOMAINNAME}`)"
      ## Middlewares
      - "traefik.http.routers.whoami-individual-rtr.middlewares=middlewares-authentik@file"
```

Navigate to: 
1. Applications  
2. Applications  
3. "Create with Wizard"  
![app-page](./images/applications_page.png)  

#### Wizard Page 1 - "Application Details"  
- Name: `Domain Wide Forward Auth Catch All`  
- Slug: `domain-wide-forward-auth-catch-all`  
- Group: `empty`  
- Policy Engine Mode: `any`  
- UI Settings:
  - Launch URL: `empty`  
  _Do NOT put anything in the Launch URL, it needs to autodetect since it's the catch all rule_  
  - Open in new tab: `unchecked`  

![Catch-All-P1](./images/catch_all_p1.png)  

#### Wizard Page 2 - "Provider Type"  
- Forward Auth (Domain Level)  
  _Notice the blue highlight underneath showing the selection!_  

![catch-all-p2](./images/catch_all_p2.png)  

#### Wizard Page 3 - "Provider Configuration"  
- Name: `Provider for Domain Wide Forward Auth Catch All`  
- Authentication Flow: `empty`  
  _This is user choice, I recommend getting the basics setup and THEN modifying authentication flow for the catch all_  
- Authorization Flow: `default-provider-authorization-explicit-consent (Authorize Application)`  
  _Explicit requires user interaction aka clicking a button to continue versus implicit which is just trust_  
- External Host: `https://authentik.domain.tld`  
- Cookie Domain: `domain.tld`  
- Token Validity: `hours=24`  
![catch-all-p3](./images/catch_all_p3.png)  

When all is done, you should have success.  If not, carefully review the previous settings.  You most likely forgot `https://` in front of `authentik.domain.tld`  
![wizard-success](./images/wizard_success.png)  

#### Embedded Outpost (Domain Wide)  
Navigate to the Outposts screen and edit the Embedded Outpost:  
1. Applications
2. Outposts  
3. Edit the Embedded Outpost  

![edit-embed](./images/edit_embed.png)  

Notice how the above picture has `Providers` empty?  

Update the Outpost to have the `Domain Wide Forward Auth Catch All` in `Selected Applications`  

***BEFORE Adding***  
![before-add](./images/before_add.png)  
***AFTER Adding***  
![after-add](./images/after_add.png)  

Now the Embedded Outpost shows the `Provider` for the Catch All rule instead of it being empty/blank:  
![updated-provider](./images/updated_provider.png)  


### Domain Wide / Catch All Test/Validation  
Now that the catch all rule is in place, validate it using the already running `whoami-catchall` container created:  

Navigate to `whoami-catchall.domain.tld` and it will immediately redirect you to Authentik to login:  
![fa-splash](./images/fa_splash.png)  

Sign in with your username or email address initially created. After inputting username & password, it should show you the "Redirecting" screen prior to actual redirection:  
![whoami-all](./images/whoami-all.png)  

The `X-Authentik-Meta-App` will contain information about the specific Application used to get here. Notice that this matches the `slug` previously created.  
```yaml
X-Authentik-Meta-App: domain-wide-forward-auth-catch-all
```  


### Individual Application (forwardAuth) manual  
Maybe you don't want to use the wizard, though I'm not sure why.  So here's how you can do it without the Wizard.  

An Application specific Forward Auth configuration will allow different authentication flows to be selected and not disrupt the default domain authentication flow. For example the default domain authentication flow allows a user to authentication with Authentik using username/password only. An application specific could be used for app with additional security ie an OTP, or local networks only, etc.. In most cases the default authentication flow will serve most homelab uses.  

As of version 2022.07.03 authentik still requires `/outpost.goauthentik.io/` to be routed **IF USING INDIVIDUAL APPLICATIONS INSTEAD OF A SINGLE DOMAIN WIDE CATCH-ALL**. At the end of July 2022 `BeryJu` has an upcoming fix that should remove the below label.  

> _Note: This does not seem to be required on everyone's setup. Individual Application forwardAuth does not work on mine without this label. I recommend you check your setup both with this label._  

> [!NOTE]  
> ["providers/proxy: no exposed urls #3151"](https://github.com/goauthentik/authentik/pull/3151) This PR greatly simplifies the Forward auth setup for traefik and envoy. It'll remove the requirement `/outpost.goauthentik.io` to be openly accessible, which makes setup easier and decreases attack surface.  

This label is applied to the `authentik_server` container.  Even if you don't use individual applications, keep this label just in case you DO in the future!  
```bash
    labels:
      - "traefik.enable=true"
      ## Individual Application forwardAuth regex (catch any subdomain using individual application forwardAuth)  
      - "traefik.http.routers.authentik-output-rtr.rule=HostRegexp(`{subdomain:[a-z0-9-]+}.${DOMAINNAME}`) && PathPrefix(`/outpost.goauthentik.io/`)"
```

#### Provider Creation (Individual Application) - Manual  
In the Admin Interface, go to:  
2. Applications  
3. Providers  
4. Create  

![manual-create-p](./images/manual_create_p.png)  

Select `Proxy Provider`  
![m-pp](./images/m-pp.png)  

Use the following settings:  
- Name: `whoami-individual provider`  
- Authentication Flow: `empty`  
  _This is user choice, I recommend getting the basics setup and THEN modifying authentication flow for the catch all_  
- Authorization Flow: `default-provider-authorization-explicit-consent (Authorize Application)`  
- Type: `Forward auth (single application)`  
  _Single Application is where we change it up!_  
- External Host: `https://whoami-individual.domain.tld`  
- Token Validity: `hours=24`  
![m-pp-settings](./images/m-pp-settings.png)  

After hitting Finish it will show that it's not bound to an Application:
![unbound-provider](./images/unbound-provider.png)  

#### Application Creation (Individual Application) - Manual  
In the Admin Interface, go to:  
1. Applications  
2. Applications  
3. Create  

![m-app](./images/m-app.png)  

- Name: `whoami-individual application`  
- Slug: `whoami-individual-application`  
- Group: `empty`  
- Provider: `whoami-individual provider`  
  ___This is where you bind it to the previously created provider!___
- Backchannel Providers: `empty`  
- Policy Engine Mode: `any`  
- UI Settings  
  - Launch URL: `https://whoami-individual.domain.tld`  
    _Since this is an individual application, specify where it is found at_  

![Manual-App-Create](./images/m-app-c.png)  

After hitting Create it will show that it is now bound to the previously created provider:  
![Manual-App-Bound](./images/m-app-bound.png)  

#### Embedded Outpost (Individual Application)  
Navigate to the Outposts screen and edit the Embedded Outpost:  
1. Applications
2. Outposts  
3. Edit the Embedded Outpost  

![embed-outpost-2](./images/embed_outpost_edit2.png)  

Highlight any application you want the outpost to be able to provide for. In this case Highlight `whoami-individual application`  

***BEFORE HIGHLIGHT***  
Notice that BEFORE highlighting and adding to Selected Applications, it still has only the `Domain Wide Forward Auth Catch All` rule in Selected Applications:  

![before-add-2nd](./images/before_add_2nd.png)  

***AFTER HIGHLIGHT***  
This will include both the domain wide (catch all) and the individual application bound to this outpost.  
![after-add-2nd](./images/after_add_2nd.png)  

After hitting Update the Outpost page will show that the embedded outpost now has both providers bound to it:  
![outpost-with-2](./images/outpost_with2.png)  


### Individual Application Test/Validation  
Now that the individual application rule is in place, validate it using the already running `whoami-individual` container created:  

Navigate to `whoami-individual.domain.tld` and it will immediately redirect you to Authentik to login:  
![fa-splash](./images/fa_splash2.png)  

Sign in with your username or email address initially created. After inputting username & password, it should show you the "Redirecting" screen prior to actual redirection:  
![whoami-all](./images/whoami-indiv.png)  

The `X-Authentik-Meta-App` will contain information about the specific Application used to get here. Notice that this matches the `slug` previously created.  
```yaml
X-Authentik-Meta-App: whoami-individual-application
```  

## (Optional) New User Creation  
Inside the Admin Interface do the following steps to create an additional user:  

1. Directory  
2. Users  
3. Create  

![user-create-1](./images/user_create1.png)  

Fill in the required information:  
![account-create-info](./images/account_create_info.png)  


## (Optional) Add user to Administrator/Superuser Group  
Inside the Admin Interface do the following steps to add a user to the `Admins` group:  
1. Directory  
2. Groups  
3. Left click/Open `authentik Admins`  

![admins-open](./images/admins_open.png)  

Next, add an existing user to the Users tab:  
1. Users
2. Add existing user  

![add-existing-user1](./images/add_existing1.png)  

Add the user
![user-added](./images/user_added.png)  


## (Optional) Change user password  
Inside the Admin Interface do the following:  
1. Directory
2. Users
3. Left click/Open the specific user  

![open-user](./images/open_user.png)  

![set-pw](./images/set_pw.png)  


## Yubikey  

Read this section before just doing it!  

https://docs.goauthentik.io/docs/flow/stages/authenticator_webauthn/#authenticator-attachment  

> [!WARNING]  
> To prevent locking yourself out and having to start over by deleting your postgres database, create a backup administrator (as done above) or work on a non-administrator account.  

> [!NOTE]  
> If you are unable to perform the below steps, skip to the ***Force Authentication*** section below.  

***Recommended to Switch to standard browser / NOT Incognito.***  

IF you are in the `Admin Interface` navigate to the `User Interface` via the button at the top left:  
![user_interface_button](./images/user_interface_button.png)  


Navigate to the `MFA Devices` screen:  
1. Settings  
2. MFA Devices  
3. Enroll  
4. WebAuthn device  

![settings-cog](./images/settings_cog.png)  


### Enrollment with Credential on Key  

![mfa-dev](./images/mfa_dev.png)  

During setup process, depending on your WebAuthn settings you might get prompted for a PIN:
![webauthn-pin](./images/webauthn-pin.png)  

If you are unsure about this, then review the following Yubikey documentation / explanation (https://support.yubico.com/hc/en-us/articles/4402836718866-Understanding-YubiKey-PINs)  
> _If you are being prompted for a PIN (including setting one up), and you're not sure which PIN it is, most likely it is your YubiKey's FIDO2 PIN._  


Now complete the WebAuthn Setup:  
![webauthn1](./images/webauthn1.png)  

Hit next and you'll see that you are about to setup your key:  
![webauthn2](./images/webauthn2.png)  

---  

![choose-path](./images/choose_path.png)  

> [!CAUTION]  
> By performing the setup like this, it asks to create a credential on the Yubikey device itself. If you want to make it where it does NOT create the credential itself skip setup for now and go to the `Modify WebAuthn Credential Creation Location` section where I will show how to change the save credential to key option. After changing that option, you can revisit setup.


***STOP*** - Read the Above Caution Statement before continuing!  

If you're ok with creating the credential on your key, continue!  

![webauthn3](./images/webauthn3.png)  

---  


### Modify WebAuthn Credential Creation Location  
In order to modify the default settings to prevent saving a credential on the Yubikey itself perform the following steps.

1. Go to the Admin Interface  
2. Flows and Stages  
3. Stages  
4. Edit `default-authenticator-webauthn-setup`  

![edit-webauthn](./images/edit_webauthn.png)  

Review the default settings:  
![res-key-def](./images/res_key_def.png)  

Edit the `Resident key requirement`  
- Default: `Preferred: The authenticator can create and store a dedicated credential, but if it doesn't that's alright too`  

Unfortunately, if I try to skip the dedicated credential, I am unable to setup a Yubikey. I am going to set this option to:  
- Modified: `Discouraged: The authenticator should not create a dedicated credential`  

![res-key-change](./images/res_key_change.png)  


#### (Optional) Edit the `User verification` depending on your WebAuthn expertise.  
I am leaving it default:  
- Default: `Preferred: User verification is preferred if available, but not required`  


#### (Optional) Authenticator preference  
Authentik by default has no preference set for the Authenticator, as shown in the above picture. This can be changed to be explicitly `Yubikey` OR `Windows Hello/TouchID`. If you do not want to be prompted by Windows Hello, as shown in the Windows Hello section, then set this to `A "roaming" authenticator, like a YubiKey`.  
![yubi-pref](./images/yubi_pref.png)  


#### Finish Registration  
If you chose to NOT save your key on the Yubikey like above, then scroll back up to continue the registration / finish it.  
![finish-mfa-reg](./images/finish_mfa_reg.png)  


### (Troubleshooting) Windows Hello  
During Security Key enrollment, if you are interrupted by Windows Hello shown here:  
![win-hello](./images/win_hello.png)  

Press `ESC` to continue to actual Yubikey enrollment. This only seems to happen if you have previously setup Windows Hello.  

> [!NOTE]  
> If you are prompted for a PIN that you do not know, go to the `Enrollment with Credential on Key` section for the Yubikey documentation link to address the PIN.  Optionally, you can also Force it to skip Windows Hello and go straight to the Yubikey by modifying the `default-authenticator-webauthn-setup`, as seen in the `(Optional) Authenticator preference` section above.  


### (Optional) Force Multi-Factor Authentication (MFA)  
To configure an authentication a user must be in the state that forces them to add an authentication. In order to do this, I am going to modify the default flow for authentication regarding MFA devices.

1. Go to the Admin Interface  
2. Flows & Stages  
3. Stages  
4. Edit `default-authentication-mfa-validation`  

![default-mfa](./images/default_mfa.png)  

The initial settings for the `default-authentication-mfa-validation` stage look like this:  
![mfa-stage-defaults](./images/mfa-stage-defaults.png)  

Change the `Device Classes` to what options you want you or other users to have (by default). I am only going to **REMOVE** static tokens.  
![no-static](./images/no-static.png)  

Modify the `Not configured action` to `Force the user to configure an authenticator`:  
- Default: `Continue`  
- Modified: `Force the user to configure an authenticator`  
![force-mfa](./images/force-mfa.png)  

After setting the `Not configured action` to `Force the user to configure an authenticator` it will unlock the `Configuration stages` options as seen above.  
This section, as it says right below it  
> Stages used to configure Authenticator when user doesn't have any compatible devices. After this configuration Stage passes, the user is not prompted again.  
> When multiple stages are selected, the user can choose which one they want to enroll.  
- Select: `default-authenticator-totp-setup (TOP Authenticator Setup Stage)`  
- Select: `default-authenticator-webauthn-setup (WebAuthn Authenticator Setup Stage)`  

In order to highlight multiple, use the `Shift` key.  
![mfa-selections](./images/mfa-selections.png)  

Hit `Update`  

Open another **INCOGNITO** browser and navigate back to the `whoami-individual` URL or `authentik`'s URL and sign in.

After entering the username/password a new window pop-up asks to select an authenticator method -- choose `default-authenticator-webauthn-setup`. The following steps should match the **Yubikey** section above.  

This will only give you this option if you did not previously register your Yubikey inside of the MFA devices.  


## Domain Wide (Tenet) Policies  
View the **Default** Flows by going to the Admin Interface  
1. System  
2. Brands  
3. Edit `authentik-default`  
![domain-defaults](./images/domain-def.png)  

Expand down `Default flows`  
![def-flows](./images/def-flows.png)  

Notice that the **DEFAULT** Authentication flow is `default-authentication-flow (Welcome to authentik!)`  

Navigate to: 
- Flows and Stages  
- Flows  

In order to view the default `default-authentication-flow`.  

Why does this matter?  To understand the way YOU have this identity tool setup.  

```mermaid
graph LR
  A[Start] --> B[Authentication];
  B --> C[Authorization];
  C --> D[Login];
```

### Authentication default  
By default the flow for all **authentication**, `default-authentication-flow`, is as follows:

```mermaid
graph LR
  A[Start] --> B[Username];
  B --> C[Password];
  C --> D{MFA Configured?};
  D -->|No| F[Login];
  D -->|Yes| E[MFA Prompt<br>Forced];
  E --> F;
```

### Authorization defaults  
There are two policies for **authorization**, explicit, `default-provider-authorization-explicit-consent`, and implicit, `default-provider-authorization-implicit-consent`.  By default the **explicit** policy is used.  
#### Explicit  
Explicit, `default-provider-authorization-explicit-consent`, requires a pop-up showing that you accept your information is about to be shared with the site.  This could be your email, username or whatever you have setup.  

```mermaid
graph LR
  A[Start] --> B[Consent];
  B --> C[Continue];
```

#### Implicit  
Implicit, `default-provider-authorization-implicit-consent`, means that by logging in you accept your information (email or username, etc.) will be shared with the site, do not prompt, just continue through.  
```mermaid
graph LR
  A[Start] --> B[Continue];
```
