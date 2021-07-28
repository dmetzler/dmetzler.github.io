---
tags: 
 - blog
layout: post
title: Mutual Authentication with NGinx and Java (Part 2)
excerpt: |
 User and password is a way to authenticate to an application. Let's discover another way with mutual authentication.

img_url: /images/2021-07-27-mutual-auth-with-java-part1.png

---
## Context

In [the first part of this serie]({% post_url 2021-07-27-mutual-auth-with-java-part1 %}), we configured an NGinx server with mutual authentication and we were able to do some requests on it using `cURL`.
In Java, the way certificates are managed can be sometimes cumbersome. In this post, we will see how to request a web resource in Java, using mutual authentication and different kind of key stores. 


## Java, Key, And Certificate Formats

In part1, we generated some keys in [PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) format. This format encodes the certificate parts in base64 and encloses the result between markers. For instance:

```
-----BEGIN PRIVATE KEY-----
MIIJQwIBADANBgkqhkiG9w0B...
...
-----END PRIVATE KEY-----
```

Certificates are also encoded in [PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) like this:

```
-----BEGIN CERTIFICATE-----
MIIFCTCCAvGgAwIBAgIULqd76vqSzSWiFDbPe99gNfmZvjQwDQYJKoZIhvcNAQEL
...
-----END CERTIFICATE-----
```

The Java SDK offers a [`java.security.Keystore`](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/security/KeyStore.html) object that allows storing keys and certificates. In order to store a key, we need the key part and the chain of certificates.

Before being able to read a key, it needs to be converted in the PKCS8 format with openssl as the JDK by itself doesn't know how to read PEM format directly:

```shell
openssl pkcs8 -topk8 -inform PEM -in client.key -out client.pk8 -nocrypt
```

Once we have that `client.pk8` file, it still needs to be read and modified before being able to instanciate a `KeySpec`:

```java
private KeySpec readKeyFromResource(URL resource) throws IOException {
        try (var in = resource.openStream()) {

            var text = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8))
                            .lines()
                            .collect(Collectors.joining("\n"));

            var privateKeyPEM = text.replaceAll("-----BEGIN (.*)-----", "")
                                    .replaceAll(System.lineSeparator(), "")
                                    .replaceAll("-----END (.*)-----", "");

            return new PKCS8EncodedKeySpec(Base64.getDecoder().decode(privateKeyPEM));

        }
    }
```

In order to read the certificate chain, it's a bit easier as Java provides a factory to that matter:

```java
private List<Certificate> readCertificatesFromResource(URL certResource) throws IOException, CertificateException {
        try (var in = certResource.openStream()) {
            var certFactory = CertificateFactory.getInstance("X.509");
            Collection<? extends Certificate> certificates = certFactory.generateCertificates(in);
            if (certificates.isEmpty()) {
                throw new IllegalArgumentException("expected non-empty set of trusted certificates");
            }
            return certificates.stream().map(Certificate.class::cast).collect(Collectors.toList());

        }

    }
```

Now that we have a `KeySpec` and a `Certificate` we can add it to our `Keystore`:

```java
var certs = readCertificatesFromResource(certResource);
var keySpec = readKeyFromResource(keyResource);
var kf = KeyFactory.getInstance("RSA");
var privateKey = kf.generatePrivate(keySpec);

keyStore.setKeyEntry("client", privateKey, DEFAULT_TRANSIENT_PASSWORD, certs.toArray(new Certificate[] {}));

```

<div class="admonition note">
<p class="admonition-title">Note</p>
<p>The code here refer to <code>DEFAULT_TRANSIENT_PASSWORD</code> which is the password of an inmemory <code>KeyStore</code>. Please look at the code in the Github repository to understand how to create that inmemory keystore</p>
</div>

    
Using those different methods, we are now able to assemble a `Keystore` containing all the required pieces to handle our mutual TLS handshake. These are some handy methods when you want to keep your keys and certificates in different PEM files. 

However, storing keys can be problematic, especially as they give access to protected resources. So, directly using those keys may be problematic. 


## Keystore and Keytool

To overcome this problem, Java offers a mechanism to store keys in a `KeyStore` backed by a file and protected by a password. The JDK ships with the utility `keytool` that can create or import keys in that keystore. Again, there are some format adjustments to run before being able to import our keys and certificates.

Let's start with our client key/certificate pair. We first need to export that pair in the [PKCS12](https://en.wikipedia.org/wiki/PKCS_12) format to be able to import it into our keystore. After that, we can import it in a new keystore file.

```shell
$ PASSWORD=changeit
$ openssl pkcs12 -export \
        -in certs/client.crt \
        -inkey certs/client.key \
        -out certs/client.p12 \
        -name client \
        -CAfile certs/ca.crt \
        -caname my-own-ca \
        -passout pass:$PASSWORD
$ keytool -importkeystore \
          -deststorepass $PASSWORD \
          -destkeypass $PASSWORD \
          -destkeystore certs/client.jks \
          -srckeystore certs/client.p12 \
          -srcstoretype PKCS12 \
          -srcstorepass $PASSWORD \
          -alias client

Importing keystore certs/client.p12 to certs/client.jks...
```

Now that we have imported our client key in the keystore, we need to add the two CA certificates. The `keytool` utility require X509 certificates to be store in `der` format before being imported. Here is how to do it:

