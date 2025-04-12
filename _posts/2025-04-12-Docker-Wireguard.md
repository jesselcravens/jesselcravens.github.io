---
layout: post
title:  Deploying Wireguard with Docker
date:   2025-04-12 10:46:29 -0500
categories: Linux
---
My Ubiquiti Edge router did not support a modern VPN solution such as openvpn and only supported pptp which I understand has security issues. 
I choose Wireguard as it is faster and easier to configure than openvpn. The design choice will be to forward wireguard traffic to an internal ubuntu machine. 

On the Ubuntu Machine:

   Install some pre-reqs, Docker, and Docker Compose: 
   
    sudo apt install apt-transport-https ca-certificates curl                                                                                                                                                                  gnupg     lsb-release     sudo -y
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dear                                                                                                                                                             mor -o /usr/share/keyrings/docker-archive-keyring.gpg
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyrin                                                                                                                                                             g.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sud                                                                                                                                                             o tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt update
    sudo apt install docker-ce docker-ce-cli containerd.io -y
    sudo systemctl start docker
    sudo systemctl enable docker
    docker --version
    sudo curl -L "https://github.com/docker/compose/releases/download/$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r .tag_name)/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose

Check that docker and docker compose are installed:

    docker version
    docker compose version

Create a directory for wireguard stuff : 

    mkdir ~/wireguard
    cd ~/wireguard

Create the docker compose yaml file:

    nano docker-compose.yml

With the following contents: 

    version: '3.8'
    
    services:
      wireguard:
        image: lscr.io/linuxserver/wireguard
        container_name: wireguard
        cap_add:
          - NET_ADMIN
          - SYS_MODULE
        environment:
          - PUID=1000
          - PGID=1000
          - SERVERURL=<server hostname goes here> 
          - SERVERPORT=51820
          - PEERS=1
          - ALLOWEDIPS=0.0.0.0/0
        volumes:
          - ./config:/config
          - /lib/modules:/lib/modules
        ports:
          - "51820:51820/udp"
        sysctls:
          - net.ipv4.conf.all.src_valid_mark=1
        restart: unless-stopped
  

*note: Change the PEERS value for more than one peer. Change the SERVERURL for your use case*

Allow the port for this through the firewall if you are using one: 

    sudo ufw allow 51820/udp

Allowing this port to forward from the Edge router to the machine host IP that is hosting this. I did this in EdgeMax. In the FireWall/Nat Groups section, in the Port Forwarding section, I added a rule to forward the original port 51820 UDP to the host machine to port 51820. 

Start the docker container: 

    sudo docker compose up -d

Look at the logs for the QR codes if you want to use them: 

    sudo docker logs -f wireguard

To look at the peer config files themselves, start a bash script into the container and look at the peer files: 

    sudo docker exec -it wireguard bash

While inside the container, navigate to the config directories: 

    cd /config

The peer file directories are listed as "peer1", "peer2", etc. You can find peer1 as "/config/peer1/peer1.conf". Copy the contents for client configuration

    less /config/peer1/peer1.conf

Once you have all your configs saved and recorded, exit container. 

    exit

Now go configure your clients with the config files. The QR codes are really nice for android setup. But you will need to maximize the ssh client window for best results.


