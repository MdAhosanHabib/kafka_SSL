<img width="454" alt="ssl_sample" src="https://github.com/MdAhosanHabib/kafka_SSL/assets/43145662/5610b443-a776-4152-a154-956dfe3c39ff">

# Enabling SSL/TLS Encryption for Kafka with JKS

#### This repository contains instructions and configurations for enabling SSL/TLS encryption in a Kafka environment using Java KeyStores (JKS).

## Table of Contents
1. Prerequisites

2. Server-Side Configuration

3. Client-Side Configuration

4. Testing the Client


## Prerequisites
Running Kafka instance

Access to Open SSL and keytool utilities

Basic understanding of SSL/TLS and Kafka concepts

#### Overview of Certificate follows:
![diagram_For_Kafka-Page-2](https://github.com/MdAhosanHabib/kafka_SSL/assets/43145662/3a33efaa-af24-4161-9361-293b7682e8ff)


## Server-Side Configuration

#### Generate CA self-signed certificate
```bash
[root@testkafka ssl]# mkdir /usr/local/kafka/config/ssl/kafka
[root@testkafka ssl]# cd /usr/local/kafka/config/ssl/kafka
[root@testkafka kafka]# cat /etc/hosts
10.0.0.6        testkafka               testkafka.com
10.0.0.6         client                 client.com
[root@testkafka kafka]#

[root@testkafka kafka]# openssl genpkey -algorithm RSA -out /usr/local/kafka/config/ssl/kafka/ca.key
..........++++++
................................++++++
[root@testkafka kafka]#

[root@testkafka kafka]# openssl req -x509 -new -key /usr/local/kafka/config/ssl/kafka/ca.key -days 3650 -out /usr/local/kafka/config/ssl/kafka/ca.crt
Country Name (2 letter code) [XX]:BD
State or Province Name (full name) []:Bangladesh
Locality Name (eg, city) [Default City]:Dhaka
Organization Name (eg, company) [Default Company Ltd]:TEST Ltd
Organizational Unit Name (eg, section) []:DevOps
Common Name (eg, your name or your server's hostname) []:testkafka.com
Email Address []:ahosan@test.com
[root@testkafka kafka]#
```

#### Generate Server Private Key and CSR
```bash
[root@testkafka kafka]# openssl req -new -newkey rsa:4096 -nodes -keyout /usr/local/kafka/config/ssl/kafka/server.key \
-out /usr/local/kafka/config/ssl/kafka/server.csr \
-subj "/C=BD/ST=Bangladesh/L=Dhaka Dhaka/O=TEST Ltd
Inc/CN=testkafka.com/emailAddress=ahosan@test.com"
```

#### Generate the server certificate using the CSR, the CA cert, and private key
```bash
[root@testkafka kafka]# vi /usr/local/kafka/config/ssl/kafka/san.cnf
  authorityKeyIdentifier=keyid,issuer
  basicConstraints=CA:FALSE
  keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
  subjectAltName = @alt_names
  
  [alt_names]
  DNS.1=testkafka.com
  DNS.2=*.testkafka.com

[root@testkafka kafka]# openssl x509 -req -in /usr/local/kafka/config/ssl/kafka/server.csr -CA /usr/local/kafka/config/ssl/kafka/ca.crt \
-CAkey /usr/local/kafka/config/ssl/kafka/ca.key -CAcreateserial -out /usr/local/kafka/config/ssl/kafka/server.crt \
-days 3650 -extfile /usr/local/kafka/config/ssl/kafka/san.cnf
```

#### Create Kafka format Keystore
```bash
[root@testkafka kafka]# openssl pkcs12 -export \
-in /usr/local/kafka/config/ssl/kafka/server.crt \
-inkey /usr/local/kafka/config/ssl/kafka/server.key \
-name kafka-broker \
-out /usr/local/kafka/config/ssl/kafka/kafka.p12
#Enter Export Password: testkafka123
```

#### Create Kafka Java KeyStore (JKS)
```bash
[root@testkafka kafka]# keytool -importkeystore \
-srckeystore /usr/local/kafka/config/ssl/kafka/kafka.p12 \
-destkeystore /usr/local/kafka/config/ssl/kafka/kafka.keystore.jks \
-srcstoretype pkcs12
#Enter destination keystore password: testkafka123
#Enter source keystore password: testkafka123
```

#### Create Kafka TrustStore
```bash
[root@testkafka kafka]# keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file /usr/local/kafka/config/ssl/kafka/ca.crt
#Enter keystore password: testkafka123
#Trust this certificate? [no]:  yes

#-- Confirm your keystore/truststore details
[root@testkafka kafka]# keytool -list -v -keystore /usr/local/kafka/config/ssl/kafka/kafka.keystore.jks
[root@testkafka kafka]# keytool -list -v -keystore /usr/local/kafka/config/ssl/kafka/kafka.server.truststore.jks
```

#### Configure Apache Kafka SSL/TLS Encryption
```bash
[root@testkafka config]# vi /usr/local/kafka/config/server.properties
############################# Socket Server Settings #############################
listeners=PLAINTEXT://testkafka.com:9093,SSL://testkafka.com:9092
inter.broker.listener.name=SSL
ssl.endpoint.identification.algorithm=
advertised.listeners=PLAINTEXT://testkafka.com:9093,SSL://testkafka.com:9092

ssl.keystore.location=/usr/local/kafka/config/ssl/kafka/kafka.keystore.jks
ssl.keystore.password=testkafka123
ssl.key.password=testkafka123
ssl.truststore.location=/usr/local/kafka/config/ssl/kafka/kafka.server.truststore.jks
ssl.truststore.password=testkafka123
ssl.client.auth=required

[root@testkafka DataStream]# systemctl restart kafka
[root@testkafka DataStream]# systemctl status kafka
```

## Client-Side Configuration
#### Generate client Private Key and CSR
```bash
[root@testkafka ssl]# mkdir /usr/local/kafka/config/ssl/client
[root@testkafka ssl]# cd /usr/local/kafka/config/ssl/client
[root@testkafka kafka]# cat /etc/hosts
10.0.0.6        testkafka               testkafka.com
10.0.0.6         client                 client.com
[root@testkafka kafka]#

[root@testkafka kafka]# openssl req -new -newkey rsa:4096 -nodes -keyout /usr/local/kafka/config/ssl/client/client.key \
-out /usr/local/kafka/config/ssl/client/client.csr \
-subj "/C=BD/ST=Bangladesh/L=Dhaka Dhaka/O=TEST Ltd
Inc/CN=client.com/emailAddress=ahosan@test.com"
```

#### Generate the client certificate using the CSR, the CA cert, and private key
```bash
[root@testkafka kafka]# vi /usr/local/kafka/config/ssl/client/san.cnf
  authorityKeyIdentifier=keyid,issuer
  basicConstraints=CA:FALSE
  keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
  subjectAltName = @alt_names
  
  [alt_names]
  DNS.1=client.com
  DNS.2=*.client.com

[root@testkafka kafka]# openssl x509 -req -in /usr/local/kafka/config/ssl/client/client.csr -CA /usr/local/kafka/config/ssl/kafka/ca.crt \
-CAkey /usr/local/kafka/config/ssl/kafka/ca.key -CAcreateserial -out /usr/local/kafka/config/ssl/client/client.crt \
-days 3650 -extfile /usr/local/kafka/config/ssl/client/san.cnf
```

#### Create Kafka format Keystore
```bash
[root@testkafka kafka]# openssl pkcs12 -export \
-in /usr/local/kafka/config/ssl/client/client.crt \
-inkey /usr/local/kafka/config/ssl/client/client.key \
-name kafka-broker \
-out /usr/local/kafka/config/ssl/client/client.p12
#Enter Export Password: testclient123
```

#### Create Kafka Java KeyStore (JKS)
```bash
[root@testkafka kafka]# keytool -importkeystore \
-srckeystore /usr/local/kafka/config/ssl/client/client.p12 \
-destkeystore /usr/local/kafka/config/ssl/client/kafka.keystore.jks \
-srcstoretype pkcs12
#Enter destination keystore password: testclient123
#Enter source keystore password: testclient123
```

#### Create Kafka TrustStore
```bash
[root@testkafka kafka]# keytool -keystore /usr/local/kafka/config/ssl/client/kafka.server.truststore.jks -alias CARoot -import -file /usr/local/kafka/config/ssl/kafka/ca.crt
#Enter keystore password: testclient123
#Trust this certificate? [no]:  yes


#-- Confirm your keystore/truststore details
[root@testkafka kafka]# keytool -list -v -keystore /usr/local/kafka/config/ssl/client/kafka.keystore.jks
[root@testkafka kafka]# keytool -list -v -keystore /usr/local/kafka/config/ssl/client/kafka.server.truststore.jks
```

## Testing the Client
#### Test Client
```bash
[root@testkafka config]# mv /usr/local/kafka/config/kafka-client-ssl-properties /usr/local/kafka/config/kafka-client-ssl-properties-bkp
[root@testkafka config]# vi /usr/local/kafka/config/kafka-client-ssl-properties
security.protocol=SSL
ssl.keystore.location=/usr/local/kafka/config/ssl/client/kafka.keystore.jks
ssl.keystore.password=testclient123
ssl.key.password=testclient123
ssl.truststore.location=/usr/local/kafka/config/ssl/client/kafka.server.truststore.jks
ssl.truststore.password=testclient123

[root@testkafka config]# cd /usr/local/kafka
[root@testkafka config]# bin/kafka-topics.sh --list --bootstrap-server testkafka.com:9092 --command-config /usr/local/kafka/config/kafka-client-ssl-properties
```

Regards from,
Ahosan Habib.
