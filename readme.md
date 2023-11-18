


## Create self signed certificate

### Basic configuration
Create a directory structure to store the certificates

| Directory | Purpose |
|--|--|
| ca/private | private keys |
| ca/certs | store certificates |
| ca/newcerts | backup certificates |
| ca/csr | store signing requests |
| | |

Change permission to the private keys folder

```bash
ubuntu@k8sworker3:~$ chmod -v 700 ca/private
mode of 'ca/private' changed from 0775 (rwxrwxr-x) to 0700 (rwx------)
```

Will configure openssl to use the file named `serial` to generate serial numbers for the certificates.</br>
Will generate this file using the commands:


```bash
# Generate 16 random hexadecimal numbers
ubuntu@k8sworker3:~$ openssl rand -hex 16 > ca/serial
# Verify
ubuntu@k8sworker3:~$ cat ca/serial
ae5989492bcc15a071ff1ab2c14f8ce2
```

Generated certificates will be stored in the file `index`

```bash
ubuntu@k8sworker3:~$ touch ca/index
```

### Create the Root CA private key

Generate a RSA key encrypted with the aes256 algoritm (password) with 4096 bits long

```bash
openssl genrsa -aes256 -out private/root-ca.key 4096
```

Generate configuration file to use openssl as a CA authority</br>
See `root-ca.conf` file

### Create certificate signing request and sign the root certificate

```bash
ubuntu@k8sworker3:~/ca$ openssl req -config root-ca.conf -extensions v3_ca -key private/root-ca.key -new -x509 -days 3650 -out certs/root-ca.crt

Enter pass phrase for private/root-ca.key: my-password
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name ( 2 letter code ) [AT]:
State or Province Name [Linz]:
Locality Name []:Linz
Organization Name [iss-dev]:
Organization Unit Name []:
Common Name []:RootCA
Email Address []:

ubuntu@k8sworker3:~/ca$ ls certs
root-ca.crt

```

Display information about de created certificate

```bash
ubuntu@k8sworker3:~/ca$ openssl x509 -noout -text -in certs/root-ca.crt

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            27:1e:17:55:e4:22:ea:f5:0d:b1:8c:f5:bb:7e:3e:1b:f4:84:69:04
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = AT, ST = Linz, L = Linz, O = iss-dev, CN = RootCA
        Validity
            Not Before: Nov 18 00:13:44 2023 GMT
            Not After : Nov 15 00:13:44 2033 GMT
        Subject: C = AT, ST = Linz, L = Linz, O = iss-dev, CN = RootCA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:ce:e5:de:d0:b9:24:e0:df:23:b7:6e:e5:a7:f1:
                    .............................................
                    ea:1e:17:43:05:c5:03:a3:07:e2:f8:c5:69:33:4d:
                    c2:57:65
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                3E:2B:34:7C:63:17:79:8D:AA:76:D8:D8:7B:02:21:34:1E:43:A6:A4
            X509v3 Authority Key Identifier:
                3E:2B:34:7C:63:17:79:8D:AA:76:D8:D8:7B:02:21:34:1E:43:A6:A4
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        6b:89:18:6a:4b:a2:62:91:42:49:de:9c:ad:09:da:ae:7d:3f:
        ......................................................
        c1:1c:12:4e:45:d8:22:b5


```

### Creating a server certificate

##### Step 1. Generate a signing request

Create a private key used by the server to create a signing request

```bash
openssl genrsa -out private/server.key 2048
```

Create configuration file, here named `server-csr.conf`

Create a certificate signing request
```bash
ubuntu@k8sworker3:~/ca$ openssl req -new -key private/server.key -sha256 -out csr/server.csr -config csr/server-csr.conf

ubuntu@k8sworker3:~/ca$ openssl req -noout -text -in csr/server.csr
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = AT, ST = Linz, L = Linz, O = iss-dev, emailAddress = yourmail@mail.com, CN = iss-dev.xyz
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c6:32:56:cf:e5:d4:b6:4e:38:8d:c3:09:5d:85:
                    .............................................
                    f2:65:17:43:b4:47:82:87:9a:50:de:b5:bf:cb:89:
                    0a:01
                Exponent: 65537 (0x10001)
        Attributes:
            Requested Extensions:
                X509v3 Subject Alternative Name:
                    DNS:*.iss-dev.xyz, DNS:*.dev.iss-dev.xyz, IP Address:138.3.255.57
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        42:f5:e4:18:e0:2a:aa:8c:26:4e:6e:ef:17:05:80:f3:9a:91:
        ......................................................
        18:3a:03:0b:a6:01:b3:c2:d1:21:d6:3c:af:55:53:f7:88:b8:
        07:52:40:4c


```

