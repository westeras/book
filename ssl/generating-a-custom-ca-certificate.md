# Generating a Custom CA Certificate

It can be beneficial for testing purposes to have your own custom CA for signing certificates.  This comes in handy when securing web based services running on your local machine or on virtual machines.  You can import the CA into your browser and avoid the security warnings you'd get from using self-signed certificates.

SSL is built on the concept of public/private keys.  At a high level, each server in a network has a private key and a certificate \(which is essentially a wrapper object containing the public key as well as some metadata\).  The private key stays private on the server it belongs to.  The certificate is freely distributed to any other server that wants to communicate securely with the respective server.  When server A requests a connection to server B, it presents a handshake that contains some data encrypted by its private key.  If server B is configured to "trust" server A's certificate, it will allow the connection.  Not only that, but the data it transfers is guaranteed to be secure since it was encrypted by server A's private key, which only server A has access to \(and only server A's certificate can successfully decrypt it\).  Finally, a given certificate can "sign" another certificate, introducing the concept of a "chain".  This is where the idea of a CA certificate comes in.  Assuming all the servers in a network "trust" the root CA certificate, then any new server introduced to the network will automatically be trusted by all others in the network, provided the new server's certificate has been signed by the CA.

I will be generated a Root CA and an Intermediate CA, which is how professional CA authorities work.  The Root CA \(which is self-signed\) is only used to sign the Intermediate CA, which is then used to sign certificates, rather than signing them with the Root CA directly.  This adds an extra layer of protection to prevent a compromise of the Root CA.

Start in a newly created directory, which we'll use as a workspace for generating our keys and certificates.  We'll also create some subdirectories:

```
mkdir pki && cd pki
mkdir private certs 
```

First, we'll generate the root private key:

`openssl genrsa -aes256 -out private/ca.key.pem 4096`

Adding `-aes256` indicates you'd like to encrypt the key with a password. The `4096` argument is the size of the generated private key in bits.

Next, we'll generate the root certificate using the private key:

    openssl req \                    # the `req` command is a PKCS#10 certificate request and certificate generating utility
    -key private/ca.key.pem \        # specifies the private key we just created
    -new \                           # indicates to create a new certificate
    -x509 \                          # indicates we'll be creating a self-signed certificate
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



