# dockerplayground
Setup a secure Docker playground on your own VPS. A work in progress but opened for feedback because I could not find another working example of Traefik 2.0 working with KeyCloak for authentication.

- Exposes any Docker service securely over HTTPS on its own sub-domain
- Automatically requests and renews LetsEncrypt certificates
- Authentication at the proxy-level with SSO to any service that supports OpenID, OAuth2.0 or SAML

## credits
The following was essential in putting all of this together:
- [Full Traefik with Keycloak SSO](https://dockerquestions.com/2019/07/10/full-traefik-with-keycloak-single-sign-on-with-postgres-db-for-ldap-capabilities/)

## Pre-requisites:
### Internet facing Linux VPS; I built on Ubuntu 18.04 LTS
Any stable distro will work. Make sure to enable automatic security updates, enable a firewall to only allow ports 22,80 and 443 and secure SSH access with a certificate.

### Domain name under your control and a Cloudflare account
Traefik can expose services on sub-directories or sub-domains of your main domain. Chosing the sub-directory method is easy since one SSL cerfiticate will do, but not all services work well like this. We will thus be working with sub-domains and a good way to secure these is using a wildcard certificate authorised for each service. 

To make this work automatically with Traefik, we need to host our domain with a provider allowing API access and whilst there's a number of them out there; [Cloudflare's free subscription](https://dash.cloudflare.com/sign-up?pt=f&utm_referrer=https://www.cloudflare.com/) will do just fine.

1. Apply for a free subscription with Clourflare, add your domain and either transfer your domain to them entirely or just point it to the Cloudflare DNS server listed in your control panel.
2. Verify, save, open your domain page and click the "DNS" tab. You will, at the least, need an *A-record* pointing at the public IP address of your Docker host, and a *CNAME-record* named "\*" (wildcard) pointed at your main domain. 
3. click on your profile picture and select "My Profile". Select the "API Tokens" tab where you will find your *Global API key*. Note this down, this will be placed in the environment file as CLOUDFLARE_API_KEY 

### Docker CE and Docker-Compose
The below assumes you are running Ubuntu 18.04 LTS. For other distros [DuckDuckGo your specifics](https://duckduckgo.com/?q=setup+docker+and+docker-compose&t=h_&ia=web); in general we need Docker CE running, Docker-Compose installed and your user added to the docker group.
```
## add supporting modules
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
## add repos GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
## add [stable] Docker repo for Ubuntu 18.04 LTS
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
## update APT packages list
sudo apt-get update
## install DockerCE
sudo apt-get install docker-ce
## test docker
sudo systemctl status docker
## find latest version of docker-compose at https://github.com/docker/compose/releases
## install docker-compose, replace [release] with version found
sudo curl -L https://github.com/docker/compose/releases/download/[release]/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
## test docker-compose
docker-compose --version
## add user to docker group to simplify admin
sudo usermod -aG docker ${USER}
```

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
5. wait... https://keycloak.yourdomain.com becomes available. If at first you get a certificate warning, wait some more and refresh.

# Configuring Keycloak

1. Once you get the Keycloak welcome screen (with a valid certificate), click to open the Admin Console and logon with the credentials you put in the environment file.

2. Keeping it simple for now, we will use the master realm. Under *realm settings*, 
2.1. Edit the *Login* settings to your liking, set Require SSL to "external requests".
2.2. Complete the *email* settings with a sendgrid or other off-site smtp server
2.3. Under *security defenses* switch on "Brute Force Detection".
3. Select the *clients* menu, and create a new client
3.1. *client id* , *name* and *description* can all be `traefik`.
3.2. As *Valid Redirect URL*, either enter all the individual service URLs or a wildcard. 
3.3. Set the *Access Type* to "Confidential" and save to get a "credentials" tab.
3.4. On the Credentials tab, select "Client Id and Secret" and note down the *Secret* listed. You will place this in the environment file as `SVC_AUTHGATE_SECRET`
4. Save, and select the "Client Scopes" menu. Create a new one.
4.1. Enter `traefik` as *name* and `OpenID-connect` as protocol. Select *Include in token scope*.
4.2. Click on "Mappers" and create a new mapper called `traefik`.
4.3. Set the *Mapper Type* to `Audience` and select `traefik` as *Included Client Audience*.
4.4. Select *Add to ID token* and *Add to Access token* and save.
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
You can now add any service by adding labels (refer to the template .yml, and don't forget to comment out exposed ports in service yml file, traefik does all the exposing) to your docker-compose files. The repo holds a few samples.
