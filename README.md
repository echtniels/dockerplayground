# dockerplayground
Setup a secure Docker playground on your own VPS.

Pre-requisites:
1. Internet facing Linux VPS; I built on Ubuntu 18.04 LTS
2. Domain name under your control (to use wildcard certs) and a Cloudflare account - a free one will do fine.
3. Docker CE and Docker-Compose running

# Preparing the environment
1. Prepare data directory and an empty letsencrypt cert-config, because docker can only create directories, not files, and we do not want either owned by root.
```
mkdir data
mkdir data/letsencrypt
touch data/letsencrypt/acme.json
chmod 600 data/letsencrypt/acme.json
```

2. Open ./environment.sample and rename variables to reflect your situation. Then, either rename it to `.env` or append to `/etc/environment` (don't forget to logout and on again to update your environment).
3. Run the compose file
```
docker-compose -p traefik-keycloak up -d
```
4. Then, for some time, check how setup runs in the container logs:
```
docker logs -f traefik
```
5. wait... https://traefik.yourdomain.com becomes available. If at first you get a certificate warning, wait some more and refresh.

6. add services by adding labels as in template .yml (don't forget to comment out exposed ports in service yml file, traefik does all the exposing)
