# rev3.tk

Le but : déployer facilement un stack NextCloud / Plex / Jellyfin / daapd / Deluge / Hydra2 / ..., avec Certificats HTTPS (nom de domaine obligatoire pour Let's Encrypt du coup, on peut prendre un domaine gratuit en .tk). Avec en option GlusterFS pour gérer une Pool de Disques et OpenLDAP pour les utilisateurs (lier Jellyfin, Nextcloud, et le domaine en un cookie mais voir pour les performances, pareil avec GlusterFS).

Niveau apprentissage c'est surtout pour du Docker, faire quelques images, pourquoi pas passer sur Kubernettes plus tard approfondir Ansible, et faire tout les tests d'audit possibles.

# Good Start

    $ sudo raspi-config #change hostname, user password, memory split to 16mb cause no gui and update, SSH is already allowed
    $ sudo passwd root #set the root password
    $ sudo nano /etc/ssh/sshd_config #change ssh port/listen adress
    $ sudo adduser <user>
    $ sudo visudo
    
    <user>  ALL=(ALL:ALL) ALL
    
    $ sudo apt update && sudo apt upgrade -y #update, upgrade
    $ sudo exit #login to the new user

    $ sudo deluser -remove-home pirate # pirate is the default user on hypriot
    $ sudo usermod -a -G docker $USER
    $ sudo reboot #need reboot after add <user> to docker group

# Disk and GlusterFS Pool

## Disk Format

    $ sudo blkid -o list
    $ sudo fdisk /dev/sda
   
    d : delete all partitions
    n : create new partition
    w : make change and exit
    
    $ sudo mkfs.ext4 /dev/sda1
    
## Mount

    $ sudo mkdir -p /data/brick1
    $ sudo mount -t ext4 -o defaults /dev/sda1 /data/brick1
    
Auto-mount at each boot :

    $ sudo blkid -o list # => UUID : ...
    $ sudo nano /etc/fstab
    
    UUID=... /data/brick1 ext4 defaults 0

Set the permissions :

    $ sudo chown $USER:$USER -R /data/brick1

Safely eject disks and reboot to test :

    $ sudo apt-get install udisks -y
    $ sudo udisks --unmount /dev/sda1
    $ sudo umount -l /data/brick1 ## if the disk is busy
    $ sudo udisks --detach /dev/sda

## GlusterFS

    node05$ sudo apt install glusterfs-server -y
    node06$ sudo apt install glusterfs-server -y

    node05$ sudo nano /etc/hosts ## On Hypriot the file to modifie is in /etc/cloud/templates
    
    127.0.1.1       node05.storage node05
    192.168.0.215   node06.storage node06

    node06$ sudo nano /etc/hosts
    
    127.0.1.1       node06.storage node06
    192.168.0.214   node05.storage node05

    node05$ sudo gluster peer probe node06
    node06$ sudo gluster peer probe node05
    
    node05$ sudo gluster peer status # => State: Peer in Cluster (Connected)
    node06$ sudo gluster peer status # => State: Peer in Cluster (Connected)

    node05$ mkdir /data/brick1/gv0
    node06$ mkdir /data/brick1/gv0

    node05$ sudo gluster volume create gv0 replica 2 node05:/data/brick1/gv0 node06:/data/brick1/gv0
    node05$ sudo gluster volume start gv0 # start/stop/status/delete
    node05$ sudo gluster volume info # => Status : started

    node05$ sudo gluster volume set gv0 auth.allow 192.168.0.210,192.168.0.211,192.168.0.212,192.168.0.213,192.168.0.214,192.168.0.215

Some tests on the Gluster Hosts :

    node05$ sudo mount -t glusterfs node05:/gv0 /mnt
    node05$ sudo su
    node05# echo "Hello World" > /mnt/hello.txt && exit
    node06$ sudo mount -t glusterfs node06:/gv0 /mnt
    node06$ cat /mnt/hello.txt # => Hello World
    
Automount volumes on boot on the Gluster hosts :

    node05+06$ sudo nano /etc/fstab

    localhost:/gv0 /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0

If the disk is not mounting on restart try this line instead in fstab :

    localhost:/gv0 /mnt glusterfs defaults,_netdev,noauto,x-systemd.automount,backupvolfile-server=localhost 0 0

Now to setup each Client :

    node0x$ sudo apt-get install glusterfs-client -y
    node0x$ sudo nano /etc/hosts
    
    192.168.0.214  node05.storage node05
    192.168.0.215  node06.storage node06
    
    node0x$ sudo nano /etc/fstab
    
    node05:/gv0 /mnt glusterfs defaults,_netdev,backupvolfile-server=node06 0 0
 
If the disk is not mounted on the client on reboot try this line instead :

    node05:/gv0 /mnt glusterfs defaults,_netdev,noauto,x-systemd.automount,backupvolfile-server=node06 0 0

Some tests Client+Server :
 
    node05$ sudo udisks --unmount /dev/sda1
    node05$ sudo umount -l /data/brick1 ## if the disk is busy
    node05$ sudo udisks --detach /dev/sda
    node01$ cd /mnt
    node01$ sudo mkdir hello
    node02$ cd /mnt && ls # => hello
    node06$ cd /data/brick1/gv0 && ls => hello
    node05$ sudo reboot
    node05$ cd /data/brick1/gv0 && ls => hello

Here it's like the first disk, actually mounted on all node, was disconnected, we see that we can continue to read/write on the /mnt dir and it's the same for all the rest of the servers. When the disk is reconnected, all the data are restored. To cleanly stop the service before umount the disk :

    $ sudo service glusterfs-server stop

# Three

You can adopt the following three for the mediaserver directories, otherwise you may want to change some values in docker-compose.yml.

    /mnt/
    ├── config/
    │   ├── plex/
    │   ├── deluge/
    │   ├── hydra2/
    │   ├── cloud/
    │   ├── nginx/
    │   ├── letsencrypt/
    │   └── .../
    ├── data/
    │   ├── mysql/
    │   ├── redis/
    │   ├── cloud/
    │   │   ├── shared_media/
    │   │   │   ├── Dépot/
    │   │   │   ├── Films/
    │   │   │   ├── Musiques/
    │   │   │   ├── Series/
    │   │   │   └── Téléchargements/
    │   │   ├── toto/
    │   │   ├── tata/
    │   │   └── .../
    │   └── musictracker/
    └── www/
        ├── mediaserver.tk/
        │   ├── cloud/
        │   ├── ampache/
        │   ├── html/
        │   └── .../
        ├── example.tk/
        ├── radio.xyz/
        └── .../

# Docker Swarm

On the master :

    $ docker swarm init 
    
We have in return a command to enter on the workers instances :
    
    $ docker swarm join --token SWMTKN-1-17fw99pvvhyf8og91i09ujgy3upzg0nnce3hav0ecjylr2anr3-c7xgby5qequwa0w679q9gx5it 192.168.0.210:2377

To verify, on the master :

    $ docker node ls
    
Set more masters :

    $ docker node promote <node_id>

Set a Network for the Project and linked Containers :

    $ docker network create -d overlay --attachable rev3

# Images

# Image Build for Armv7

## Pull

    $ docker pull adann0/certbot:armv7
    $ docker pull adann0/openldap:armv7
    $ docker pull adann0/phpldapadmin:armv7

## Build

### Enable Docker Experimental Features

Client :

    $ cd
    $ nano .docker/config.json # ,"experimental":"enabled"

Server :

    $ echo {"experimental":true} >> /etc/docker/daemon.json

### (Ok The Problem)

    $ docker manifest inspect --verbose balenalib/raspberrypi3-debian # => Architecture : amd64

Is build on top of :
    
    $ docker manifest inspect --verbose arm32v7/debian # => Architecture : arm

### Certbot

Just build the image on the Raspberry :

    $ git clone https://github.com/certbot/certbot.git
    $ cd certbot
    $ docker build -t "certbot:armv7" .
    $ cd ..

### OpenLDAP + phpLDAPadmin

   https://github.com/adann0/openldap-armv7

### Adminer

    $ git clone https://github.com/TimWolla/docker-adminer.git &&
    cd docker-adminer/4 &&
    docker build -t "adminer:armv7" .

### Push

To push Images on Docker Hub :

    $ docker login --username=<hub_repo> --email=<hub_email>
    $ docker images
    $ docker tag <image_id> <hub_repo>/<image>:<tag>
    $ docker push <hub_repo>/<image>:<tag>

Then :

    $ docker manifest inspect --verbose <hub_repo>/<image>:<tag> # => Architecture : arm
    $ docker pull <hub_repo>/<image>:<tag>
    
# Let's Encrypt

    $ git clone https://github.com/adann0/rev3.tk.git
    $ cd certbot
    
_Change the value "example.com" by our domain name in nginx.conf._

    $ nano nginx/nginx.conf
    $ docker-compose up -d

Verify that you can access to your website. Then change the value again and run the command :

    $ sudo docker run -it --rm \
    -v /mnt/config/letsencrypt/etc:/etc/letsencrypt \
    -v /mnt/config/letsencrypt/var:/var/lib/letsencrypt \
    -v /mnt/www/example.com:/data/letsencrypt \
    -v /mnt/config/letsencrypt/logs:/var/log/letsencrypt \
    certbot:armv7 \
    certonly --webroot \
    --email mail@example.com --agree-tos --no-eff-email \
    --webroot-path=/data/letsencrypt \
    -d example.com -d www.example.com

Once we have the certificates we can stop the temporary website :

    $ docker-compose down
    $ cd ../prod
    
Now for the production again change the values in nginx.conf :

    $ nano nginx/nginx.conf
    $ mv nginx/nginx.conf /mnt/config/nginx/
    
And make a certificate :

    $ mkdir -p /mnt/config/ssl/certs/
    $ sudo openssl dhparam -out /mnt/config/ssl/certs/dhparam-2048.pem 2048
    
To test the site in HTTPS :

    $ docker-compose up -d
    $ docker-compose down

Now to make the renewall of the certificate (on the two masters) :

    $ sudo crontab -e
    
    0 23 * * * docker run --rm -it --name certbot -v "/mnt/config/letsencrypt/etc:/etc/letsencrypt" -v "/mnt/config/letsencrypt/lib:/var/lib/letsencrypt" -v "/mnt/www/rev3.tk:/data/letsencrypt" -v "/mnt/config/letsencrypt/logs:/var/log/letsencrypt" certbot:Dockerfile renew --webroot -w /data/letsencrypt --quiet && docker kill --signal=HUP production-nginx-container

# Docker Secret

    $ printf "This is a secret" | docker secret create my_secret_data -

# Deploy

    $ docker service create --name registry --publish published=5000,target=5000 registry:2

    $ docker-compose up -d
    $ docker-compose ps
    $ docker-compose down --volumes

    $ docker-compose push

    $ docker stack deploy -c docker-compose.yml rev3tk
    $ docker stack services rev3tk
    $ docker stack rm rev3tk
    $ docker service rm registry

Portainer is included into the final docker-compose.yml (made for the Swarm). From this docker-compose you can add the services by adding the services you want into the followed config files.

# Sources :

- GlusterFS :
  - https://medium.com/running-a-software-factory/setup-3-node-high-availability-cluster-with-glusterfs-and-docker-swarm-b4ff80c6b5c3
  - http://banoffeepiserver.com/glusterfs/set-up-glusterfs-on-two-nodes.html
  - https://docs.gluster.org/en/v3/Administrator%20Guide/Managing%20Volumes/
  - GlusterFS Fail to Mount on Boot with Fstab : https://serverfault.com/a/823582
  
- Let's Encrypt + Nginx + Docker + Swarm :
  - https://www.humankode.com/ssl/how-to-set-up-free-ssl-certificates-from-lets-encrypt-using-docker-and-nginx
  - https://medium.com/@rhrn/setup-front-docker-machine-in-swarm-mode-with-letsencrypt-and-registry-890a4a1df090

- Enable Docker Experimental Features : https://github.com/docker/docker-ce/blob/master/components/cli/experimental/README.md

- Scaling :
  - https://medium.com/brian-anstett-things-i-learned/high-availability-and-horizontal-scaling-with-docker-swarm-76e69845825e

