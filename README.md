# dockerplayground
Setup a secure Docker playground on your own VPS. Work in progress but opened because I could not find another working example of Traefik 2.0 working with KeyCloak for authentication.

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

With the generated client secret placed in the .env file (or added to the /etc/environment file, and dont forget the logout logon),
```
docker-compose -p traefik-keycloak down
docker-compose -p traefik-keycloak up -d
```
and let compose re-create the forward-auth service with the right credentials.

Check if things work by opening the Traefik dashboard at https://traefik.yourdomain.com since that one is secured with the new KeyCloak SSO.

# Onwards, additional services
You can now add any service by adding labels (refer to the template .yml, and don't forget to comment out exposed ports in service yml file, traefik does all the exposing) to your docker-compose files. The repo holds a few samples.
