---
title: Kafka Encryption with SSL/TLS
date: 2022-12-20T00:00:00+09:00
categories:
    - Kafka
    - Confluent
tags:
    - kafka
    - ssl/tls
    - encryption
draft: true
---


# Encryption with SSL/TLS

## mTLS kafka

```bash
#!/usr/bin/env bash
############### Parameters ###############
PASSWORD='password'
VALIDITY=730
KAFKA_BROKER_COUNT=3
###############  Existing certs Cleanup ###############
rm -rf ./certs && mkdir certs
cd certs
 
###############  Creating sslconfig file for CA ###############
cat > "openssl.cnf" << EOF
[req]
default_bits = 4096
encrypt_key  = no # Change to encrypt the private key using des3 or similar
default_md   = sha256
prompt       = no
utf8         = yes
# Specify the DN here so we aren't prompted (along with prompt = no above).
distinguished_name = req_distinguished_name
# Extensions for SAN IP and SAN DNS
req_extensions = v3_req
# Be sure to update the subject to match your organization.
[req_distinguished_name]
C  = KR
ST = SEOUL
L  = Confluent
O  = Organization
OU = OrganizationUnit/emailAddress=my@email.com
CN = Kafka-Security-CA
# Allow client and server auth. You may want to only allow server auth.
# Link to SAN names.
[v3_req]
basicConstraints     = CA:TRUE
subjectKeyIdentifier = hash
keyUsage             = digitalSignature, keyEncipherment
extendedKeyUsage     = clientAuth, serverAuth
EOF
###############  Creating CA with above config  ###############
openssl req -new -x509 -keyout ca.key -out ca.crt -days $VALIDITY -config openssl.cnf
 
#################################################
########### Server Certificates #################
#################################################
 
 
###############  Generate certificates for each broker  ###############
for i in `seq 0 $(( $KAFKA_BROKER_COUNT-1))`; do
echo Generating for kafka $i
 
###############  Create certificate and store in keystore  ###############
keytool -keystore mtls-kafka-$i.keystore.jks -alias mtls-kafka-$i -validity $VALIDITY -genkey -keyalg RSA -ext SAN=dns:mtls-kafka-$i.mtls-kafka-headless.axplatform.svc.cluster.local,dns:mtls-kafka.mtls-kafka-headless.axplatform.svc.cluster.local,dns:mtls-kafka-$i.mtls-kafka-headless,dns:mtls-kafka-$i.mtls-kafka-headless.axplatform,dns:mtls-kafka-$i.mtls-kafka-headless,dns:mtls-kafka -storepass ${PASSWORD} -noprompt -dname "CN=mtls-kafka-$i.mtls-kafka-headless, OU=OrganizationUnit, O=Organization, L=Confluent, S=Seoul, C=KR"
 
###############  Create certificate signing request(CSR)  ###############
keytool -keystore mtls-kafka-$i.keystore.jks -alias mtls-kafka-$i -certreq -file ca-request-mtls-kafka-$i -storepass ${PASSWORD} -noprompt
 
###############  Create ssl config file with DNS names anf other params  ###############
cat > ext$i.cnf << EOF
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = mtls-kafka-$i.mtls-kafka-headless.axplatform.svc.cluster.local
DNS.2 = mtls-kafka-$i.mtls-kafka-headless
DNS.3 = mtls-kafka-$i.mtls-kafka-headless.axplatform
DNS.4 = mtls-kafka.mtls-kafka-headless
DNS.5 = mtls-kafka
EOF
 
###############  Sign CSR with CA  ###############
openssl x509 -req -CA ca.crt -CAkey ca.key -in ca-request-mtls-kafka-$i -out ca-signed-mtls-kafka-$i -days $VALIDITY -CAcreateserial -extfile ext$i.cnf
 
###############  Import CA certificate in keystore   ###############
keytool -keystore mtls-kafka-$i.keystore.jks -alias ca.crt -import -file ca.crt -storepass ${PASSWORD} -noprompt
 
###############  Import Signed certificate in keystore   ###############
keytool -keystore mtls-kafka-$i.keystore.jks -alias mtls-kafka-$i -import -file ca-signed-mtls-kafka-$i -storepass ${PASSWORD} -noprompt
done
 
###############  Import CA certificate in truststore   ###############
keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file ca.crt -storepass $PASSWORD -keypass $PASSWORD -noprompt
 
#################################################
########### Client Certificates #################
#################################################
 
###############  Create certificate and store in keystore  ###############
keytool -keystore mtls-kafka-client.keystore.jks -alias mtls-kafka-client -validity $VALIDITY -genkey -keyalg RSA -ext SAN=dns:mtls-kafka-client -storepass ${PASSWORD} -noprompt -dname "CN=mtls-kafka-client, OU=OrganizationUnit, O=Organization, L=Confluent, S=Seoul, C=KR"
 
###############  Create certificate signing request(CSR)  ###############
keytool -keystore mtls-kafka-client.keystore.jks -alias mtls-kafka-client -certreq -file ca-request-mtls-kafka-client -storepass ${PASSWORD} -noprompt
 
###############  Create ssl config file with DNS names anf other params  ###############
cat > extclient.cnf << EOF
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = mtls-kafka-client
EOF
 
###############  Sign CSR with CA  ###############
openssl x509 -req -CA ca.crt -CAkey ca.key -in ca-request-mtls-kafka-client -out ca-signed-mtls-kafka-client -days $VALIDITY -CAcreateserial -extfile extclient.cnf
 
###############  Import CA certificate in keystore   ###############
keytool -keystore mtls-kafka-client.keystore.jks -alias ca.crt -import -file ca.crt -storepass ${PASSWORD} -noprompt
 
###############  Import Signed certificate in keystore   ###############
keytool -keystore mtls-kafka-client.keystore.jks -alias mtls-kafka-client -import -file ca-signed-mtls-kafka-client -storepass ${PASSWORD} -noprompt
 
###############  Import CA certificate in truststore   ###############
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file ca.crt -storepass $PASSWORD -keypass $PASSWORD -noprompt

```

  

