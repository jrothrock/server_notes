Here are various notes I've made for creating a server. These rules are for Ubuntu running Nginx. 

## SECURING SERVER


1. Create droplet (sudo isn't neccessary for 1-6, as you're root but it's a good habit to get into.)

2. ```SSH into server - ssh root@{ip-address}```

3. ```sudo apt-get update; sudo apt-get upgrade -y```

4. ```sudo apt-get install fail2ban -y```

5. ```sudo adduser {username}```
    - fill in user info.

6. Add user to sudo group (two ways to do this, but the second one doesn't scale as well)
    - first way: ```sudo usermod -aG sudo {username}``` 
    - second way: ```sudo visudo```.
         - under ```root    ALL=(ALL:ALL) ALL``` add ```{username}    ALL=(ALL:ALL) ALL```
        
7. exit server

8. create new ssh key
    ```ssh-keygen -t rsa -b 4096```

    - FWIW, RSA can be cracked by a quantum computer, if you worry about that ish.

9. Copy key to server.
    - ```ssh-copy-id -i ~/.ssh/{name of key}.pub “username@ip_address”```

10. SSH into your server (not root)

11. ```sudo (nano | vim | vi) /etc/ssh/sshd_config```
    - change ssh ```port``` from 22 (this will just keep the logs a little cleaner)
    - find ```PermitRootLogin``` and change it to ```PermitRootLogin no``` 
    - find ```PasswordAuthentication``` and change it to ```PasswordAuthentication: no``` 
    - exit/save file and run: ```sudo service ssh restart```

12. Add fire wall rules - with ex. rules
    - ```sudo ufw allow 80``` - http
    - ```sudo ufw allow 443``` - https
    - ```sudo ufw limit ssh``` - rate limit SSH
    - ```sudo ufw allow 22/{your new ssh port}``` -ssh
    - ```sudo ufw allow from 15.15.15.0/24 to any port 5432``` - postgres from remote server, change subnet
    - ```sudo ufw enable```


13. Set up pre firewall rules (this will ghost the server - ping packets will be dropped)
    - ```sudo (nano | vim | vi) /etc/ufw/before.rules```
    - DROP everything related to all ICMP/Pinging - I believe there are 8-10 of these in total
        
        - these are usually the fourth and fifth blocks.

        - these two blocks have ```ICMP``` mentioned in the comments above them.

    - exit/save file and run: ```sudo ufw reload```

## Add SSL - Let's Encrypt  

1. ```sudo apt-get install letsencrypt``` 

2. ```sudo letsencrypt certonly --standalone --rsa-key-size 4096 --force-renew -d example.com -d www.example.com```
    - nginx needs to be turned-off

        -```sudo service nginx stop```

3. ```sudo crontab -e```
    - may need to choose editor

4. at the bottom of the file add: ```30 2 * * 1 sudo /usr/bin/letsencrypt renew --rsa-key-size 4096 >> /home/${username}/le-renew.log```
    - this will regen a key every monday at 2:30 AM.
    - you can try /var/log, but I've run into permission issues when using sudo
        - you could set this in the root cron though, just remove sudo

5. create DH key:
    - ```sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 3072```

## SECURE SSL
I may upload the actual nginx file later, but for now, I'll just add the neccessary parts.

```
 Vary Upgrade-Insecure-Requests;
 add_header X-Content-Type-Options nosniff;
 add_header X-Xss-Protection "1; mode=block";
 add_header Content-Security-Policy "default-src https: data: 'unsafe-inline' 'unsafe-eval'";
 add_header X-Frame-Options "SAMEORIGIN";
 server_tokens off;

 ssl on;
 ssl_certificate /etc/letsencrypt/live/{your site}/fullchain.pem;
 ssl_certificate_key /etc/letsencrypt/live/{your site}/privkey.pem;
 ssl_dhparam /etc/ssl/certs/dhparam.pem;

 # As of writing this, TSLv1.3 hadn't be released for Nginx, this has most likely changed.
 #ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
 ssl_protocols TLSv1.1 TLSv1.2;

 ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';

 ssl_prefer_server_ciphers on;
 ssl_ecdh_curve secp384r1;
 ssl_session_cache shared:SSL:10m;
 ssl_session_tickets off;
 add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload" ;
 ssl_session_timeout 2h;
 ssl_stapling on;
 ssl_stapling_verify on;
```

exit and restart nginx - `sudo service nginx restart`

Double check everything on [ssllabs.com](https://ssllabs.com) and [securityheaders.io](https://securityheaders.io)

More information can be found here: [https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)

## Tripwire
This is honestly a bit of a lengthy process. However, Justin Ellingwood has written a [great piece over at DO on it.](https://www.digitalocean.com/community/tutorials/how-to-use-tripwire-to-detect-server-intrusions-on-an-ubuntu-vps)

## Configure Postfix
To change Postfix from sending from user@domain-localhost, I found the best way is to change the hostname for Ubuntu

1. hostname yourdomain.com

2. ``sudo (nano | vim | vi) /etc/hostname`` and change it to yourdomain.com

3. You will have to exit your droplet and ssh back into it.

Your hostname will have changed

*** Note
With the above you will receive an email with your super user account. This isn't ideal. However, you can set up Postfix to use an email relay to send stuff from Google instead, see the Linode link at the bottom for instructions on how to do so.

## Get email notifications when a user/root is logged into
0. This is assuming that the tripwire has been set, forcing you to have downloaded a mail client - in the case above, it would be Postfix.

1. Let's start with root

    - ```sudo su```

    - ```vi /root/.bashrc```

    - At the bottom of the file add: ```mail -s "Alert: Root Access from `who | cut -d'(' -f2 | cut -d')' -f1`" your_email@domain.com```

2. Your user

    - `vi /home/${username}/.bashrc`

    - At the bottom of the file add: ```echo 'ALERT - Root Shell Access (ServerName) on:' `date` `who` | mail -s "Alert: Root Access from `who | cut -d'(' -f2 | cut -d')' -f1`" your_email@domain.com```

*** Note
With the above you will receive an email with your super user account. This isn't ideal. However, you can set up Postfix to use an email relay to send stuff from Google instead, see the linode link at the bottom for instructions on how to do so.

## Enable Automatic Updates

1. ```sudo apt-get install unattended-upgrades```
2. ```sudo vi /etc/apt/apt.conf.d/10periodic```
3. Update the following lines to resemble below:
```
    APT::Periodic::Update-Package-Lists "1";
    APT::Periodic::Download-Upgradeable-Packages "1";
    APT::Periodic::AutocleanInterval "7";
    APT::Periodic::Unattended-Upgrade "1";
```
4. ``` sudo vi /etc/apt/apt.conf.d/50unattended-upgrades```
5. Update the file to resemble below - currently it's just set to do security updates, but uncommenting the second line will include package updates.
```
    Unattended-Upgrade::Allowed-Origins {
        "Ubuntu lucid-security";
     // "Ubuntu lucid-updates";
    };
```

## Create an SSH config file - OSX

`touch ~/.ssh/config`

```sudo (nano | vim | vi) ~/.ssh/config```


Host {name} - ex. Personal

HostName {your ip address} - ex. 104.159.103.195 

Port {your ssh port} - ex. 22

User {your username} - ex. root (don't use root!)

IdentityFile ~/.ssh/example.key - the private key for the server

Example: 
```
Host Personal
HostName 104.159.103.195 
Port 2222
User Jack
IdentityFile ~/.ssh/personal
```
can now use ssh as: ```ssh Personal```

## Add SWAP
* Note SWAPs can be harmful to older SSDs...

1. ```sudo fallocate -l 4G /swapfile```
2. ```ls -lh /swapfile```
3. ```sudo chmod 600 /swapfile```
4. ```ls -lh /swapfile```
5. ```sudo mkswap /swapfile```
6. ```sudo swapon /swapfile```
7. ```sudo nano /etc/fstab```
8. Add to the bottom of the file:
   ``` - /swapfile   none    swap    sw    0   0```
9. ```sudo sysctl vm.swappiness=10```
10. ```sudo sysctl vm.vfs_cache_pressure=50```
11. ```sudo nano /etc/sysctl.conf```
12. Add to the bottom:
    - ```vm.swappiness=10```
    - ```vm.vfs_cache_pressure = 50```

## Increase PHP and Nginx memory sizes:

php5 location = /etc/php5/fpm/php.ini

1. ```sudo (nano | vim | vi ) /etc/php7.0/fpm/php.ini```
2. Update the following lines to resemble below:
```
    upload_max_filesize = 50M
    post_max_size = 50M
    max_execution_time = 120
    max_input_time = 120
    memory_limit = 64M
```

3. ```sudo (nano | vim | vi) /etc/nginx/nginx.conf```
4. Update ```client_max_body_size``` to: ```client_max_body_size 100M;```

## Remove Nginx headers
1. add `server_tokens off` to ```/etc/nginx/sites-available/defult/```
    - this will only remove the version

***I haven't tested this, but if you want to change the server in the response

2. `sudo apt-get install nginx-extras`
3. `more_set_headers 'Server: some server name';`

Test it: `curl -I ${website}`

***

## Remove PHP headers
1. Open php.ini  

    - ```sudo (nano | vim | vi) /etc/php/7.0/fpm/php.ini``` 

2. uncomment ```cgi.fix_pathinfo=1``` - remove the semi-colon in front of it, and change the value to 0

    - ```cgi.fix_pathinfo=0```

3. Remove X-Powered-by
    
    - ```expose_php = 0```

4. Restart php: ```sudo systemctl restart php7.0-fpm```

## Remove Express headers

    app.disable('x-powered-by');

## Serving and Securing assets with S3 over CloudFront
link: https://jackrothrock.com/a-to-z-with-amazon-s3/


## Hosting multiple domains on a single droplet
link: https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-14-04-lts

1. create an file for each domains to house the server blocks in /etc/nginx/sites-enabled/
    1. name of file should be like, `thesite.com`
    2. you can put the file `/etc/nginx/sites-available/` but you will have to create a symbolic link `ln` to `/etc/nginx/sites-enabled/`
    3. it should follow the same format as the default file in `/etc/nginx/sites-enabled/`
2. `sudo vi /etc/nginx/nginx.conf`
    1. Uncomment: server_names_hash_bucket_size 64;

## Mitigate DoS on Nginx
1. add the limits outside the server block
2. add the client timeouts to close long connections
3. add the limits to the location
4. deny areas where people may try to access during a DDoS
```
limit_req_zone $binary_remote_addr zone=one:10m rate=3r/m;
limit_conn_zone $binary_remote_addr zone=addr:10m;
server {
    # other stuff

    client_body_timeout 5s;
    client_header_timeout 5s;

    location / {
        limit_req zone=one burst=5;
        limit_conn addr 10;
        # other stuff
    }

    ## can change this to /login or where ever. You may just want to set it to your ip
    location /wp-login.php {
        # allow 111.333.444.555
        deny all;
    }

    #other stuff
}
```

## Gzip
Later I may try and update this to use [brotli](https://afasterweb.com/2016/03/15/serving-up-brotli-with-nginx-and-jekyll/)

1. ```sudo vi /etc/nginx/nginx.conf```
2. Make yours look similar to:
```
        # Gzip Settings
        ##

        gzip on;
        gzip_disable "msie6";

        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 4 42k;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/x-font-ttf application/x-font-opentype font/eot font/opentype image/svg+xml font/otf application/xml+rss text/javascript;
``` 

Double check this with [GTmetrix](https://gtmetrix.com) and [Web Page Test](https://webpagetest.org)

## Enable caching
1. ``` sudo vi /etc/nginx/sites-available/default ```
2. Add this somewhere in your server block:
    - ```location ~* \.(ico|svg|css|js|gif|jpe?g|png|woff|woff2)$ { expires 30d; add_header Pragma public; add_header Cache-Control "public"; }```

Double check this with [GTmetrix](https://gtmetrix.com) and [Web Page Test](https://webpagetest.org)

## Enable http2
If you are using ssl, which you should be, only add this to your ssl server block. 

1. ```sudo vi /etc/nginx/sites-available/default```
2. update ssl server block:
    ``` 
        server {

            listen 443 http2 default_server;

            listen [::]:443 ssl http2 default_server;

            #other stuff 
    ```

Double check this with [Lighthouse](https://developers.google.com/web/tools/lighthouse/)

## Closing notes:
There are other practices that need to be followed to ensure security - always using sftp, using different keys for different servers, always running `sudo apt-get update; sudo apt-get upgrade -y` when logging into the server, not reusing the same passwords - but in the end, if someone finds a zero day in the hypervisor - none of this really matters.

Also, in the above, having the email sent showing the super user's account name isn't ideal. I currently haven't figured out a way to 'spoof' the name to just show "mail" or something. However, relaying can be done to use something like gmail - you could then use an email alias through gmail.

[https://www.linode.com/docs/email/postfix/postfix-smtp-debian7](https://www.linode.com/docs/email/postfix/postfix-smtp-debian7)

## Extras
Here's a great video that goes into some of the stuff listed above while setting up a DO server: 

[https://www.youtube.com/watch?v=YZzhIIJmlE0](https://www.youtube.com/watch?v=YZzhIIJmlE0)

Reddit discussion - [https://www.reddit.com/r/webdev/comments/5x59z2/step_by_step_guide_on_how_to_secureconfigure_an/](https://www.reddit.com/r/webdev/comments/5x59z2/step_by_step_guide_on_how_to_secureconfigure_an/)

## License 
    These notes are released under MIT.