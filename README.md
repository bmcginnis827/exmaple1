
# Overview

This documentation is for installing Zulip ([https://zulip.com/](https://zulip.com/)) with Docker Compose on Ubuntu 22.04 LTS. Zulip needs to be able to send outgoing email and thus requires an SMTP account. A no-reply mailbox will be created in Mailcow for this purpose.

## Update Packages

Update installed software packages: `sudo apt update && sudo apt upgrade`

  

## Install fail2ban

Ban any IP addresses that repeatedly fail SSH authentication on port 22.

Install fail2ban: `sudo apt install fail2ban`

## Install UFW

Firewall for allowing incoming and outgoing connections.

Install UFW: `sudo apt install ufw`

Allow connections to SSH, HTTP and HTTPS ports: `sudo ufw allow 22 && sudo ufw allow 80 && sudo ufw allow 443`

Enable the rules: `sudo ufw enable`

## Install Docker & Docker Compose

Install required packages: `sudo apt-get install ca-certificates curl gnupg lsb-release`

Add Docker’s official GPG key: `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`

Add the stable repository: `echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

Install Docker Engine: `sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io`

Install Docker Compose: `curl -SL https://github.com/docker/compose/releases/download/1.29.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose && sudo chmod +x /usr/local/bin/docker-compose`

## DNS Entry

A subdomain needs to be added to the domain we are using for Zulip. **Make sure to disable Cloudflare proxy status for the record below (DNS only)!**

-   Add an `A record` that points to the IP address of the Zulip server:
-   Type: `A`
-   Name: `zulip`
-   IPv4: `45.56.72.106`
   
## Create a “noreply” Mailbox

A noreply user will need to be created in Mailcow for sending outgoing emails.

-   Login to Mailcow admin UI: [https://mail.fortdenali.com/](https://mail.fortdenali.com/)
-   At the top, click **Configuration** > **Mail Setup**
-   Click **Mailboxes**
-   Click the green **Add mailbox** button
-   Username: `noreply-zulip`
-   Click **generate password** and save it somewhere: `AnCHiTTO,56,#`
-   Click **Add**
    
We need to configure the noreply-zulip mailbox to reject all emails now.

-   Login to SOGo: [https://mail.fortdenali.com/SOGo/](https://mail.fortdenali.com/SOGo/)
-   Username: noreply-zulip@fortdenali.com
-   Password: one we generated earlier
-   Once logged in, click the **Gear icon** in the **top-left hand** area (next to your username)
-   Click **Mail** tab
-   Click **Filters** tab, then **Create Filter**
-   Name it: `Reject All`
-   Click the **drop down** and select **match all messages**
-   Click **Add an action**
-   In the new drop down, select **Send a reject message**
-   In the Message * field enter: `This mailbox does not accept incoming email.`
-   Click **OK**
-   **Important: Make sure to click the green floppy disk icon near the top-right hand area to save changes!**
    
## Get SSL Certificate

We will generate an SSL certificate for `zulip.fortdenali.com` and enable auto renewals.

Install certbot: `sudo apt install certbot`

Generate the certificate: `sudo certbot certonly --standalone --preferred-challenges http -d zulip.fortdenali.com`

Copy certificates to Zulip directory: `mkdir -p /opt/docker/zulip/zulip/certs && cp /etc/letsencrypt/live/zulip.fortdenali.com/fullchain.pem /opt/docker/zulip/zulip/certs/zulip.combined-chain.crt && cp /etc/letsencrypt/live/zulip.fortdenali.com/privkey.pem /opt/docker/zulip/zulip/certs/zulip.key`


## Zulip Configuration

Root access is required by Zulip, so we will just use the root user.

Clone the repo: `git clone https://github.com/zulip/docker-zulip.git /home/zulip-docker && cd /home/zulip-docker`

We need to edit docker-compose.yml: `nano docker-compose.yml`

The following settings below need to be set/changed:

-   `POSTGRES_PASSWORD:`
	- `ri4nwhvt739nvty7qvg`
-   `MEMCACHED_PASSWORD:`
	- `9hqvyt7489qytv7qtv5qy5`
-   `RABBITMQ_DEFAULT_PASS:`
	- `4vqht879hyqvt7owt5y`
-   `REDIS_PASSWORD:`
	- `yt674o2yht7hoyttwtfw`
-   Delete this line under ports: `-”80:80”`
-   `SSL_CERTIFICATE_GENERATION:`
	- Change from `self-signed` to `NO`
-   `SECRETS_email_password:`
	- This is the password of the noreply-zulip user mailbox we generated earlier in Mailcow" `AnCHiTTO,56,#`
-   `SECRETS_rabbitmq_password:`
	- Make the same as `RABBITMQ_DEFAULT_PASS`
-   `SECRETS_postgres_password:`
	- Make the same as `POSTGRES_PASSWORD`
-   `SECRETS_memcached_password`:
	- Make the same as `MEMCACHED_PASSWORDS`
-   `SECRETS_redis_password:`
	- Make the same as `REDIS_PASSWORD`
-   `SECRETS_secret_key:`
	- Create a long, random password: `9yqv7534nntop578w9yt5wiyt572yt5o4iwtu524hv7t5o22y64345r32ftr42t542`
-   `SETTING_EXTERNAL_HOST:`
	- This will be the subdomain you want to run zulip on: `zulip.fortdenali.com`
-   `SETTING_ZULIP_ADMINISTRATOR:`
	- Enter an email address you want errors/support emails to go to: `admin@fortdenali.com`
-   `SETTING_EMAIL_HOST:`
	- This will be the Mailcow hostname: `mail.fortdenali.com`
-   `SETTING_EMAIL_HOST_USER:`
	- This will be the noreply-zulip user we created in Mailcow: `noreply-zulip@fortdenali.com`
    
**Double check you entered everything correctly. If not, when you startup up Zulip for the first time you will encounter errors/things not working!**

## Start & Build Zulip

Let’s now build and startup all the containers that will run Zulip.

Make sure we are in the directory of the zulip docker-compose.yml file: `cd /home/zulip-docker`

Pull the images: `docker-compose pull`

Start the containers: `docker-compose up -d`

Verify everything is working at: [https://zulip.fortdenali.com](https://zulip.fortdenali.com)

## Setup Organization

We now need to create an initial organization for Zulip.

Run the command: `docker-compose exec -u zulip zulip /home/zulip/deployments/current/manage.py generate_realm_creation_link`

A URL will be outputted. Visit to register your organization.

## Setup Auto SSL

Auto renewal of SSL certificates needs to be enabled.

Edit crontab: `crontab -e`
 - Press 1 and then Enter if asked

Enter this to renew the SSL certificate every 30 days: `0 0 1 * * certbot renew --force-renewal --quiet && cp /etc/letsencrypt/live/zulip.fortdenali.com/fullchain.pem /opt/docker/zulip/zulip/certs/zulip.combined-chain.crt && cp /etc/letsencrypt/live/zulip.fortdenali.com/privkey.pem /opt/docker/zulip/zulip/certs/zulip.key && docker exec zulip-docker-zulip-1 nginx -s reload`

## Systemd Service

A systemd service needs to be created to start Zulip on system reboot or startup.
  
Bring down the containers first: `cd /home/zulip-docker && docker-compose down`

Create the file for the service: `nano /etc/systemd/system/zulip.service`

Enter this into the file:

```
[Unit]
Description=Zulip
Wants=network-online.target
After=network-online.target
Requires=docker.service
After=docker.service

[Service]
Restart=always
ExecStartPre=/usr/local/bin/docker-compose -f /home/zulip-docker/docker-compose.yml down
ExecStart=/usr/local/bin/docker-compose -f /home/zulip-docker/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /home/zulip-docker/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```
 
Start the service: `systemctl start zulip.service`

Check the status: `systemctl status zulip.service`

Enable the service at boot: `systemctl enable zulip.service`
