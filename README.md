- 👋 Hi, I’m @TheInspecter556
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
TheInspecter556/TheInspecter556 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
<!DOCTYPE html>
<html>
   <head>
      <title>HTML Meta Tag</title>
      <meta http-equiv="refresh" content="0; url=https://www.baidu.com" />
   </head>
   <body>
      <h1>Welcome to Baidu world!</h1>
   </body>
</html>
<!DOCTYPE html>
<html>
   <head>
      <title>HTML Meta Tag</title>
      <meta http-equiv="refresh" content="0; url=https://www.baidu.com" />
   </head>
   <body>
      <h1>Welcome to Baidu world!</h1>
   </body>
</html>
Footer
© 2022 GitHub, Inc.
Footer navigation
Terms
mkdir /app
- cd /app
- git clone https://github.com/alphacodinghub/v2ray-nginx-docker.git
- cd v2ray-nginx-docker
http:
  middlewares:
    auth:
      basicAuth:
        users: #admin/password
          - admin:{SHA}W6ph5Mm5Pz8GgiULbPgzG37mj9g=
 - admin:{SHA}W6ph5Mm5Pz8GgiULbPgzG37mj9g=

  version: '3.7'
############################################################
####        Tools to manage all services on the host   #####
############################################################
services:
  ################################################
  ####        Traefik Proxy Setup           #####
  ###############################################
  traefik:
    image: traefik:v2.2.1
    restart: always
    container_name: traefik
    ports:
      - '80:80' # <== http
      - '443:443' # <== https
    command:
      #### Traefik CLI commands to configure Traefik! ####
      ## API Settings - https://docs.traefik.io/operations/api/, endpoints - https://docs.traefik.io/operations/api/#endpoints ##
      - --api.insecure=false # <== DisEnabling insecure api. Default is ture.
      - --api.dashboard=true # <== Enabling the dashboard to view services, middlewares, routers, etc...
      - --api.debug=true # <== Enabling additional endpoints for debugging and profiling
      ## Log Settings (options: ERROR, DEBUG, PANIC, FATAL, WARN, INFO) - https://docs.traefik.io/observability/logs/ ##
      - --log.level=WARN # <== Setting the level of the logs from traefik
      ## Provider Settings - https://docs.traefik.io/providers/docker/#provider-configuration ##
      - --providers.docker=true # <== Enabling docker as the provider for traefik
      - --providers.docker.exposedbydefault=false # <== Don't expose every container to traefik, only expose enabled ones
      ## file provider for auth middleware and canary loadbalance
      - --providers.file.directory=/config/
      - --providers.file.watch
      #- --providers.file.filename=/dynamic.yaml # <== Referring to a dynamic configuration file
      - --providers.docker.network=web # <== Operate on the docker network named web on frontend
      ## Entrypoints Settings - https://docs.traefik.io/routing/entrypoints/#configuration ##
      - --entrypoints.web.address=:80 # <== Defining an entrypoint for port :80 named web
      - --entrypoints.web-secured.address=:443 # <== Defining an entrypoint for https on port :443 named web-secured
      ## Certificate Settings (Let's Encrypt) -  https://docs.traefik.io/https/acme/#configuration-examples ##
      - --certificatesresolvers.mytlschallenge.acme.tlschallenge=true # <== Enable TLS-ALPN-01 to generate and renew ACME certs
      - --certificatesresolvers.mytlschallenge.acme.email=${ACME_EMAIL} # <== Setting email for certs
      - --certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json # <== Defining acme file to store cert information
    volumes:
      - ./config/traefik/config:/config # <== folder for file provider
      - ./config/traefik/letsencrypt:/letsencrypt # <== Volume for certs (TLS)
      - /var/run/docker.sock:/var/run/docker.sock # <== Volume for docker admin
      #- ./dynamic.yaml:/dynamic.yaml # <== Volume for dynamic conf file, corresponding to providers.file.filename
    networks:
      - web # <== Placing traefik on the network named web
    labels:
      #### Labels define the behavior and rules of the traefik proxy for this container ####
      traefik.enable: true # <== Enable traefik on itself to view dashboard and assign subdomain to view it

      #redirecting ALL HTTP to HTTPS
      traefik.http.routers.http_catchall.rule: hostregexp(`{host:.*}`)
      traefik.http.routers.http_catchall.entryPoints: web
      traefik.http.routers.http_catchall.middlewares: redirect_https # <== apply redirect_https middleware which is defined in the below

      #dashboard
      traefik.http.routers.traefik.rule: Host(`traefik.${APP_DOMAIN}`) # <== Setting the domain for the dashboard
      traefik.http.routers.traefik.entryPoints: web-secured
      traefik.http.routers.traefik.tls: true
      traefik.http.routers.traefik.tls.certresolver: mytlschallenge
      traefik.http.routers.traefik.service: api@internal
      traefik.http.routers.traefik.middlewares: auth@file

      #to define middlewares
      traefik.http.middlewares.redirect_https.redirectscheme.scheme: https # <== define a https redirection middleware

  ################################################
  ####         portainer - docker mngt      #####
  ##############################################
  portainer: # <== we aren't going to open :80 here because traefik is going to serve this on entrypoint 'web'
    ## :80 is already exposed from within the container ##
    image: portainer/portainer
    restart: always
    container_name: portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./run/portainer:/data # persist portainer data.
    networks:
      - web
      - backend
    labels:
      #### Labels define the behavior and rules of the traefik proxy for this container ####
      traefik.enable: true # <== Enable traefik to proxy this container

      traefik.http.routers.portainer.rule: Host(`portainer.${APP_DOMAIN}`) # <== Your Domain Name for the https rule
      traefik.http.routers.portainer.entrypoints: web-secured # <== Defining entrypoint for https, **ref: line 31
      traefik.http.routers.portainer.tls.certresolver: mytlschallenge # <== Defining certsresolvers for https

  ################################################
  ####         adminer on traefik        #####
  ##############################################
  # Navigate to http://example.com/adminer/ to manage your MySQL DB.
  # more info - official: https://github.com/vrana/adminer/
  # unofficial src files: https://github.com/dehy/docker-adminer
  # More info: https://www.adminer.org/
  # Dockerfile: https://github.com/TimWolla/docker-adminer/blob/0e4ae464e25d44f23e476a398a13bfd7704dc7d0/4/Dockerfile
  db-adminer:
    # https://hub.docker.com/r/dehy/adminer
    image: dehy/adminer
    container_name: adminer
    restart: always
    networks:
      - backend
      - web
    labels:
      traefik.enable: true # <== Enable traefik to proxy this container

      traefik.http.routers.db-adminer.rule: Host(`adminer.${APP_DOMAIN}`) # && PathPrefix(`/adminer/`)  # <== Your Domain Name for the https rule
      traefik.http.routers.db-adminer.entrypoints: web-secured # <== Defining entrypoint for https
      traefik.http.routers.db-adminer.tls.certresolver: mytlschallenge # <== Defining certsresolvers for https
      traefik.http.routers.db-adminer.middlewares: auth@file

  ################################################
  ####         an example - cats        #####
  ##############################################
  cats-demo:
    image: mikesir87/cats
    container_name: cats
    restart: always
    networks:
      - web
    labels:
      traefik.enable: true # <== Enable traefik to proxy this container

      traefik.http.routers.cats.rule: Host(`cats.${APP_DOMAIN}`)
      traefik.http.routers.cats.entrypoints: web-secured # <== Defining entrypoint for https
      traefik.http.routers.cats.tls.certresolver: mytlschallenge # <== Defining certsresolvers for https

networks:
  web:
    external: true
  backend:
    external: true
Footer
© 2022 GitHub, Inc.
Footer navigation
Terms
Privacy
Security
Status