# Inter-Communication with components

![](img/image.png)

  

- 카프카는 기본적으로 PLAINTEXT 로 통신한다.
    
- Secure Sockets Layer (SSL) : the Processor of Transport Layer Security (TLS)
    

    - Deprecated since June 2015, 하지만 일반적으로 SSL를 TLS 대신 계속 사용
    
    - SSL을 사용하는 것은 encryption 부담이 더 해진다
    

- Client Authentication 에 SASL 을 사용할 수 있다.
    

  

![](img/image%202.png)

## 1\. SSL Keystore 및 Certificates 생성

![](img/image%203.png)

### 1-1. Certificate Authority (CA) 설정 파일 생성
```
\[ policy\_match \]
countryName = match
stateOrProvinceName = match
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

\[ req \]
prompt = no
distinguished\_name = dn
default\_md = sha256
default\_bits = 4096
x509\_extensions = v3\_ca

\[ dn \]
countryName = {country code, e.g., US}
organizationName = {organization name}
localityName = {locality, e.g., city name}
commonName = {certificate common name, e.g., <company name>-ca}

\[ v3\_ca \]
subjectKeyIdentifier=hash
basicConstraints = critical,CA:true
authorityKeyIdentifier=keyid:always,issuer:always
keyUsage = critical,keyCertSign,cRLSign
```

### 1-2. CA key 및 Certificate 생성 후 pem format 으로 변경
```
openssl req -new -nodes \\
  -x509 \\
  -days {validity} \\
  -newkey rsa:2048 \\
  -keyout ca.key \\
  -out ca.crt \\
  -config ca.cnf
  

cat ca.crt ca.key > ca.pem
```
### 1-3.  server key 및 Certificate signing request (.csr 파일)
```
openssl req -new \\
    -newkey rsa:2048 \\
    -keyout kafka.server.key \\
    -out kafka.server.csr \\
    -config kafka.server.cnf \\
    -nodes
```

### 1-4. Server certificate 를 CA 로 sign 하기
```
openssl x509 -req \\
    -days {validity} \\
    -in kafka.server.csr \\
    -CA ca.crt \\
    -CAkey ca.key \\
    -CAcreateserial \\
    -out kafka.server.crt \\
    -extfile kafka.server.cnf \\
    -extensions v3\_req
```

### 1-5. Server certificate 를 pkcs12 포멧으로 변경
```
openssl pkcs12 -export \\
    -in kafka.server.crt \\
    -inkey kafka.server.key \\
    -chain \\
    -CAfile ca.pem \\
    -name localhost \\
    -out kafka.server.p12 \\
    -password pass:test1234
```

