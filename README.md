# dockerplayground
Setup a secure Docker playground on your own VPS. A work in progress but opened for feedback because I could not find another working example of Traefik 2.0 working with KeyCloak for authentication.

- Exposes any Docker service securely over HTTPS on its own sub-domain
- Automatically requests and renews LetsEncrypt certificates
- Authentication at the proxy-level with SSO to any service that supports OpenID, OAuth2.0 or SAML

The below steps will allow you to replicate my setup on your own VPS or home server running Docker. There is quite a bit of setup involved though so it is not merely a matter of editing the configuration file and running docker-compose.

## Acknowledgement
The following was essential in putting all of this together:
- [Traefik Tutorial: Treafik Reverse Proxy with LetsEncrypt for Docker](https://www.smarthomebeginner.com/traefik-reverse-proxy-tutorial-for-docker/) which - describing Traefik 1.x - is now out of date but got me set up,
- [Full Traefik with Keycloak Single Sign-On with Postgres db for LDAP capabilities](https://dockerquestions.com/2019/07/10/full-traefik-with-keycloak-single-sign-on-with-postgres-db-for-ldap-capabilities/) which again did not cover the new Traefik format but helped with everything else, and
- [The Traefik 2.0 Docs](https://docs.traefik.io/) full of examples of the new router-middleware-service approach to labels.

## Pre-requisites:
### Internet facing Linux VPS; I built on Ubuntu 18.04 LTS
Any stable distro will work; I'm sure you can even get it all working on top of Windows but why.
Make sure to [set-up automatic security updates](https://phoenixnap.com/kb/automatic-security-updates-ubuntu), [enable a firewall](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04) to only allow ports 22,80 and 443 and [secure SSH access with a certificate](https://www.cyberciti.biz/faq/ubuntu-18-04-setup-ssh-public-key-authentication/).

### Domain name under your control and a Cloudflare account
Traefik can expose services on a *subdirectory-per-service* or *subdomain-per-service* basis. The first method is the easiest to set-up since a single SSL cerfiticate will do but not all services work well like this (looking at you, NextCloud). We will thus be working with sub-domains and a good way to secure these is using a wildcard certificate authorised for each service.

To make this work automatically with Traefik, we need to host our domain with a provider allowing API access and whilst there's a number of them out there; [Cloudflare's free subscription](https://dash.cloudflare.com/sign-up?pt=f&utm_referrer=https://www.cloudflare.com/) will do just fine. Cloudflare's API allows Traefik to add a TXT record to your domain records which LetsEncrypt uses to verify the requested certificate update.

1. Apply for a *free subscription* with Cloudflare, add your domain and either transfer it to them entirely or just point it to the Cloudflare nameservers listed in your control panel.
2. Verify, save, open your domain page and move to the *"DNS"* tab. You will, at the very least, need:
   
   - **A-record** pointing at the public IP address of your Docker host, and 
   - **CNAME-record** named "**\***" (wildcard) pointed at your main domain. 
   
3. Select the *"SSL/TLS"* tab and make sure **full encryption** is selected.
4. Click on your profile picture and select "My Profile". Select the *"API Tokens"* tab where you will find your *Global API key*. Note this down; this will be placed in the environment file as **CLOUDFLARE_API_KEY**. 

### Docker CE and Docker-Compose
The below assumes you are running Ubuntu 18.04 LTS. For other distros [DuckDuckGo your specifics](https://duckduckgo.com/?q=setup+docker+and+docker-compose&t=h_&ia=web). We need *Docker CE* running, *Docker-Compose* installed and your user added to the *docker* group.
```
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce
sudo systemctl status docker
## find latest version of docker-compose at https://github.com/docker/compose/releases
## install docker-compose, replace [release] with version found
sudo curl -L https://github.com/docker/compose/releases/download/[release]/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

## add user to docker group to simplify admin
sudo usermod -aG docker ${USER}
```

# Preparing the environment
Move to the `dockerplayground` directory, or wherever you cloned this repo into. Then,

1. Prepare the **data** directory and an empty *letsencrypt certificate-config*, because docker can only create directories, not files, and we do not want either owned by root.
```
mkdir data
mkdir data/letsencrypt
touch data/letsencrypt/acme.json
chmod 600 data/letsencrypt/acme.json
```

2. Open `environment.sample` and rename variables to reflect your situation. Use strong, random passwords. Save the file and either rename it to `.env` or append to `/etc/environment` (in which case don't forget to logout and on again to update your environment).

3. Run Docker-compose to initialise the *traefik-keycloak* stack:
```
docker-compose -p traefik-keycloak up -d
```
4. Check how setup runs in the container logs, and when satisfied escape with \[Ctrl]+\[C].
```
docker logs -f traefik
```
5. wait... until https://keycloak.yourdomain.com becomes available. If at first you get a certificate warning, wait some more and refresh. To prevent being locked-out, a delay of 5 minutes is set between two certificate requests to LetsEncrypt.

# Configuring Keycloak

1. On the Keycloak welcome screen open the *Admin Console* and logon with the credentials you put in the environment file.

2. Keeping it simple for now, we will use the master realm. Under *realm settings*, 

   * Edit the *Login* settings to your liking, set Require SSL to "external requests".

   * Complete the *email* settings with a sendgrid or other off-site smtp server

   * Under *security defenses* switch on "Brute Force Detection".

3. Select the *clients* menu, and create a new client

   * *client id* , *name* and *description* can all be `traefik`.

   * As *Valid Redirect URL*, either enter all the individual service URLs or a wildcard. 

   * Set the *Access Type* to "Confidential" and save to get a "credentials" tab.

   * On the Credentials tab, select "Client Id and Secret" and note down the *Secret* listed. You will place this in the environment file as `SVC_AUTHGATE_SECRET`

4. Save, and select the "Client Scopes" menu. Create a new one.

   * Enter `traefik` as *name* and `OpenID-connect` as protocol. Select *Include in token scope*.

   * Click on "Mappers" and create a new mapper called `traefik`.

   * Set the *Mapper Type* to `Audience` and select `traefik` as *Included Client Audience*.

   * Select *Add to ID token* and *Add to Access token* and save.

5. Go back to the "Clients" menu, "Client Scopes" tab, and add `traefik` to the *Assigned Default Client Scopes*.

6. Don't forget to create yourself a user, if - as recommended - self registration is disabled for the master realm, and set a good password. This is also a good time to setup 2FA for your user.

# Setup the forward-auth service

With the generated client secret placed in the .env file (or added to the /etc/environment file, and dont forget the logout logon), we need to update the traefik-forward-auth container. The below is a bit sloppy but it works.
```
docker-compose -p traefik-keycloak down
docker-compose -p traefik-keycloak up -d
```

Check if things work by opening the Traefik dashboard at https://traefik.yourdomain.com since that one is secured with the new KeyCloak SSO.

# Onwards, additional services
You can now add any service by adding labels to your service's .yml file (refer to the [sample-service.yml](https://github.com/echtniels/dockerplayground/blob/master/sample-service.yml) and don't forget to comment out exposed ports; traefik does all the exposing). 

If a service is public, has its own authentication or can be integrated to Keycloak using OpenID or SAML, you're set. In other cases, simply remove the comment marks from the auth-forward labels in the sample to make sure all traffic goes through Keycloak.

As you spin new services up and down, Traefik will set up a route to them, and handle certificates automatically.

## sample services
The repo holds a few samples of useful services I rely on:
- [nextcloud.yml](https://github.com/echtniels/dockerplayground/blob/master/nextcloud.yml) spins up a NextCloud instance with a MariaDB back-end.
- [onlyoffice.yml](https://github.com/echtniels/dockerplayground/blob/master/onlyoffice.yml) enriches NextCloud with a web-based office suite.
- [teedy.yml](https://github.com/echtniels/dockerplayground/blob/master/teedy.yml) the Teedy Document Management System.
- [searx.yml](https://github.com/echtniels/dockerplayground/blob/master/searx.yml) the SearX Meta-Search-Engine.
- [netdata.yml](https://github.com/echtniels/dockerplayground/blob/master/netdata.yml) web-based statistics for your Docker host.
- [portainer.yml](https://github.com/echtniels/dockerplayground/blob/master/portainer.yml) is a web-frontend to Docker.

And my favorite:
- [code-server](https://github.com/echtniels/dockerplayground/blob/master/code-server.yml) a perfect VS-Code environment in your browser.

You can spin up any of these services with `docker-compose -f projectname.yml -p projectname up -d` .
