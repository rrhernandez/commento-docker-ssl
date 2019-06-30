# Commento easy-install with Docker Compose and SSL

This repository has easy install instructions for installing the Self-hosting Commento server on your own server

## Prerequisites

* An VPS you own, Google Cloud Compute Engine free tier (512Mb RAM and 30 Gb disk space) should be fine.
* Install docker ce, instructions: https://docs.docker.com/install/linux/docker-ce/ubuntu/
* Install docker-compose, instructions: https://docs.docker.com/compose/install/
* Add DNS Record to register a domain name that will point to your Commento server: commento.mydomain.com
* A SMTP service account (for sending mail), mailgun.org for example
  
## First steps

Ensure your system is up to date:

```
sudo apt update && sudo apt upgrade
```

Your Commento site will serve its content over HTTPS, so you will need to obtain an SSL/TLS certificate. Use Certbot to request and download a free certificate from Let’s Encrypt:
```
sudo apt install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt update
sudo apt install certbot
sudo certbot certonly --standalone -d commento.mydomain.com
```
These commands will download a certificate to /etc/letsencrypt/live/commento.mydomain.com/ on your server.

## Install Commento

Clone repository:

```
git clone https://github.com/rrhernandez/commento-docker-ssl.git
cd commento-docker-ssl
```

Edit `docker-compose.yml`, replace `commento.mydomain.com` with your domain and insert a new database password where `your_database_root_password appears`, the values for `database__connection__password` and `MYSQL_ROOT_PASSWORD` should be the same.
The Docker Compose file create a bind mount so create a directory for that bind mount:

```
sudo mkdir -p /usr/share/nginx/html
```

Edit `default.conf` file (inside nginx directory), replace all instances of `mydomain.com` with your domain.

This configuration will redirect all requests on HTTP to HTTPS (except for Let’s Encrypt challenge requests), and all requests on HTTPS will be proxied to the commento service.

## Run and Test Your Site

From the `commento-docker-ssl` directory start the Commento server by running all services defined in the docker-compose.yml file:
```
docker-compose up
```
Check carefully all the messages, if there are no errors then you can `CTRL-C` and run again with the daemon option this time:
```
docker-compose up -d
```

## Complete the setup

To complete the setup process, navigate to `https://commento.mydomain.com` and follow [Commento docs](https://docs.commento.io/installation/self-hosting/register-your-website/)

Then embed the following piece of HTML in your website wherever you want Commento to load. You may want to do this at the bottom of each post.

```
<div id="commento"></div>
<script defer
  src="http://commento.mydomain.com/js/commento.js">
</script>
```

And we're done.

## Usage and Maintenance

Because the option restart: always was assigned to your services in your docker-compose.yml file, you do not need to manually start your containers if you reboot your Linode. This option tells Docker Compose to automatically start your services when the server boots.

### Update Commento

Your docker-compose.yml specifies the latest version of the Commento image, so it’s easy to update your Commento version:

```
docker-compose down
docker-compose pull && docker-compose up -d
```

### Renew your Let’s Encrypt Certificate
Open your Crontab in your editor:
```
sudo crontab -e
```

Add a line which will automatically invoke Certbot at 11PM every day. Replace commento.mydomain.com with your domain:
```
0 23 * * *   certbot certonly -n --webroot -w /usr/share/nginx/html -d commento.mydomain.com --deploy-hook='docker exec ghost_nginx_1 nginx -s reload'
```
Certbot will only renew your certificate if its expiration date is within 30 days. Running this every night ensures that if something goes wrong at first, the script will have a number of chances to try again before the expiration.

You can test your new job with the --dry-run option:
```
sudo bash -c "certbot certonly -n --webroot -w /usr/share/nginx/html -d commento.mydomain.com --deploy-hook='docker exec ghost_nginx_1 nginx -s reload'"
```