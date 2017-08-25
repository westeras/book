# Generating a Custom CA Certificate

It can be beneficial for testing purposes to have your own custom CA for signing certificates.  This comes in handy when securing web based services running on your local machine or on virtual machines.  You can import the CA into your browser and avoid the security warnings you'd get from using self-signed certificates.

SSL is built on the concept of public/private keys.  At a high level, each server in a network has a private key and a certificate \(which is essentially a wrapper object containing the public key as well as some metadata\).  The private key stays private on the server it belongs to.  The certificate is freely distributed to any other server that wants to communicate securely with the respective server.  When server A requests a connection to server B, it presents a handshake that contains some data encrypted by its private key.  If server B is configured to "trust" server A's certificate, it will allow the connection.  Not only that, but the data it transfers is guaranteed to be secure since it was encrypted by server A's private key, which only server A has access to \(and only server A's certificate can successfully decrypt it\).  Finally, a given certificate can "sign" another certificate, introducing the concept of a "chain".  This is where the idea of a CA certificate comes in.  Assuming all the servers in a network "trust" the root CA certificate, then any new server introduced to the network will automatically be trusted by all others in the network, provided the new server's certificate has been signed by the CA.

I will be generated a Root CA and an Intermediate CA, which is how professional CA authorities work.  The Root CA \(which is self-signed\) is only used to sign the Intermediate CA, which is then used to sign certificates, rather than signing them with the Root CA directly.  This adds an extra layer of protection to prevent a compromise of the Root CA.

Note: This guide assumes the default openssl configuration on a vanilla CentOS machine, so we won't be adding our own.  The only configuration required on our end is to create a couple files that act as a database that keeps track of certificate signatures by the CA.  To create the files, do the following _as root_:

```
touch /etc/pki/CA/index.txt
echo 1000 > /etc/pki/CA/serial
```

Start in a newly created directory, which we'll use as a workspace for generating our keys and certificates.  We'll also create some subdirectories:

```
mkdir pki && cd pki
mkdir private certs
```

First, we'll generate the root private key:

`openssl genrsa -aes256 -out private/ca.key.pem 4096`

Adding `-aes256` indicates you'd like to encrypt the key with a password. The `4096` argument is the size of the generated private key in bits.

Next, we'll generate the root certificate using the private key:

    openssl rq \                    # the `req` command is a PKCS#10 certificate request and certificate generating utility
    -key private/ca.key.pem \        # specifies the private key we just created
    -new \                           # signals to create a new CSR
    -x509 \                          # indicates we'll be creating a self-signed certificate rather an a CSR (more below)
    -days 10000 \                    # length of validity for certificate
    -sha256 \                        # specifies the message digest to sign the request with
    -extensions v3_ca \              # indicates our certificate will be a certificate authority (CA)
    -out certs/ca.cert.pem           # file to save generated certificate

You'll be prompted for some basic information about the certificate, which looks something like this:

```
$ openssl req -key private/ca.key.pem -new -x509 -days 10000 -sha256 -extensions v3_ca -out certs/ca.cert.pem
Enter pass phrase for private/ca.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:Texas
Locality Name (eg, city) [Default City]:Austin
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:WestermanRootCA
Email Address []:
```

The responses you provide will make up the Subject of the certificate which is used to identify it.  You can output the subject of your new certificate with the following:

```
$ openssl x509 -noout -subject -in certs/ca.cert.pem
subject= /C=US/ST=Texas/L=Austin/O=Default Company Ltd/CN=WestermanRootCA
```

Next we'll do the same thing for the Intermediate CA.  The Intermediate CA certificate/key pair will be generated in a similar manner but signed by the Root CA.

First, generate the private key:

`openssl genrsa -aes256 -out private/intermediate.key.pem 4096`

Now we'll generate a certificate signing request \(CSR\).  A CSR encapsulates your private key, and is typically sent to a Certificate Authority to be signed and returned to you as a CA-signed certificate.  We will be generating the CSR and using our Root CA certificate to sign the certificate.

To create the CSR:

`openssl req -new -sha256 -key private/intermediate.key.pem -out csr/intermediate.csr.pem`

Which leads you to another series of prompts similar to when we generated the Root CA certificate \(the password can be left empty\):

```
$ openssl req -new -sha256 -key private/intermediate.key.pem -out csr/intermediate.csr.pem
Enter pass phrase for private/intermediate.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:Texas
Locality Name (eg, city) [Default City]:Austin
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:WestermanIntermediateCA
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Now we can sign the CSR with our Root CA and generate our Intermediate CA certificate:

```
sudo openssl ca \                        # ca command is used to sign CSRs
-extensions v3_ca \                      # indicates this will be a CA certificate
-days 10000 \                            # lifetime of the certificate
-notext \                                # specifies to not output the text form of the certificate
-md sha256 \                             # message digest to use
-in csr/intermediate.csr.pem  \          # CSR from previous step
-out certs/intermediate.cert.pem \       # certificate output file
-keyfile private/ca.key.pem \            # CA private key file
-cert certs/ca.cert.pem                  # CA certificate file
```

After following the prompts you should see your new Intermediate CA certificate in the certs directory.  Notice that it is owned by root \(the previous command needs to run as root in order to access the CA database files created at the beginning\).  You'll need to log in as root and chown the file to your user:

```
sudo -i
chown <user>:<group> certs/intermediate.cert.pem
exit
```



