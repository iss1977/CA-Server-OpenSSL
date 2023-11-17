


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

Generate configuration file to use openssl as a CA authority