### 1-6. server keystore 생성
```
sudo keytool -importkeystore \\
    -deststorepass test1234 \\
    -destkeystore kafka.server.keystore.pkcs12 \\
    -srckeystore kafka.server.p12 \\
    -deststoretype PKCS12  \\
    -srcstoretype PKCS12 \\
    -noprompt \\
    -srcstorepass test1234
```

##   

## 2\. Kafka Broker SSL Properties 설정

### Broker properties 파일에 keystore 설정 추가
```
ssl.keystore.location=/var/ssl/private/kafka.server.keystore.pkcs12
ssl.keystore.password=test1234
ssl.key.password=test1234
```  

## 3\. Kafka Broker SSL Listener 설정

### Kafka Broker SSL Listeners 설정
```
listeners=SSL://:9093,SASL\_SSL://:9094
```
  

- The SSL://:9093 listener does not require client authentication unless ssl.client.auth=required is also set
    
- The SASL\_SSL://:9094 listener requires SASL client authentication
    

  

  

# Hands-on

## kafkka broker server properties

- SSL Listener 추가
    
```
#KAFKA\_LISTENERS: PLAINTEXT://0.0.0.0:19092,BROKER://0.0.0.0:9092

#KAFKA\_ADVERTISED\_LISTENERS: PLAINTEXT://kafka-1-external:19092,BROKER://kafka-1:9092

KAFKA\_LISTENERS: PLAINTEXT://0.0.0.0:19092,SSL://0.0.0.0:19093,BROKER://0.0.0.0:9092

KAFKA\_ADVERTISED\_LISTENERS: PLAINTEXT://kafka-1-external:19092,SSL://kafka-1-external:19093,BROKER://kafka-1:9092
```
  

- listener.security.protocol.map 에 SSL 추가
    
```
KAFKA\_LISTENER\_SECURITY\_PROTOCOL\_MAP: PLAINTEXT:PLAINTEXT,SSL:SSL,BROKER:PLAINTEXT
```
  

## Broker Keystore

1. Certificate Authority (CA) Key 및 Certificate 생성
    
```
openssl req -new -nodes \\
  -x509 \\
  -days 365 \\
  -newkey rsa:2048 \\
  -keyout ~/tls/ca.key \\
  -out ~/tls/ca.crt \\
  -config ~/tls/ca.cnf
```
  

  
```
Generating a RSA private key

...................+++++

.........................+++++

writing new private key to '/home/training/tls/ca.key'

\-----
```
- ca.cnf
    
```
\[ policy\_match \]

countryName = match

stateOrProvinceName = match

organizationName = match

organizationalUnitName = optional

commonName = supplied

emailAddress = optional

  

\[ req \]

prompt = no

distinguished\_name = dn

default\_md = sha256

default\_bits = 4096

x509\_extensions = v3\_ca

  

\[ dn \]

countryName = US

organizationName = Confluent

localityName = MountainView

commonName = confluent-ca

  

\[ v3\_ca \]

subjectKeyIdentifier=hash

basicConstraints = critical,CA:true

authorityKeyIdentifier=keyid:always,issuer:always

keyUsage = critical,keyCertSign,cRLSign
```
  

  

1. CA Key 를 pem 포멧으로 변경
    
```
cat ~/tls/ca.crt ~/tls/ca.key > ~/tls/ca.pem
```
  

1. server key 및 certificate signing request (.csr file) 생성
    
```
openssl req -new \\

    -newkey rsa:2048 \\

    -keyout ~/tls/kafka-1-creds/kafka-1.key \\

    -out ~/tls/kafka-1-creds/kafka-1.csr \\

    -config ~/tls/kafka-1-creds/kafka-1.cnf \\

    -nodes
```
- kafka-1.cnf
    
```
\[req\]

prompt = no

distinguished\_name = dn

default\_md = sha256

default\_bits = 4096

req\_extensions = v3\_req

  

\[ dn \]

countryName = US

organizationName = CONFLUENT

localityName = MountainView

commonName=kafka-1

  

\[ v3\_ca \]

subjectKeyIdentifier=hash

basicConstraints = critical,CA:true

authorityKeyIdentifier=keyid:always,issuer:always

keyUsage = critical,keyCertSign,cRLSign

  

\[ v3\_req \]

subjectKeyIdentifier = hash

basicConstraints = CA:FALSE

nsComment = "OpenSSL Generated Certificate"

keyUsage = critical, digitalSignature, keyEncipherment

extendedKeyUsage = serverAuth, clientAuth

subjectAltName = @alt\_names

  

\[ alt\_names \]

DNS.1=kafka-1

DNS.2=kafka-1-external
```
  

