Key Stores description and passwords
====================================

### Server Key Store
* File: `serverKeyStore.jks`
* Password: `server`
* Contains:
  * `rootCA` : the trusted root CA.
  * `untrustedrootCA` : an untrusted root CA.
  * `server` : server keys and certificate, signed by root CA.
  * `server_self_signed` : a self-signed server certificate. 

### Client Key Store
* File: `clientKeyStore.jks`
* Password: `client`
* Contains:
  * `client` : client keys and certificate, signed by root CA with a valid CN.
  * `client_bad_cn` : a client certificate signed by root CA but with bad CN.
  * `client_self_signed` : a self-signed client certificate with a valid CN.
  * `client_not_trusted` : a client certificate signed by untrusted root CA with a valid CN.

Create keys and certificates !
==============================

Here is the commands use to create it using `openssl`.  
:warning: We are using `process substitution` to allow to launch some `openssl` command without creating file, so you could maybe face issue using a none-unix system ...

## Create root CA credentials

### Trusted root CA
```
openssl ecparam -out rootCA.pem -name prime256v1 -genkey
openssl req -x509 -new -nodes -key rootCA.pem -sha256 -days 36500 -out rootCA.crt
```

### Untrusted root CA
```
openssl ecparam -out untrustedrootCA.pem -name prime256v1 -genkey
openssl req -x509 -new -nodes -key untrustedrootCA.pem -sha256 -days 36500 -out untrustedrootCA.crt
```

## Create Server credentials
### Server private/public Keys
Those keys will used to create all server certificates.
```
openssl ecparam -out server.pem -name prime256v1 -genkey
```

### Server certificate signed by the trusted root CA
```
openssl req -new -key server.pem  -out server.csr
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.pem -CAcreateserial \
                  -out server.crt -days 36500 -sha256
```

### Self-signed Server certificate
```
openssl req -x509 -new -nodes -key server.pem -sha256 -days 36500 -out server_self_signed.crt
```

## Create Client credentials
### Client private/public Keys
Thoses keys will be used to create all client certificates.
 
```
openssl ecparam -out client.pem -name prime256v1 -genkey
```

### Client certificate signed by the trusted root CA
During create of the Certificate Signing Request (CSR), choose a CN which start by `leshan_integration_test`.  
:warning: We are using `process substitution` :  `<(printf "keyUsage = digitalSignature")` to avoid to create a file needed for `-extfile` option.

```
openssl req -new -key client.pem  -out client_bad_cn.csr 
openssl x509 -req -in client.csr -CA rootCA.crt \
             -extfile <(printf "keyUsage = digitalSignature") \
             -CAkey rootCA.pem -CAcreateserial -out client.crt -days 36500 -sha256
```

### Client certificate signed by the trusted root CA with a bad CN
During create of the Certificate Signing Request (CSR), choose a CN which **does not start** by `leshan_integration_test`.  
:warning: We are using `process substitution` :  `<(printf "keyUsage = digitalSignature")` to avoid to create a file needed for `-extfile` option.

```
openssl req -new -key client.pem  -out client_bad_cn.csr 
openssl x509 -req -in client.csr -CA rootCA.crt \
             -extfile <(printf "keyUsage = digitalSignature") \
             -CAkey rootCA.pem -CAcreateserial -out client.crt -days 36500 -sha256
```

### Self-signed Client certificate
During create of the Certificate Signing Request (CSR), choose a CN which start by `leshan_integration_test`.  
:warning: We are using `process substitution` :  `-config <(cat /etc/ssl/openssl.cnf <(printf "[v3_req]\nkeyUsage = digitalSignature"))` to avoid to create a file needed for `-config` option.  
:warning: You maybe need to use your system default openssl.cnf path.

```
openssl req -x509 -new -nodes -extensions 'v3_req' \
            -config <(cat /etc/ssl/openssl.cnf <(printf "[v3_req]\nkeyUsage = digitalSignature")) \
            -key client.pem -sha256 -days 36500 -out client_self_signed.crt
```

### Client certificate signed by the untrusted root CA
We can reuse the CSR from client certificate signed by the trusted root CA.  
:warning: We are using `process substitution` :  `<(printf "keyUsage = digitalSignature")` to avoid to create a file needed for `-extfile` option.

```
openssl x509 -req -in client.csr -CA untrustedrootCA.crt \
             -extfile <(printf "keyUsage = digitalSignature") \
             -CAkey untrustedrootCA.pem -CAcreateserial -out client_not_trusted.crt -days 36500 -sha256
```
Import in java keystore
=======================
Here is the commands use to create keystore and import certificate in it using `openssl` and `keytool`.

## server keystore
### import server keys
To import keys in java keystore you need to create a pkcs12 keystore with openssl first ...  
:warning: Use `server` password

```
openssl pkcs12 -export -in server.crt -inkey server.pem \
               -out server.p12 -name server 
```
Now you can import it in a keystore.

```
keytool -importkeystore \
        -srckeystore server.p12 -srcstoretype PKCS12 -srcstorepass server \
        -deststorepass server  -destkeypass server -destkeystore serverKeyStore.jks \
        -alias server
```

### import server certificates
Don't forget password is `server`.

```
keytool -import -alias rootCA -file rootCA.crt -keystore serverKeyStore.jks
keytool -import -alias untrustedrootCA -file untrustedrootCA.crt -keystore serverKeyStore.jks
keytool -import -alias server_self_signed -file server_self_signed.crt -keystore serverKeyStore.jks
```

## client keystore
### import client keys
To import keys in java keystore you need to create a pkcs12 keystore with openssl first ...  
:warning: Use `client` password.

```
openssl pkcs12 -export -in client.crt -inkey client.pem \
               -out client.p12 -name client
```
Now you can import it in a keystore.

```
keytool -importkeystore \
        -srckeystore client.p12 -srcstoretype PKCS12 -srcstorepass client \
        -deststorepass client  -destkeypass client -destkeystore clientKeyStore.jks \
        -alias client
```

## import server certificates
Don't forget password is `client`.

```
keytool -import -alias client_bad_cn -file client_bad_cn.crt -keystore clientKeyStore.jks
keytool -import -alias client_self_signed -file client_self_signed.crt -keystore clientKeyStore.jks
keytool -import -alias client_not_trusted -file client_not_trusted.crt -keystore clientKeyStore.jks
```

Some Extra commands
===================
### see certificate content
```
openssl x509 -in certificate.crt -text -noout

```

### see content of keystore
```
keytool -list -keystore yourkeystore.jks
```

### remove entry from keystore
```
keytool -delete -alias entryname -keystore yourkeystore.jks 
```