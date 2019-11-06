# dockerplayground
Setup a secure Docker playground on your own VPS

Pre-requisites:
1. Internet facing Linux VPS; I built on Ubuntu 18.04 LTS
2. Domain name under your control (to use wildcard certs)
3. Docker CE and Docker-Compose running

# Setup Traefik 2.0 as reverse proxy
1. mkdir $USERDIR/docker/letsencrypt: cd $USERDIR/docker/letsencrypt ; touch acme.json ; chmod 600 acme.json
2. add ./environment.sample to /etc/environment and replace dummy values
3. docker-compose -p traefik up -d
4. docker logs -f traefik
5. wait... http://traefik.yourdomain.com becomes available
6. add services by adding labels as in template below (and comment out exposed ports in service yml file)

# other services in this repo
Run under their own sub-domains, with labels in place.