1. Server certificate 를 CA로 sign 하기
    
```
openssl x509 -req \\

    -days 3650 \\

    -in ~/tls/kafka-1-creds/kafka-1.csr \\

    -CA ~/tls/ca.crt \\

    -CAkey ~/tls/ca.key \\

    -CAcreateserial \\

    -out ~/tls/kafka-1-creds/kafka-1.crt \\

    -extfile ~/tls/kafka-1-creds/kafka-1.cnf \\

    -extensions v3\_req
```

  

Signature ok

subject=C = US, O = CONFLUENT, L = MountainView, CN = Sca1.test.confluent.io

Getting CA Private Key

  

1. server certificate 를 pkcs12 포멧으로 변경
    
```
openssl pkcs12 -export \\

    -in ~/tls/kafka-1-creds/kafka-1.crt \\

    -inkey ~/tls/kafka-1-creds/kafka-1.key \\

    -chain \\

    -CAfile ~/tls/ca.pem \\

    -name kafka-1 \\

    -out ~/tls/kafka-1-creds/kafka-1.p12 \\

    -password pass:confluent
```
  

1. Broker keystore 생성
    
```
sudo keytool -importkeystore \\

    -deststorepass confluent \\

    -destkeystore ~/tls/kafka-1-creds/kafka.kafka-1.keystore.pkcs12 \\

    -srckeystore ~/tls/kafka-1-creds/kafka-1.p12 \\

    -deststoretype PKCS12  \\

    -srcstoretype PKCS12 \\

    -noprompt \\

    -srcstorepass confluent
```
  

Importing keystore /home/training/tls/kafka-1-creds/kafka-1.p12 to /home/training/tls/kafka-1-creds/kafka.kafka-1.keystore.pkcs12...

Entry for alias kafka-1 successfully imported.

Import command completed:  1 entries successfully imported, 0 entries failed or cancelled

  

1. kafka-1 broker keystore 확인
    
```
keytool -list -v \\

    -keystore ~/tls/kafka-1-creds/kafka.kafka-1.keystore.pkcs12 \\

    -storepass confluent
```
  

Keystore type: PKCS12

Keystore provider: SUN

  

Your keystore contains 1 entry

  

Alias name: kafka-1

Creation date: Mar 26, 2021

Entry type: PrivateKeyEntry

Certificate chain length: 2

Certificate\[1\]:

Owner: CN=kafka-1, L=MountainView, O=CONFLUENT, C=US

Issuer: CN=confluent-ca, L=MountainView, O=Confluent, C=US

Serial number: 76c397402188371ce7819063ea41f00fa3c61115

Valid from: Fri Mar 26 18:50:36 UTC 2021 until: Mon Mar 24 18:50:36 UTC 2031

Certificate fingerprints:

SHA1: 2D:AF:8A:8B:4E:0E:73:52:9E:65:9E:61:25:B0:A2:35:B1:02:A0:88

SHA256: 76:98:A3:60:57:47:56:A3:18:26:E3:93:58:49:20:B8:61:4E:08:6B:03:BD:13:20:D2:03:8E:1B:36:F6:32:72

Signature algorithm name: SHA256withRSA

Subject Public Key Algorithm: 2048-bit RSA key

Version: 3

  

Extensions:

  

#1: ObjectId: 2.16.840.1.113730.1.13 Criticality=false

0000: 16 1D 4F 70 65 6E 53 53  4C 20 47 65 6E 65 72 61  ..OpenSSL Genera

0010: 74 65 64 20 43 65 72 74  69 66 69 63 61 74 65    ted Certificate

  

  

#2: ObjectId: 2.5.29.19 Criticality=false

BasicConstraints:\[

  CA:false

  PathLen: undefined

\]

  

#3: ObjectId: 2.5.29.37 Criticality=false

ExtendedKeyUsages \[

  serverAuth

  clientAuth

\]

  

#4: ObjectId: 2.5.29.15 Criticality=true

KeyUsage \[

  DigitalSignature

  Key\_Encipherment

\]

  

#5: ObjectId: 2.5.29.17 Criticality=false

