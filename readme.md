# Creating a Self-Signed Certificate With OpenSSL and nginx

### Basic configuration

- Create a directory structure to store the certificates

    | Directory | Purpose |
    |--|--|
    | ca/private | private keys |
    | ca/certs | store certificates |
    | ca/newcerts | backup certificates |
    | ca/csr | store signing requests |
    | | |

- Change permission to the private keys folder [ security relevant but optional ]

    ```bash
    ubuntu@k8sworker3:~$ chmod -v 700 ca/private
    mode of 'ca/private' changed from 0775 (rwxrwxr-x) to 0700 (rwx------)
    ```

- Configure openssl to use the file named `serial` to generate serial numbers for the certificates.</br>

    ```bash
    # Generate 16 random hexadecimal numbers
    ubuntu@k8sworker3:~$ openssl rand -hex 16 > ca/serial
    # Verify
    ubuntu@k8sworker3:~$ cat ca/serial
    ae5989492bcc15a071ff1ab2c14f8ce2
    ```

    Generated certificates will be stored in the file `index`:

    ```bash
    ubuntu@k8sworker3:~$ touch ca/index
    ```

### Create the Root CA private key

- Generate a RSA key, encrypted with password (optional) - here aes256 algorithm with 4096 bits long

    ```bash
    ubuntu@k8sworker3:~/ca$ openssl genrsa -aes256 -out private/root-ca.key 4096
    ```

- Generate configuration file to use openssl as a CA authority</br> 
    See `root-ca.conf` file

### Create certificate signing request and sign the root certificate

- Create both the private key and CSR with a single command
    ```bash
    ubuntu@k8sworker3:~/ca$ openssl req -config root-ca.conf -extensions v3_ca -key private/root-ca.key -new -x509 -days 3650 -out certs/root-ca.crt

    ubuntu@k8sworker3:~/ca$ ls certs
    root-ca.crt
    ```

- Display information about de created certificate

    ```bash
    ubuntu@k8sworker3:~/ca$ openssl x509 -noout -text -in certs/root-ca.crt
    ```

### Creating a server certificate 

Creating a CA-Signed Certificate With Our Own CA. This will be used by nginx.

##### Step 1. Generate a signing request

-   Create a private key used by the server to create a signing request

    ```bash
    openssl genrsa -out private/server.key 2048
    ```

-   Create configuration file, here named `server-csr.conf`.</br>
    Note: DNS.1 in subjectAltName must be the domain name. Ex: `mypage.com`

-   Create a certificate signing request
    ```bash
    ubuntu@k8sworker3:~/ca$ openssl req -new -key private/server.key -sha256 -out csr/server.csr -config csr/server-csr.conf
    # Verify request
    ubuntu@k8sworker3:~/ca$ openssl req -noout -text -in csr/server.csr
    ```

-   Generate certificate 

    ```bash
    ubuntu@k8sworker3:~/ca$ openssl ca -config root-ca.conf -in csr/server.csr -out certs/server.crt -extensions req_ext -extfile csr/server-csr.conf
    ```

-   Display server certificate info
    ```bash
    ubuntu@k8sworker3:~/ca$ openssl x509 -noout -text -in certs/server.crt
    ```

### Verify private keys and certificates [ optional ]

-   Check if a private key that does meet integrity

    ```bash
    root@k8sworker3:/home/ubuntu/ca# openssl rsa -in private/root-ca.key -check -noout
    Enter pass phrase for private/root-ca.key:
    RSA key ok
    ```

-   Confirm the Modulus Value Matching with Private Key and SSL/TLS certificate Key Pair.</br> The modulus of the private key and certificate must match exactly.

    ```bash
    root@k8sworker3:/home/ubuntu/ca# openssl x509 -noout -modulus -in certs/root-ca.crt
    Modulus=CEE5DED0........4DC25765

    root@k8sworker3:/home/ubuntu/ca# openssl rsa -noout -modulus -in private/root-ca.key
    Modulus=CEE5DED0........4DC25765
    ```

