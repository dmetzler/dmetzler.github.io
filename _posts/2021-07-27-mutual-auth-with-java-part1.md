---
tags: 
 - blog
layout: post
title: Mutual Authentication with NGinx and Java (Part 1)
excerpt: |
 User and password is a way to authenticate to an application. Let's discover another way with mutual authentication.

img_url: /images/2021-07-27-mutual-auth-with-java-part1.png

---
## Context
When you need to securely access a resource on a server, whereas it is a document or the result of a web service call, the transaction needs to verify different things:
- The client can trust the server
- The server must verify that the client can access the resource

The first part is verified by the HTTP**S** protocol and is now well known. For the second part, one can use a login and a password to identify a user. Another option is to present a client TLS certificate that the server can trust, this is what is called [Mutual authentication ](https://en.wikipedia.org/wiki/Mutual_authentication) or *two-way authentication*.  This series of posts will detail how it works and how to configure a web server and a java client to establish that trust relationship.

## How does this work
Let's first introduce the notion of the key. A cryptographic key is composed of a pair of two keys:
- the private key, that must remain secret and never shared with anyone. This key can sign messages.
- the public key allows us to verify that a message has been correctly signed by the private key. The public key can be shared with anyone, it doesn't allow to sign a message.

When you want to verify a message, you look at the signature of the message, check it with the public key, and voila... The only problem here is that you need to trust the public key and ensure that it relies upon the supposed sender. You then need a *certificate* of authenticity for that key.

So either you know the sender in person, and he gave you in person his public key, or you need someone who you trust, that can certify that the public key is authentic. That someone is known as the *certificate authority* (or CA) and can present his certificate. That certificate can be signed by another CA that can, in turn,  be signed by another CA, etc... until we find a common CA that both parties can trust.

The certificate is composed of the public key and its signature by a trusted authority. A certificate has an expiration date so that after a certain period, it is not valid anymore. That's why certificates need to be renewed regularly.

In browsers, there is a list of pre-installed *Trusted root certification authorities* which allow trusting the vast majority of SSL sites on the internet.

In mutual TLS, both the server and the client have to show their signed certificate. Here is the flow:


![Mutual TLS flow](/images/2021-07-27-Mutual-TLS.svg)

1. The user (client) requests a resource on an endpoint using HTTPs (`GET /index.html`)
2. The server initiates the *SSL handshake* by presenting its certificate
3. The user verifies that the certificate has been signed by a trusted authority
4. The user, send its certificate to the server
5. The server verifies that the certificate has been signed by a trusted authority
6. The SSL handshake is over, both parties have verified their certificate, the user can now access the resource

## Generating keys and certificates
### Using OpenSSL
Now, let's get our hands dirty and generate some certificates. The first key/certificate pair that we will generate is the one for our CA. We can do that in one command:
  

```shell
$ openssl req -nodes -x509 -newkey rsa:4096 \
 -keyout ca.key \
 -out ca.crt \
 -days 365 \
 -subj "/CN=my-own-ca"
Generating a 4096 bit RSA private key
.......................................................++
writing new private key to 'ca.key'
-----
```

This will generate two files:
- `ca.key`: the private key that MUST no be shared
- `ca.crt`: a self-signed certificate that we can decide to trust (you should always trust yourself!). 
  
That certificate is valid for 365 days (one year), and is identified by the *common name* (CN) `my-own-ca`.
  

The second key that we will generate is our client key. We will then sign this key by the CA to generate its certificate:

```shell
$ openssl genrsa -out client.key 2048
Generating RSA private key, 2048 bit long modulus
..+++
.........................................+++
e is 65537 (0x10001)
```

This generates a private key in the file `client.key`. To sign that key we need to generate at *Certificate Signing Request* (CSR) that our CA will sign. 
  

```shell
$ openssl req -new \
 -key client.key \
 -out client.csr \
 -subj "/CN=my-identity" \
 -sha256
```
This generates a `client.csr` file that requests to sign my key with the *common name* `my-identity`. That common name identifies who you are. In the case of the server certificate, you would use the DNS value for that server. 

Once you have that CSR, you would usually send it to your CA (for instance Gandi) to sign it. The CA will do its verification and send you back the certificate. As we are our own CA here, we can sign our key ourselves:
 
```shell
$ openssl x509 -req \
 -in client.csr \
 -CA ca.crt \
 -CAkey ca.key \
 -CAcreateserial \
 -out client.crt \
 -days 1024
Signature ok
subject=/CN=my-identity
Getting CA Private Key
```

This generates a `client.crt` file that contains our certificate and is valid for 1024 days. If your system trusts the CA, then it will trust `client.crt`.

### Using mkcert

We still need to have a third certificate for the server. We can generate it with `openssl` like all the others, but we will here use another tool that simplifies all the configurations. That tool is called `mkcert`, it runs all complex `openssl` command for you and installs a common CA in all the needed place of the operating system so that your browsers will trust that certificate. In essence, it is super helpful to generate TLS certificates for your local development.

The first thing is to [install mkcert following their instructions](https://github.com/FiloSottile/mkcert). Once you've been able to run `mkcert -install`, generating a certificate for your system is as simple as running: 
  

```shell
$ mkcert nginx.local
  
Created a new certificate valid for the following names 📜
 - "nginx.local"
  
The certificate is at "./nginx.local.pem" and the key at "./nginx.local-key.pem" ✅
  
It will expire on 27 October 2023 🗓
```

It will generate two files:
- `nginx.local-key.pem`: the private key
- `nginx.local.pem`: a local trusted certificate

The CA used to sign this certificate is not the same as the one for our client key, but that's not a problem as it has been installed by `mkcert` in our OS trusted CA.

  

## Configuring NGinx

We now have 3 pairs of keys and certificates. Let's configure NGinx so that:
- It uses our `nginx.local.pem` certificate to serve requests
- It trusts only request in which it can find a trusted certificated 

The complete `nginx.conf` file can be found in the [project's repository](https://github.com/dmetzler/mutual-auth-with-java/blob/main/nginx.conf), and we will detail here only the interesting part:

```conf
server {
 listen 443 ssl;
 server_name nginx.local;
  
 # Server certificate
 ssl_protocols TLSv1.1 TLSv1.2;
 ssl_certificate /etc/nginx/certs/nginx.local.pem;
 ssl_certificate_key /etc/nginx/certs/nginx.local-key.pem;
  
 
 # client certificate
 ssl_verify_depth 2;
 ssl_client_certificate /etc/nginx/certs/ca.crt;
 ssl_ciphers 'HIGH:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:@SECLEVEL=0';
  
 # make verification optional, so we can display a 403 message to those
 # who fail authentication
 ssl_verify_client optional;
  
 root /var/www;
  
 location / {
 # if the client-side certificate failed to authenticate, show a 403
 # message to the client
 if ($ssl_client_verify != SUCCESS) {
 return 403;
 }
  
 }
}
```

The configuration is pretty straightforward, we just specify the `ssl_certificate` and `ssl_certificate_key` to configure the server certificate, and we just give the `ca.crt` to `ssl_client_certificate` so that all trusted client keys are accepted.
  
Here, the SSL verification is set as optional and manually handled so that users get a `403` error instead of an SSL handshake error.
  
If you clone the project's repository, you can then run:

```shell
$ docker run --name nginx \
 -v $(pwd)/www:/var/www \
 -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
 -v $(pwd)/certs:/etc/nginx/certs \
 -p 80:80 \
 -p 443:443 \
 --rm \
 nginx
```

## Accessing the resource

Now that we have our NGinx server up and running, we need to access the resources using our client certificate. The first thing to do is to be able to resolve the address https://nginx.local to 127.0.0.1. In order to do that we have to choices:

- add a `127.0.0.1 nginx.local` entry in `/etc/hosts`
- use `dnsmasq` to have a more flexible solution following the [documentation here](https://passingcuriosity.com/2013/dnsmasq-dev-osx/)
  
Once we can ping `nginx.local`, we can use `curl`, trying to access our resource:
  
```shell
$ curl https://nginx.local
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

By not providing any kind of key, we are not able to access the resource. If now we provide the key and certificate to `curl`:
  
```shell
$ curl --cert client.crt --key client.key  https://nginx.local
<!DOCTYPE html>
<html>
 <head>
 <title>Example</title>
 </head>
 <body>
 <p>This is an example of a simple HTML page with one paragraph.</p>
 </body>
</html>
```
  
then we can access the secured resource.

## Conclusion
In this part, we have configured an NGinx server that secures its resource using mutual authentication. In this example, we generated all our certificates and were able to sign the keys by ourselves. In a real-world scenario, the server cannot self sign its certificate if he wants to be trusted, and it is the same for the client. 

For the server part, you would usually generate a key and the CSR, then send the CSR to a third-party trusted authority like [Gandi](https://www.gandi.net/fr) or [GoDaddy](https://www.godaddy.com/), that would then sign your certificate. Another option would be to use the ACME protocol and [Let's Encrypt](https://letsencrypt.org/).

For the client part, as you manage the server, you may decide on a scenario where the clients would send you some CSR, and you would sign them with a CA that you trust. You don't need a third party there.

[In the second part]({% post_url 2021-07-28-mutual-auth-with-java-part2 %}) of this series, we will see how to use the client certificate in a Java application.

## References
- `Github Repository:` https://github.com/dmetzler/mutual-auth-with-java