```shell
$ openssl x509 -outform der -in certs/ca.crt  -out certs/ca.der
$ keytool -import \
          -noprompt \
          -alias clientca \
          -deststorepass $PASSWORD \
          -keystore certs/client.jks \
          -file certs/ca.der
Certificate was added to keystore

$ openssl x509 \
    -outform der \
    -in "$(mkcert -CAROOT)/rootCa.pem"  \
    -out certs/server.der
$ keytool \
    -import \
    -noprompt \
    -alias serverca \
    -deststorepass $PASSWORD \
    -keystore certs/client.jks \
    -file certs/server.der
Certificate was added to keystore
```

We can now list the content of our keystore and verify that we have one `PrivateKeyEntry` and two `trustedCertEntry`:

```shell
$ keytool -list -keystore certs/client.jks
Enter keystore password:
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 3 entries

client, 28 Jul 2021, PrivateKeyEntry,
Certificate fingerprint (SHA-256): D0:17:71:78:C7:C2:0A:10:2A:D2:6C:FF:F4:08:30:D1:48:AF:78:50:57:CD:DC:EB:DF:38:7D:7D:58:53:0F:B3
clientca, 28 Jul 2021, trustedCertEntry,
Certificate fingerprint (SHA-256): CC:D1:28:89:72:C2:B5:5E:22:2B:D0:88:0A:05:B7:5E:53:E3:4F:0F:6B:28:C7:24:03:CA:F6:C7:24:8E:93:51
serverca, 28 Jul 2021, trustedCertEntry,
Certificate fingerprint (SHA-256): DD:9B:E9:5A:8D:04:F7:AF:93:94:D7:C7:8F:B9:05:49:BD:C5:95:A5:11:C7:96:19:51:A7:09:6F:5C:9F:81:6C
```

Once we have our JKS file, it is straightforward to build a `Keystore` in java:

```java
try (var in = jksUrl.openStream()) {
    var ks = KeyStore.getInstance("JKS");
    ks.load(in, "changeit".toCharArray());
    return ks;
}
```

This saves a lot of boilerplate code compared to when you need to read each file individually.
Now that we have a Keystore populated with all the keys and certificates that we need, we can configure our SSL connection so that it makes use of them.

## TrustManager and SSL Context

In order to run some request on our endpoint, we will use the [OkHttp](https://github.com/square/okhttp) library. 

The only thing that we need is to provide an [`SSLSocketFactory`](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/javax/net/ssl/SSLSocketFactory.html) and an [`X509TrustManager`](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/javax/net/ssl/X509TrustManager.html) to the OkHttp builder. Here is how to do:

```java
public OkHttpClient build() {

    try {
        var ks = buildKeyStore();

        var kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        kmf.init(ks, password);
        var tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(ks);

        var sslContext = SSLContext.getInstance("TLS");
        sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), new SecureRandom());

        var trustManager = getX509TrustManager(tmf);

        return new OkHttpClient.Builder() //
                                         .sslSocketFactory(sslContext.getSocketFactory(), trustManager)
                                         .build();

    } catch (IOException | KeyManagementException | NoSuchAlgorithmException | UnrecoverableKeyException
            | KeyStoreException e) {
        return null;
    }

}

private X509TrustManager getX509TrustManager(TrustManagerFactory tmf) {
    TrustManager[] trustManagers = tmf.getTrustManagers();
    if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
        throw new IllegalStateException("Unexpected default trust managers:" + Arrays.toString(trustManagers));
    }
    return (X509TrustManager) trustManagers[0];
}


```

Now that we have our client object, we can run our requests. By using the code in the linked Github repository, you can look at the tests and see how that works:


```java
@Test
public void mutual_tls_builder() throws Exception {
    var client = new MutualTlsClientBuilder()//
                         .withCA(getUrl("/ca.crt"))//
                         .withClientKey(getUrl("/client.pk8"))//
                         .withClientCert(getUrl("/client.crt"))//
                         .build();

    Request request = new Request.Builder().url(URL).build();

    try (Response response = client.newCall(request).execute()) {
        assertThat(response.code()).isEqualTo(200);
    }
}

@Test
public void mutual_tls_build_with_jks() throws Exception {
    var client = new MutualTlsClientBuilder()//
                         .withJKS(getUrl("/client.jks"))//
                         .withPassword("changeit".toCharArray())
                         .build();

    Request request = new Request.Builder().url(URL).build();

    try (Response response = client.newCall(request).execute()) {
        assertThat(response.code()).isEqualTo(200);
    }
}
```


## Conclusion

In this second part, we have seen how to use the certificates that we created in the first part in a Java application. Java has its own way of dealing with certificates and it can be sometimes difficult to find its way between all the keys and certificate chains.

This step-by-step guide showed how to create the keys and certificates to deploy a secured server and how to run some requests with OkHttp and mutual authentication. 

The [Github repository](https://github.com/dmetzler/mutual-auth-with-java) linked in references, contains the complete working source code of all examples that were given, feel free to use it in your own development of course.

## References
- [Part 1]({% post_url 2021-07-27-mutual-auth-with-java-part1 %})
- `Github Repository:` https://github.com/dmetzler/mutual-auth-with-java
