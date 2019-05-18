# docker-nginx-letsencrypt

    $ git clone https://github.com/adann0/docker-letsencrypt.git && cd docker-letsencrypt/dev
    
_Change the value "example.com" by our domain name in nginx.conf and docker-compose.yml._

    $ nano nginx/nginx.conf
    $ nano docker-compose.yml
    
Then :

    $ docker-compose up -d

Verify that you can access to your website. Then change the example values again, make directory and subdirectory for letsencrypt and run the command, if you use Armv7 you can use this image of Certbot, if you're on Amd64 replace it with the official one :

    $ sudo docker run -it --rm \
    -v /path/to/<your_directory>/letsencrypt/etc:/etc/letsencrypt \
    -v /path/to/<your_directory>/letsencrypt/var:/var/lib/letsencrypt \
    -v /path/to/<your_directory>/letsencrypt/logs:/var/log/letsencrypt \
    -v /path/to/docker-letsencrypt/dev/html:/data/letsencrypt \
    adann0/certbot:armv7 \
    certonly --webroot \
    --email mail@example.com --agree-tos --no-eff-email \
    --webroot-path=/data/letsencrypt \
    -d example.com -d www.example.com

Once we have the certificates we can stop the temporary website :

    $ docker-compose down
    
Now for the production, again change the values in nginx.conf and docker-compose.yml :

    $ cd ../prod
    $ nano nginx/nginx.conf
    $ nano docker-compose.yml

    
And make a certificate :

    $ sudo openssl dhparam -out dhparam-2048.pem 2048
    
To test the site in HTTPS :

    $ docker-compose up -d
    $ docker-compose down

Now to make the renewall of the certificate (on all masters node) :

    $ sudo crontab -e
    
    0 23 * * * docker run --rm -it --name certbot -v "/mnt/config/letsencrypt/etc:/etc/letsencrypt" -v "/mnt/config/letsencrypt/lib:/var/lib/letsencrypt" -v "/mnt/www/rev3.tk:/data/letsencrypt" -v "/mnt/config/letsencrypt/logs:/var/log/letsencrypt" adann0/certbot:armv7 renew --webroot -w /data/letsencrypt --quiet && docker kill --signal=HUP production-nginx-container

# Sources :

  - https://www.humankode.com/ssl/how-to-set-up-free-ssl-certificates-from-lets-encrypt-using-docker-and-nginx
  - https://medium.com/@rhrn/setup-front-docker-machine-in-swarm-mode-with-letsencrypt-and-registry-890a4a1df090
