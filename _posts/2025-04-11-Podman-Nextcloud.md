---
layout: post
title:  Deploying Nextcloud with Podman 
date:   2024-08-13 14:10:32 -0500
categories: Linux
---

I wanted to learn more about Podman which is an open source competitor of Docker and I felt Nextcloud would be a worthy challenge. These are the steps that I performed using new minimal installed Ubuntu Server 24.04.2. 

Get Podman installed on the server and some pre-reqs

    sudo apt update
    sudo apt upgrade -y
    sudo apt install podman podman-compose nano iptables curl

In order to pull Docker containers into Podman, you will need to modify the /etc/containers/registries.conf file. 

    sudo nano /etc/containers/registries.conf
Add the following line at the bottom of the file: 

    unqualified-search-registries=["docker.io"]
As I understand, there are Container Network Interface plugin updates that need to be performed or you might have Podman container networking issues. 
Determine the latest version of of the plugins from https://github.com/containernetworking/plugins/releases. At the time of writing, this was 1.6.2. Plug this value into the command below and replace as needed: 

    export CNI_PLUGIN_VERSION=v1.6.2
Detect the current architecture of the plugin needed:

    export ARCH_CNI=$( [ $(uname -m) = aarch64 ] && echo arm64 || echo amd64)
Download the plugin with these variables: 

    curl -L -o cni-plugins.tgz "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGIN_VERSION}/cni-plugins-linux-${ARCH_CNI}-${CNI_PLUGIN_VERSION}.tgz"
Backup the old plugins and move the new ones into place:

    mkdir ~/src/cni
    mkdir -p ~/src/cni
    sudo tar -C ~/src/cni -xzf cni-plugins.tgz
    sudo mv /usr/lib/cni /usr/lib/cni.old
    sudo cp -rf ~/src/cni/ /usr/lib/cni

Pull the Nextcloud docker container into Podman

    podman pull nextcloud
Pull MariaDB container into Podman

    podman pull mariadb
I wanted the persistent volumes in a certain mount point, so I defined them as:

    podman volume create   --opt type=none   --opt device=/data/nextcloud/app   --opt o=bind   nc-app
    podman volume create   --opt type=none   --opt device=/data/nextcloud/data   --opt o=bind   nc-data
    podman volume create   --opt type=none   --opt device=/data/nextcloud/db   --opt o=bind   nc-db
Create a network in podman with:

    podman network create nc-net
Create MariaDB deployment script `nano nano nc-podman-db-deploy.sh
`
With the following contents:

    #!/bin/bash
    
    podman run --detach \
      --env MYSQL_DATABASE=nextcloud \
      --env MYSQL_USER=nextcloud \
      --env MYSQL_PASSWORD=<Enter a unique Password> \
      --env MYSQL_ROOT_PASSWORD=<Enter a unique Password> \
      --volume nc-db:/var/lib/mysql \
      --network nc-net \
      --restart on-failure \
      --name nextcloud-db \
      docker.io/library/mariadb:latest
Create a NextCloud App deployment script `nano nc-podman-app-deploy.sh`

With the following contents:

    #!/bin/bash
    
    # Run Podman container with environment variables
    podman run --detach \
      --env MYSQL_HOST=nextcloud-db.dns.podman \
      --env MYSQL_DATABASE=nextcloud \
      --env MYSQL_USER=nextcloud \
      --env MYSQL_PASSWORD=<Use from MYSQL_PASSWORD above> \
      --env NEXTCLOUD_ADMIN_USER=<Create a new Username> \
      --env NEXTCLOUD_ADMIN_PASSWORD=<Create for the above> \
      --volume nc-app:/var/www/html \
      --volume nc-data:/var/www/html/data \
      --network nc-net \
      --restart on-failure \
      --name nextcloud \
      --publish 8080:80 \
      docker.io/library/nextcloud:latest
  
Execute the scripts: 

    bash nc-podman-db-deploy.sh
    bash nc-podman-app-deploy.sh

Edit the trusted domains to include your use case. I wanted to access this over my local network and with wireguard VPN tunnel only. I did not want this to work open to the internet. 
To edit the file inside the container: 

    podman exec -u root -it nextcloud nano /var/www/html/config/config.php

Edit this part (I added my local machines IP to #1): 

      'trusted_domains' => 
      array (
        0 => 'localhost',
        1 =>'192.168.68.159',
      ),
Restart your containers to make sure that settings were applied:

     podman container restart nextcloud-db
     podman container restart nextcloud

Access your new Nextcloud Service accessible through web browser. In my case, I could access through 192.168.68.159. Login with the Admin user and password set in `nc-podman-app-deploy.sh`

Lastly, Podman is different than Docker in regards to autostarting of containers. Docker uses the dockerd daemon that runs in the background and can restart and autostart containers. Podman is daemonless and this must be accomplished using systemd units. You will notice that a podman container will not start itself after a reboot unless you do these steps: 

Generate the service files: 

    podman generate systemd --name nextcloud-db --files --restart-policy=always
    podman generate systemd --name nextcloud --files --restart-policy=always

For a rootless deployment, these are not going to be created in system wide locations: 

    mkdir -p ~/.config/systemd/user
    mv container-nextcloud-db.service ~/.config/systemd/user/
    mv container-nextcloud.service ~/.config/systemd/user/

Create the services with these files: 

    systemctl --user daemon-reload
    systemctl --user enable container-nextcloud-db.service
    systemctl --user enable container-nextcloud.service

Start the Containers: 

    systemctl --user start container-nextcloud-db.service
    systemctl --user start container-nextcloud.service
Make sure the containers are running: 

    systemctl --user status container-nextcloud.service
    systemctl --user status container-nextcloud-db.service
Also, make sure that the containers are running in podman: 

    podman ps
Reboot and Test the containers are started after a reboot: 

    sudo reboot
    podman ps