Generate certificate 

```bash
ubuntu@k8sworker3:~/ca$ openssl ca -config root-ca.conf -in csr/server.csr -out certs/server.crt -extensions req_ext -extfile csr/server-csr.conf

Using configuration from root-ca.conf
Enter pass phrase for ./private/root-ca.key: my-password
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            ae:59:89:49:2b:cc:15:a0:71:ff:1a:b2:c1:4f:8c:e2
        Validity
            Not Before: Nov 18 01:53:27 2023 GMT
            Not After : Nov 17 01:53:27 2024 GMT
        Subject:
            countryName               = AT
            stateOrProvinceName       = Linz
            organizationName          = iss-dev
            commonName                = iss-dev.xyz
            emailAddress              = yourmail@mail.com
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:*.iss-dev.xyz, DNS:*.dev.iss-dev.xyz, IP Address:138.3.255.57
Certificate is to be certified until Nov 17 01:53:27 2024 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Display server certificate info
```bash
ubuntu@k8sworker3:~/ca$ openssl x509 -noout -text -in certs/server.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            ae:59:89:49:2b:cc:15:a0:71:ff:1a:b2:c1:4f:8c:e2
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = AT, ST = Linz, L = Linz, O = iss-dev, CN = RootCA
        Validity
            Not Before: Nov 18 01:53:27 2023 GMT
            Not After : Nov 17 01:53:27 2024 GMT
        Subject: C = AT, ST = Linz, O = iss-dev, CN = iss-dev.xyz, emailAddress = yourmail@mail.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c6:32:56:cf:e5:d4:b6:4e:38:8d:c3:09:5d:85:
                    .............................................
                    f2:65:17:43:b4:47:82:87:9a:50:de:b5:bf:cb:89:
                    0a:01
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:*.iss-dev.xyz, DNS:*.dev.iss-dev.xyz, IP Address:138.3.255.57
            X509v3 Subject Key Identifier:
                86:CC:51:7B:18:B9:B8:CE:7B:EC:5A:93:D7:DD:53:44:C4:9B:B8:5F
            X509v3 Authority Key Identifier:
                3E:2B:34:7C:63:17:79:8D:AA:76:D8:D8:7B:02:21:34:1E:43:A6:A4
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        9d:3c:85:53:d9:72:c7:5b:cf:ac:c6:b8:f3:cc:59:08:e2:34:
        .....................................................
        2d:4c:ab:1d:da:29:de:ff
```

### Configure Nginx to Use SSL

In this step we can check the certificate to be used with an nginx server.

#### Copy certificates and create DH group

- Copy the certificate and the private key as follows ( will also be renamed )

```bash
ubuntu@k8sworker3:/etc/ssl/certs$ sudo cp ~/ca/certs/server.crt ./nginx-selfsigned.crt
ubuntu@k8sworker3:/etc/ssl$ sudo cp ~/ca/private/server.key /etc/ssl/private/nginx-selfsigned.key
```

- Create a strong DH (Diffie-Hellman) group 

This determine the strength of the key used in the key exchange process with clients. It will be stored in `/etc/nginx/dhparam.pem` that can be used in Nginx configuration.

```bash
ubuntu@k8sworker3:/etc/ssl$ sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
Generating DH parameters, 4096 bit long safe prime
```

#### Configuring Nginx HTTP Web Server to use SSL

- Creating a Configuration Snippet for SSL Key and Certificate

```bash
ubuntu@k8sworker3:/etc/ssl$ sudo nano /etc/nginx/snippets/self-signed.conf
```
Content of `self-signed.conf`:
```conf
ssl_certificate /et##c/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

- Create additional snippet to set Nginx up with a strong SSL cipher suite to keep the server safe:

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

- Adjust the Nginx Configuration to Use SSL

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

- Adjust firewall to enable http and https traffic

- Verify nginx configuration and restart nginx

```bash
sudo nginx -t
sudo systemctl restart nginx
```


https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-16-04
https://hostadvice.com/how-to/web-hosting/vps/how-to-configure-nginx-to-use-self-signed-ssl-tls-certificate-on-ubuntu-18-04-vps-or-dedicated-server/
https://youtu.be/nOSl4dmywe8?si=XyFr-zyyKCNcvjth&t=5195