### Configure Nginx to Use SSL

In this step we can check the certificate to be used with an nginx server.

#### Copy certificates and create DH group

-   Copy the certificate and the private key as follows ( will also be renamed )

    ```bash
    ubuntu@k8sworker3:/etc/ssl/certs$ sudo cp ~/ca/certs/server.crt ./nginx-selfsigned.crt
    ubuntu@k8sworker3:/etc/ssl$ sudo cp ~/ca/private/server.key /etc/ssl/private/nginx-selfsigned.key
    ```

-   Create a strong DH (Diffie-Hellman) group 

    > This determine the strength of the key used in the key exchange process with clients. It will be stored in `/etc/nginx/dhparam.pem` that can be used in Nginx configuration. This is used for perfect forward secrecy, which generates ephemeral session keys to ensure that past communications cannot be decrypted if the session key is compromised.

    ```bash
    ubuntu@k8sworker3:/etc/ssl$ sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
    Generating DH parameters, 4096 bit long safe prime
    ```

#### Configuring Nginx HTTP Web Server to use SSL

-   Creating a Configuration Snippet for SSL Key and Certificate

    ```bash
    ubuntu@k8sworker3:/etc/ssl$ sudo nano /etc/nginx/snippets/self-signed.conf
    ```
    Content of `self-signed.conf`:
    ```conf
    ssl_certificate /et##c/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ```

-   Create additional snippet to set Nginx up with a strong SSL cipher suite to keep the server safe:

    ```bash
    ubuntu@k8sworker3:/etc/ssl$ sudo nano /etc/nginx/snippets/ssl-params.conf
    ```

    Content:
    ```conf
    # from https://cipherli.st/
    # and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    # Disable preloading HSTS for now.  You can use the commented out header line that includes
    # the "preload" directive if you understand the implications.
    #add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    ssl_dhparam /etc/ssl/certs/dhparam.pem; # as generated above 
    ```

    Additional info:
        - Because we are using a self-signed certificate, the SSL stapling will not be used. Nginx will simply output a warning, disable stapling for our self-signed cert, and continue to operate correctly
        - you can adjust the preferred DNS resolver for upstream requests
        - You will get a warning:
            nginx: [warn] "ssl_stapling" ignored, issuer certificate not found for certificate "/etc/ssl/certs/nginx-selfsigned.crt"

-   Adjust the Nginx Configuration to Use SSL

    Edit your server block. Usually named `default` i`n `/etc/nginx/sites-available`

    ```conf
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name server_domain_or_IP;
        return 302 https://$server_name$request_uri;
    }

    server {

        # SSL configuration

        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;
        include snippets/self-signed.conf;
        include snippets/ssl-params.conf;

        . . .
    }
    ```

-   Adjust firewall to enable http and https traffic

-   Verify nginx configuration and restart nginx

    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

#### Add a Root Certificate in Google Chrome

        Procedure
            1. Open the browser.
            2. Click Customize and control Google Chrome button in the upper right corner.
            3. Choose Settings. ...
            4. Under Privacy and security section, click More. ...
            5. Click Manage certificates, The new window will appear. ...
            6. Choose Trusted Root Certification Authorities tab.
            7. Click Import. ...
            8. In the opened window, click Next.

Useful links:

[How to create and use self signed ssl on nginx](https://www.howtogeek.com/devops/how-to-create-and-use-self-signed-ssl-on-nginx/)</br>
[OpenSSL self signet cert](https://www.baeldung.com/openssl-self-signed-cert)</br>
[How to create a self signed ssl certificate for nginx](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-16-04)</br>
[How to configure nginx to use self signed ssl tls certificate on ubuntu](https://hostadvice.com/how-to/web-hosting/vps/how-to-configure-nginx-to-use-self-signed-ssl-tls-certificate-on-ubuntu-18-04-vps-or-dedicated-server/)</br>
[CA Server - OpenSSL, Tech Tutorials - David McKone](https://youtu.be/nOSl4dmywe8?si=XyFr-zyyKCNcvjth&t=5195)
