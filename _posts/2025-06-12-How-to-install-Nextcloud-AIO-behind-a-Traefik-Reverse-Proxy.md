---
layout: post
title: How to Install Nextcloud AIO Behind a Traefik Reverse Proxy
description: Learn how to install and configure Nextcloud AIO (All-in-One) behind a Traefik reverse proxy using Docker.
date: 2025-06-12 08:00:00 +0200
tags: [Self-Hosting]
tags_color: '#4ee30c'
image: '/images/2025-06-06-Securing-Your-Self-Hosted-Mail-Server/Preview.png'
toc: true
featured: true
---

Setting up your own private cloud has never been easier, thanks to Nextcloud AIO (All-in-One) and the flexibility of Docker. But to run it securely and efficiently—especially with HTTPS—you'll want to place it behind a reliable reverse proxy like Traefik. In this guide, we’ll walk you through deploying Nextcloud AIO behind Traefik, enabling automatic TLS, clean domain routing, and better control over your self-hosted infrastructure.

## **Prerequisites**

* A fresh and clean Ubuntu VM (IP in my case: 10.10.10.50)
* For this guide i have a second virtual hard drive connected for the Nextcloud data. This for easy management and backup.
* Traefik proxy server that handles certificates.
* Subdomain pointing to the external IP of the Traefik server (e.g. storage.opentechpulse.org)
<br><br>

## **Configure Traefik**

You need to add the correct config in Traefik that handles the incoming request for, in my case, https://storage.opentechpulse.org

In the "routers:" section add the following entry:

```yaml
  routers:
    nextcloud:
      entryPoints:
        - "https"
      rule: "Host(`storage.opentechpulse.org`)"
      middlewares:
        - default-security-headers
        - https-redirectscheme
      tls: {}
      service: nextcloud
```

In the "services:" section add the following entry:

```yaml
  services:
    nextcloud:
      loadBalancer:
        servers:
          - url: "http://10.10.10.50:11000" # Replace IP with your Nextcloud VM
        passHostHeader: true
```

## **Install Docker on the Nextcloud VM**

This can be done with one simple command:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

## **Change permissions on the data drive**

I assume you have the second drive already mounted somewhere, for me i mounted this on /mnt/nextcloud-data

You have to change some permission for Nextcloud to be able to use the drive:

```bash
sudo chown www-data:www-data /mnt/nextcloud-data
sudo chmod 750 /mnt/nextcloud-data
```

## **Create the docker-compose.yaml file**

```bash
cd
mkdir nextcloud-aio-docker
cd nextcloud-aio-docker
nano docker-compose.yaml
```

Paste the following:

```yaml
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    #init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer # This line is not allowed to be changed as otherwise AIO will not work correctly
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config # This line is not allowed to be changed as otherwise the built-in backup solution will not work
      - /var/run/docker.sock:/var/run/docker.sock:ro # May be changed on macOS, Windows or docker rootless. See the applicable documentation. If adjusting, don't forget to also set 'WATCHTOWER_DOCKER_SOCKET_PATH'!
    network_mode: bridge
    ports:
      #- 80:80 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
      - 8080:8080
      #- 8443:8443 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
    environment: # Is needed when using any of the options below
      APACHE_PORT: 11000 # Is needed when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
      APACHE_IP_BINDING: 0.0.0.0 # Should be set when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else) that is running on the same host. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
      NEXTCLOUD_DATADIR: /nextcloud-data # Allows to set the host directory for Nextcloud's datadir. ⚠⚠⚠ Warning: do not set or adjust this value after the initial Nextcloud installation is done! See https://github.com/nextcloud/all-in-one#how-to-change-the-default-location-of-nextclouds-datadir
      SKIP_DOMAIN_VALIDATION: true # This should only be set to true if things are correctly configured. See https://github.com/nextcloud/all-in-one?tab=readme-ov-file#how-to-skip-the-domain-validation

volumes: # If you want to store the data on a different drive, see https://github.com/nextcloud/all-in-one#how-to-store-the-filesinstallation-on-a-separate-drive
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer # This line is not allowed to be changed as otherwise the built-in backup solution will not work
```

Those are the required minimum to run Nextcloud AIO behind a reverse proxy. You can find all the options on the <a href="https://github.com/nextcloud/all-in-one/blob/main/compose.yaml" target="_blank">Github page</a>.

## **Run docker compose**

```bash
sudo docker compose up -d
```

![docker-compose](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/1-nextcloud-aio-docker-compose.png)

## **Goto internal URL**

Open a webbrowser and go to the internal url: https://10.10.10.50:8080

You will get a message about invalid certificate. This is normal because we are accessing the internal URL. You can accept this and go on.

![passphrase](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/2-nextcloud-aio-initial-setup.png)

{: .important }
Copy the Passphrase, this is only showed once here and you need it for the following step.

Click on the button "Open Nextcloud AIO login".

## **Login with passphrase**

Now you will get a page "Log in using your Nextcloud AIO passpgrase".

![passphrase-login](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/3-nextcloud-aio-initial-login-with-key-phrase.png)

Paste yoour copied passphrase and click "Log in".

## **Enter external domain**

In this page you need to enter your external domain, in my case this is storage.opentechpulse.org and click "Submit domain"

![external-domain](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/4-nextcloud-aio-initial-domain-check.png)

## **Optional containers**

On this page you can enable some optional containers if needed. Beware that you VM has enough recources to run them.

![optional-containers](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/5-nextcloud-aio-optional-containers-timezone.png)

You can just leave this default and click "Download and start containers".

The containers are downloaded and started. This could take a few minutes.

![optional-containers](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/6-nextcloud-aio-optional-containers-starting.png)

You can click the "Reload" button. Once all containers are downloaded and started you will get your Initial Nextcloud admin username and password:

![initial-username-password](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/7-nextcloud-aio-initialusername-and-password.png)

{: .important }
Copy the initial password, you will need this for logging in.

Click on the "Open your Nextcloud" button. Your Nextcloud will open with the external url (e.g. https://storage.opentechpulse.org) and with a SSL certificate from Traefik.

## **Login to your Nextcloud**

![login](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/8-nextcloud-aio-login.png)

Login with user "admin" and the initial password you copied.

When everything goes right it will open the dashboard:

![dashboard](/images/2025-06-12-How-to-install-Nextcloud-AIO-behind-a-Traefik-Reverse-Proxy/9-nextcloud-aio-dashboard.png)

***

By combining the power of Nextcloud AIO with the flexibility of Traefik, you’ve created a secure, self-hosted cloud environment with automated HTTPS and simplified routing. This setup not only gives you full control over your data but also scales easily with your infrastructure. Whether you're running a home lab or deploying for a small business, this approach offers a solid foundation for modern, privacy-focused cloud services.