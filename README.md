# Guide step by step

- Step 1: Allow port inbound 80 & 443

```terminal
sudo ufw allow 80
sudo ufw allow 443
```

```text
Note: if config on EC2 (AWS) -> edit inbound rules in security groups -> allow all trafic from anywhere (0.0.0.0/0)
```

- Step 2: Setup SSL

    - **2.0 Install nginx**

        ```terminal
        sudo apt install nginx
        sudo service nginx start
        ```

    - On VPS other EC2 (Ex: Vultr,...)
        - **2.1 Create the SSL Certificate**

            ```terminal
            sudo mkdir /etc/ssl/private
            sudo chmod 700 /etc/ssl/private
            ```

            ```terminal
            sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
            ```

            **=> Output**
            ```terminal
            Country Name (2 letter code) [XX]:US
            State or Province Name (full name) []:Example
            Locality Name (eg, city) [Default City]:Example 
            Organization Name (eg, company) [Default Company Ltd]:Example Inc
            Organizational Unit Name (eg, section) []:Example Dept
            Common Name (eg, your name or your server's hostname) []:your_domain_or_ip
            Email Address []:webmaster@example.com
            ```

            ```terminal
            sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
            ```
    
        - **2.2 Configure Nginx to Use SSL**

            ***Create the TLS/SSL Server Block***
            ```terminal
            sudo vi /etc/nginx/conf.d/ssl.conf
            ```
            
            edit file ***/etc/nginx/conf.d/ssl.conf***
            ```shell
            server {
                listen 443 http2 ssl;
                listen [::]:443 http2 ssl;

                server_name your_server_ip;

                ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
                ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
                ssl_dhparam /etc/ssl/certs/dhparam.pem;

                ########################################################################
                # from https://cipherlist.eu/                                            #
                ########################################################################
                
                ssl_protocols TLSv1.3;# Requires nginx >= 1.13.0 else use TLSv1.2
                ssl_prefer_server_ciphers on;
                ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
                ssl_ecdh_curve secp384r1; # Requires nginx >= 1.1.0
                ssl_session_timeout  10m;
                ssl_session_cache shared:SSL:10m;
                ssl_session_tickets off; # Requires nginx >= 1.5.9
                ssl_stapling on; # Requires nginx >= 1.3.7
                ssl_stapling_verify on; # Requires nginx => 1.3.7
                resolver 8.8.8.8 8.8.4.4 valid=300s;
                resolver_timeout 5s;
                # Disable preloading HSTS for now.  You can use the commented out header line that includes
                # the "preload" directive if you understand the implications.
                #add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
                add_header X-Frame-Options DENY;
                add_header X-Content-Type-Options nosniff;
                add_header X-XSS-Protection "1; mode=block";
                ##################################
                # END https://cipherlist.eu/ BLOCK #
                ##################################
            }
            ```

    - On EC2 (AWS Cloud Compute)
        - **Setting up SSL with Letsencrypt**

            ```terminal
            sudo wget http://nginx.org/keys/nginx_signing.key
            sudo apt-key add nginx_signing.key
            cd /etc/apt
            echo -e "deb http://nginx.org/packages/ubuntu xenial nginx \ndeb-src http://nginx.org/packages/ubuntu xenial nginx" | sudo tee -a sources.list
            sudo apt-get update
            ```

- Step 3: Certbot domain

    - **Install certbot**

        ```terminal
        sudo apt install python3-certbot-nginx
        ```

    - **Certbot domain**

        ```terminal
        sudo certbot --nginx -d domain
        ```

- Step 4: Setup location nginx for domain

    - **Edit location in etc/nginx/sites-available/default**

        ```shell
        location / {
            proxy_pass http://localhost:5000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
        ```

    