SubjectAlternativeName \[

  DNSName: kafka-1

  DNSName: kafka-1-external

\]

  

#6: ObjectId: 2.5.29.14 Criticality=false

SubjectKeyIdentifier \[

KeyIdentifier \[

0000: F5 49 61 12 B9 49 F5 AB  EC F0 16 E2 09 37 9D 81  .Ia..I.......7..

0010: BD B5 5D F7                                        ..\].

\]

\]

  

Certificate\[2\]:

Owner: CN=confluent-ca, L=MountainView, O=Confluent, C=US

Issuer: CN=confluent-ca, L=MountainView, O=Confluent, C=US

Serial number: 634db0df543037728cd7d347d77e916beccf3c78

Valid from: Fri Mar 26 18:50:13 UTC 2021 until: Sat Mar 26 18:50:13 UTC 2022

Certificate fingerprints:

SHA1: E6:07:FB:52:34:A7:F5:CA:8B:1F:AD:39:9A:69:6D:0A:C9:7E:F2:B1

SHA256: AC:09:98:27:27:97:83:61:CE:D8:56:9D:A4:3B:B2:42:EB:03:D0:ED:FF:E3:64:E8:5C:DA:3B:51:15:02:84:C8

Signature algorithm name: SHA256withRSA

Subject Public Key Algorithm: 2048-bit RSA key

Version: 3

  

Extensions:

  

#1: ObjectId: 2.5.29.35 Criticality=false

AuthorityKeyIdentifier \[

KeyIdentifier \[

0000: CC 83 15 C6 40 E2 ED 0F  2D D0 88 B9 C5 28 49 E1  ....@...-....(I.

0010: B5 A6 53 D4                                        ..S.

\]

\[CN=confluent-ca, L=MountainView, O=Confluent, C=US\]

SerialNumber: \[    634db0df 54303772 8cd7d347 d77e916b eccf3c78\]

\]

  

#2: ObjectId: 2.5.29.19 Criticality=true

BasicConstraints:\[

  CA:true

  PathLen:2147483647

\]

  

#3: ObjectId: 2.5.29.15 Criticality=true

KeyUsage \[

  Key\_CertSign

  Crl\_Sign

\]

  

#4: ObjectId: 2.5.29.14 Criticality=false

SubjectKeyIdentifier \[

KeyIdentifier \[

0000: CC 83 15 C6 40 E2 ED 0F  2D D0 88 B9 C5 28 49 E1  ....@...-....(I.

0010: B5 A6 53 D4                                        ..S.

\]

\]

  

  

  

\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

  

1. kafka credentials 저장
    

- ssl.keystore.credentials
    

sudo tee ~/tls/kafka-1-creds/kafka-1\_sslkey\_creds << EOF >/dev/null

confluent

EOF

- ssl.key.credentials
    

sudo tee ~/tls/kafka-1-creds/kafka-1\_keystore\_creds << EOF >/dev/null

confluent

EOF

  

Environment 의

KAFKA\_SSL\_KEYSTORE\_CREDENTIALS: kafka-1\_keystore\_creds

KAFKA\_SSL\_KEY\_CREDENTIALS: kafk-1\_sslkey\_creds

  

## Kafka Broker 에서 SSL 이 동작하는지 점증하기

1. Kafka Broker 시작
    
```
docker-compose -f ~/tls/docker-compose.yml up -d
```
  

1. Zookeeper / Kafka broker 가 정상적으로 동작중인지 확인
    
```
docker-compose -f ~/tls/docker-compose.yml ps
```
  

1. Kafka-1 Broker 에 SSL로 연결
    
```
openssl s\_client -connect kafka-1-external:19093 -tls1\_3 -showcerts

  

CONNECTED(00000005)

depth=1 C = US, O = Confluent, L = MountainView, CN = confluent-ca

verify error:num=19:self signed certificate in certificate chain

...

...
```
  

  

# 참고 URL

- Creating SSL Keys and Certificates [https://docs.confluent.io/platform/current/kafka/encryption.html#creating-ssl-keys-and-certificates](https://docs.confluent.io/platform/current/kafka/encryption.html#creating-ssl-keys-and-certificates)
    
- Kafka Broker SSL Configuration [https://docs.confluent.io/platform/current/kafka/encryption.html#brokers](https://docs.confluent.io/platform/current/kafka/encryption.html#brokers)