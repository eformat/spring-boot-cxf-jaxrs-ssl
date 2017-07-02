# Spring-Boot CXF JAXRS SSL QuickStart

Example REST Apache CXF, SpringBoot, Using SSL/TLS1.2 and MutualSSL / ClientCertMandatory policy

### Generate certificates
```
# Generating the Server Keystore

keytool -genkeypair -alias secure-server -keyalg RSA -dname "CN=DEV, OU=DEV, O=ACME, L=Melbourne, ST=VIC, C=AU" -keypass password -keystore server-keystore.jks -storepass password

# Export the server public certificate: 

keytool -exportcert -alias secure-server -file server-public.cer -keystore server-keystore.jks -storepass password

# Import it in the client truststore:

keytool -importcert -keystore client-truststore.jks -alias servercert -file server-public.cer -storepass password

# if you want to access https://localhost:8443 from the browser you'll have to install cert
# To export it: 

keytool -importkeystore -srckeystore client-keystore.jks -destkeystore codependent-client.p12 -deststoretype PKCS12


# Create a CA
openssl req -new -x509 -keyout codependent-ca-key.pem -out codependent-ca.pem -days 365

# For every client: 

# Create the server truststore including the client CA

keytool -import -alias codependent-ca -file codependent-ca.pem -keystore server-truststore.jks -storepass password

# Generate an unsigned client certificate for secure-client

keytool -genkeypair -alias secure-client -keyalg RSA -dname "CN=client1, OU=DEV, O=ACME, L=Melbourne, ST=VIC, C=AU" -keypass password -keystore client-keystore.jks -storepass password

# Generate the Certificate Signing Request:

keytool -keystore client-keystore.jks -certreq -alias secure-client -keyalg rsa -storepass password > unsigned-client.csr

# Generate a signed certificate for the associated Certificate Signing Request:

openssl x509 -req -CA codependent-ca.pem -CAkey codependent-ca-key.pem -in unsigned-client.csr -out signed-client.cer -days 365 -CAcreateserial

# You get a signed-client.cer file.

# Import the CA and the client signed certificate into the client keystore:

keytool -import -keystore client-keystore.jks -file codependent-ca.pem -alias codependent-ca
keytool -import -keystore client-keystore.jks -file signed-client.cer -alias secure-client
```

### Run locally

```
mvn clean spring-boot:run
```

### Deploy OpenShift

```
oc secrets new spring-boot-cxf-jaxrs-keystores ./secrets

echo -n "password" > ./password.txt
oc secrets new spring-boot-cxf-jaxrs-passwords server.ssl.key-store-password=password.txt server.ssl.trust-store-password=password.txt

mvn clean fabric8:deploy
```

### Notes on Configuration
Uses application.properties for local run

Configuration is via [Resource Fragments](https://maven.fabric8.io/#resource-fragments)

- application properties are deployed from src/main/fabric8/configmap.yml
- deployment from src/main/fabric8/deployment.yml
- service from src/main/fabric8/service.yml

The health checks are normally generated by maven fabric8 plugin as ssl if you set *server.ssl.enabled*, however we are using client cert mandatory policy, and we override the spring managment health checks as http only as they are internal to application (not exposed as routes).

TODO - The keystore passwords are mounted as secrets, but not being read as part of application.properties (yet)