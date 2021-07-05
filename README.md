# Nextcloud dockerized

## Intro
This is my very personal log about my first steps with Docker in order to create a fresh Nextcloud instance.

This was performed on a decommissioned **Dell Powered R620** running [**RHEL 7**](https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.9/x86_64/product-software) *(thanks to the [No-Cost RHEL Developer Subscription](https://developers.redhat.com/blog/2016/03/31/no-cost-rhel-developer-subscription-now-available))*

The point was to install Nextcloud behind a full compliant *HTTPS reverse proxy*, get an *A+ grade* on [*Nextcloud security test*](https://scan.nextcloud.com/) and with a *custom data directory* **(without editing the nextcloud config.php neither the database !)** !

Nextcloud and other components are in separates containers

Then I fixed some known issues from Nextcloud 21...

## Disclaimer
😇 Like I'm a total *docker-n00b*, i know i could do very better... like writing only one **_docker-compose.yml_** for creating all my containers at once... but, this is working fine !

## Shopping list

I don't know why, but i more a nginx/mariadb/fpm guy 🤷‍♂️


➡️ So i need an **nextcloud:fpm** version *(currently nextcloud 21.0.3)* and a docker tuto with **nginx** and **maria-db**.
I found this excellent work by [Terence Chateigné](https://www.padok.fr/blog/nextcloud-docker) and go with it *(sorry, this is in 🇫🇷)*.

👉 https://github.com/terencec-padok/Nextcloud-in-Docker
>Nextcloud is in 5 containers :
>* nextcloud-db : MariaDB database
>* nextcloud-redis : File caching service
>* nextcloud-app : PHP server running the nextcloud code
>* nextcloud-web : NginX server
>* nextcloud-cron : maintainer servive


➡️ I went for [**Traefik**](https://doc.traefik.io/traefik/v2.0/) for the reverse proxy (based on @hyndruide advice) and the [Keith Thompson's instructions](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-v2-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04) were so clear that i could deploy my own Traefik instance according my needs. This all work with [**Let's Encrypt**](https://doc.traefik.io/traefik/v2.0/https/acme/) out of the box !

👉 https://www.digitalocean.com/community/tutorials/how-to-use-traefik-v2-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04

As i wrote theses lines, Traefik is on version v2.4.9, aka "Livarot" *(🧀 yes, for a french guy like me, versioning with cheese names is a very good reason to use it !)

⚠️ I use the **_image:latest_** for all containers, exept for Traefik:
> "Prefer a fixed version than the latest that could be an unexpected version. ex: traefik:v2.0.0" See [Traefik's documentation](https://doc.traefik.io/traefik/v2.0/getting-started/install-traefik/#use-the-official-docker-image)

## Time to work

### Prerequisites
>To complete this tutorial, you will need the following:
>
>1. One Linux-based server with a **sudo non-root user** and a firewall.
>2. **Docker** and **Docker Compose** installed on your server
>3. A domain and two A records,  *nextcloud.your_domain.tld* and *traefik.your_domain.tld* Each should point to the IP address of your server.
>
>*(yes this is from the Keith's blog, again 😃)*

### 1️⃣ Deploy Traefik

1. Upload to your server the **_traefik_** directory
2. Run a `<docker network create web>` to create the network that Traefik will use for receving requests from internet
3. Perform a `<sudo chmod 600 acme.json>` on the **traefik/conf/acme.json** file
4. Edit the **_traefik.toml_** file : add you email
5. Edit the **_traefik_dynamic.toml_** file :
* create your credentials using a hashed password with `<htpasswd>` or any [online tool](https://www.web2generators.com/apache-tools/htpasswd-generator)
* edit your host for the **Traefik webUI** with *traefik.your_domain.tld*
> 💡 PS: Your host has to be reachable from the internet on both ports 80 and 443, please consider forwarding theses ports in your ISP router...
6. Run a `<docker-compose up -d>`


### 2️⃣ Deploy Nextcloud

1. Upload to your server the **_nextcloud_** directory
2. Edit the **_nextcloud.env_** file : choose a strong passwords for your DB
4. Edit the **_docker-compose.yml_** file :
* Choose your **data directory** as a **docker volume** (The path has to be created first)
* Edit your domain name, like *nextcloud.mydomain.tld*
5. Run a `<docker-compose up -d>`
6. Go to *"nextcloud.mydomain.tld"* and create your nextcloud admin account

💡 As Terence's says :
> 🇫🇷 "C’est toujours une bonne pratique d’utiliser un compte non-privilégié pour effectuer vos opérations de tous les jours, il n’y a pas de raison de ne pas le faire ici non plus !"
> 
> 🇬🇧 "It's always a good practice to use a non-privileged account for your day-to-day transactions, there's no reason not to do that here either!
6. Edit the **_/nextcloud-data/config/config.php_** file :
* add lines about localisation, or whatever you want
* add the *"trusted-proxies"* by editing with your own traefik reverse-proxy IP (you will find on your traefik dashboard)
7. Perform a `<docker restart nextcloud-app>`

👉 The Magic is in the **_docker-compose.yml_** file : the native [**HTTP Strict Transport Security** (HSTS)](https://doc.traefik.io/traefik/middlewares/headers/) is made by the traefik's labels onto the nginx container.

### 3️⃣ Enjoy
![Capture d’écran 2021-07-05 à 18 53 37 1](https://user-images.githubusercontent.com/54755498/124501702-58e92380-ddc2-11eb-929c-2fb998c3827e.png)


## Bug fixing

I ran into two commons issues with this version of nextcloud. But i had to dig a lot to find a suitable solution. Here is mine :
### 1️⃣ php-imagick
![Capture d’écran 2021-07-05 à 17 07 19](https://user-images.githubusercontent.com/54755498/124501476-e8da9d80-ddc1-11eb-8628-0544e5f1f2fe.png)

> Module php-imagick in this instance has no SVG support. For better compatibility it is recommended to install it.

1. Run a `<docker exec -it nextcloud-app apt update>`
2. Then run `<docker exec -it nextcloud-app apt install imagemagick -y>`
3. And run `<docker restart nextcloud-web>`

### 2️⃣ */.well-known/webfinger* and */.well-known/nodeinfo*
![Capture d’écran 2021-07-05 à 17 13 23](https://user-images.githubusercontent.com/54755498/124501855-9a79ce80-ddc2-11eb-8494-655aecb6bd20.png)

> Your web server is not properly set up to resolve "/.well-known/webfinger". Further information can be found in the documentation." and/or the "Your web server is not properly set up to resolve "/.well-known/nodeinfo". Further information can be found in the documentation.

1. add to your **_nginx.conf_** file (near line 83 will be fine) :
```
    location = /.well-known/webfinger { return 301 https://$host/index.php/.well-known/webfinger; }
    location = /.well-known/nodeinfo { return 301 https://$host/index.php/.well-known/nodeinfo; }
```
💡 Found here : https://github.com/YunoHost-Apps/nextcloud_ynh/issues/198#issuecomment-490527306

⚠️ The **_nginx.conf_** file in this repo already contains these lines. For a *"clean"* **_nginx.conf_** file, use [this one](https://gist.github.com/terencec-padok/6f4413f3709a58e8110282c253e5cdff)

2. Then run `<docker restart nextcloud-web>`

## Bonus

### DynDNS

As my domains are registred at [Gandi](http://gandi.net)'s, i use [jbbodart/gandi-livedns](https://github.com/jbbodart/gandi-livedns) container. It works perfectly fine!
