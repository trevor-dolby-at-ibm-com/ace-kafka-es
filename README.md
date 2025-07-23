# ace-kafka-es

This repo contains examples of configuring ACE and Kafka with IBM Event Streams in the Cloud
Pak for Integration (CP4i) and also using the free tier of IBM Cloud. The main focus is on how 
communication can be secured, and how ACE policies and credentials should be configured to 
enable secure connectivity.

See also Dale Lane's blog post at https://dalelane.co.uk/blog/?p=4573 for a detailed walkthrough
of connecting ACE to Event Streams.

## Overview

The CP4i examples use an Event Streams instance based on the "Development SCRAM" sample YAML 
(see "YAML view"->"Samples" in the OpenShift UI) and many other configurations are possible. 
The principles in this repo (how components can trust each other, ACE configuration possiblities, etc)
should apply in general with appropriate modifications.

For CP4i, several options exist depending on the location of the ACE runtime. This diagram is
a simplified view (Kafka has more than one network listener while only one is shown, etc) but the
basic ideas are as follows:

![ace-es-light](demo-infrastructure/images/ace-and-es-cp4i.png#gh-light-mode-only)![ace-es-dark](demo-infrastructure/images/ace-and-es-cp4i-dark.png#gh-dark-mode-only)
The primary options are

1. ACE outside the cluster connecting to the external listener for Event Streams, using the OpenShift ingress Routes.
2. ACE inside the cluster connecting to the internal listener for Event Streams, using the OpenShift internal services.

The other options have drawbacks and may not behave as desired:

- Option 3 above shows an internal ACE attempting to use the external service without going through
  the ingress Routes, but this approach actually falls back to using the ingress in reality despite
  the apparent configuration and is generally less secure due to certificate hostname issues. See 
  below for further details, and option 2 is preferred over this one.
- Attempting to port-forward from the ACE system into the cluster will not work as expected due to
  the same issues presented by option 3 and also because of the way Kafka works: the client connects
  to the bootstrap server and is then told to connect to other Kafka servers, and those later connections
  will be made to hostnames that will in general not be forwarded. Option 1 is preferred over port
  forwarding.

For the Lite plan on IBM Cloud, the (simplified) picture is as follows:

![ace-es-cloud-light](demo-infrastructure/images/ace-and-es-cloud.png#gh-light-mode-only)![ace-es-cloud-dark](demo-infrastructure/images/ace-and-es-cloud-dark.png#gh-dark-mode-only)
This resembles option 1 in the CP4i picture but uses different security protocols.

## Security options

Various authentication options exist for Event Streams (authorization is a different subject), but at
a high level they revolve around two questions:

- How can the client trust that the server is what it claims to be?
- How can the server be sure that the client is allowed to connect and perform work?

Different connect options in the diagrams above have different answers but all address the questions:

|Option|Why can the server be trusted?|Why can the client be trusted?|Client artifacts needed|
|---|---|---|---|
|[External CP4i](#external-cp4i)|The server presents a TLS key for the correct hostname issued by a CA in the truststore provided by an ES administrator|The client sends SCRAM credentials issued by Event Streams|es-cert.p12 and SCRAM credentials|
|[Internal CP4i](#internal-cp4i)|The server presents a TLS key for the correct hostname issued by a CA in the truststore provided by an ES administrator|The client presents a TLS key (mTLS) issued by Event Streams or provided by an ES administrator|es-cert.p12 and user.p12|
|[IBM Cloud Lite Plan](#ibm-cloud-lite-plan)|The server presents a TLS key for the correct hostname issued by a globally-trusted CA (Let's Encrypt at the time of writing)|The client sends a cloud token provided by Event Streams|Cloud token|

The other options mentioned above (option 3 and port forwarding) answer the "Why can the server be trusted?"
question with various forms of "it can't be trusted" because it will present a TLS key with the wrong hostname.
Although some of the options can be made to work at a technical level by switching off hostname validation, this
is not a secure way to communicate. 

## External CP4i

A truststore is needed to ensure the client can trust the server, and SCRAM credentials are needed to 
ensure the server can trust the client. A list of bootstrap servers is also needed, and that can be
found in the Event Streams UI in the "External" section.

The truststore can be downloaded from the ES UI:

![download-truststore](/demo-infrastructure/images/download-truststore.png)

where the truststore itself should be stored in an accessible location (such as `/tmp/es-cert.p12`)
while the password should be given to ACE with a command such as
```
mqsisetdbparms -w /tmp/ace-kafka-es-work-dir -n truststore::truststorePass -u dummy -p RaNdOmLeTtErS
```
The SCRAM credentials are generated from the same UI:

![scram-credentials](/demo-infrastructure/images/scram-credentials.png)

and are given to ACE with a command such as
```
mqsisetdbparms -w /tmp/ace-kafka-es-work-dir -n kafka::kafka-credentials -u ace-kafka-es -p longPasswordString
```

The [StandardExternal/EventStreams.policyxml](/StandardExternal/EventStreams.policyxml) file shows the 
key elements of the policy that tie the solution together:
```
    <bootstrapServers>dev-scram-kafka-bootstrap-integration.apps.687f8d0dea2fca3eff46b2e1.am1.techzone.ibm.com:443</bootstrapServers>
    <securityProtocol>SASL_SSL</securityProtocol>
    <saslMechanism>SCRAM-SHA-512</saslMechanism>
    <sslProtocol>TLSv1.3</sslProtocol>
    <securityIdentity>kafka-credentials</securityIdentity>
    <saslConfig>org.apache.kafka.common.security.scram.ScramLoginModule required;</saslConfig>
    <sslTruststoreLocation>/tmp/es-cert.p12</sslTruststoreLocation>
    <sslTruststoreType>PKCS12</sslTruststoreType>
    <sslTruststoreSecurityIdentity>truststorePass</sslTruststoreSecurityIdentity>
    <sslEnableCertificateHostnameChecking>true</sslEnableCertificateHostnameChecking>
```
Note the `securityProtocol`, `saslMechanism`, and `saslConfig` lines that are needed to enable the
connection; these values can be found in the "Sample configuration properties" section of the 
"Connect to this cluster" wizard and also in [Dale Lane's blog post](https://dalelane.co.uk/blog/?p=4573)

## Internal CP4i

A truststore is needed to ensure the client can trust the server, and a client keystore is needed
to ensure the server can trust the client. A list of bootstrap servers is also needed, and that can be
found in the Event Streams UI in the "Internal" section.

The PKCS12 truststore can be downloaded from the ES UI (same as the External case above):

![download-truststore](/demo-infrastructure/images/download-truststore.png)

where the truststore itself should be stored in an accessible location (such as `/tmp/es-cert.p12`)
while the password should be given to ACE with a command such as
```
mqsisetdbparms -w /tmp/ace-kafka-es-work-dir -n truststore::truststorePass -u dummy -p RaNdOmLeTtErS
```
The client keystore is generated from the same UI:

![download-keystore](/demo-infrastructure/images/download-keystore.png)

and is a ZIP file that contains the TLS key, a TLS keystore, and the keystore password:

```
tdolby@IBM-PF3K066L:/tmp/unzip$ unzip -l ace-kafka-es-internal.zip
Archive:  ace-kafka-es-internal.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
     1529  2025-07-22 15:31   user.crt
     1704  2025-07-22 15:31   user.key
     2974  2025-07-22 15:31   user.p12
       32  2025-07-22 15:31   user.password
---------                     -------
     6239                     4 files
```
The `user.p12` file should be stored in an accessible location (such as `/tmp/user.p12`) while
the password in the `user.password` file should be given to ACE with a command such as
```
mqsisetdbparms -w /tmp/ace-kafka-es-work-dir -n keystore::keystorePass -u dummy -p RaNdOmLeTtErS
```
The other two files are ignored for this use case, and the
[StandardInternal/EventStreams.policyxml](/StandardInternal/EventStreams.policyxml) file shows the 
key elements of the policy that tie the solution together:
```
    <bootstrapServers>dev-scram-kafka-bootstrap.integration.svc:9093</bootstrapServers>
    <securityProtocol>SSL</securityProtocol>
    <sslProtocol>TLSv1.3</sslProtocol>
    <sslKeystoreLocation>/tmp/user.p12</sslKeystoreLocation>
    <sslKeystoreType>PKCS12</sslKeystoreType>
    <sslKeystoreSecurityIdentity>keystorePass</sslKeystoreSecurityIdentity>
    <sslTruststoreLocation>/tmp/es-cert.p12</sslTruststoreLocation>
    <sslTruststoreType>PKCS12</sslTruststoreType>
    <sslTruststoreSecurityIdentity>truststorePass</sslTruststoreSecurityIdentity>
    <sslEnableCertificateHostnameChecking>true</sslEnableCertificateHostnameChecking>
```
Note the absence of a securityIdentity: the client keystore takes on the role of providing 
credentials, with the username being the CN of the certificate:
```
tdolby@IBM-PF3K066L:/tmp/unzip$ cat user.crt  | openssl x509 -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            28:76:ff:93:dc:09:15:21:4c:dd:5a:0d:b8:ae:75:b9:2a:40:b3:67
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: O = io.strimzi, CN = clients-ca v0
        Validity
            Not Before: Jul 22 15:30:54 2025 GMT
            Not After : Oct 20 15:30:54 2025 GMT
        Subject: CN = ace-kafka-es-internal
```

## IBM Cloud Lite Plan

No truststores are needed in this case, and the credentials can be created using the
"Service credentials" tab in the Event Streams cloud UI. Once the credentials have been created,
the user and password fields can be given to ACE with a command such as
```
mqsisetdbparms -w /tmp/ace-kafka-es-work-dir -n kafka::kafka-credentials -u token -p longTokenString
```
The list of bootstrap servers is also provided in the service credentials JSON.

The [IBMCloudLitePlan/EventStreams.policyxml](/IBMCloudLitePlan/EventStreams.policyxml) file shows the 
key elements of the policy that tie the solution together:
```
    <bootstrapServers>broker-3-m81pgqtr89mv4vz2.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093,broker-2-m81pgqtr89mv4vz2.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093,broker-4-m81pgqtr89mv4vz2.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093,broker-0-m81pgqtr89mv4vz2.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093,broker-5-m81pgqtr89mv4vz2.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093,broker-1-m81pgqtr89mv4vz2.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093</bootstrapServers>
    <securityProtocol>SASL_SSL</securityProtocol>
    <saslMechanism>PLAIN</saslMechanism>
    <sslProtocol>TLSv1.3</sslProtocol>
    <securityIdentity>kafka-credentials</securityIdentity>
    <saslConfig>org.apache.kafka.common.security.plain.PlainLoginModule required;</saslConfig>
    <sslEnableCertificateHostnameChecking>true</sslEnableCertificateHostnameChecking>
```
Note the absence of any keystore and truststore configuration plus the `securityProtocol`, 
`saslMechanism`, and `saslConfig` lines that are needed to enable the connection; these values can
be found in the "Sample configuration properties" section of the "Connect to this service" wizard.

## Certificate notes

Web broswers and the openssl command can be used to examine the certificates presented by the 
bootstrap servers. For the CP4i "Development SCRAM" cluster, the certificates are issued by 
Strimzi using a self-signed CA certificate (hence the need for a truststore) and look as follows:
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            53:16:74:80:ef:88:b0:b0:e6:dd:51:55:83:ed:e3:bd:3e:1f:a0:a5
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: O = io.strimzi, CN = cluster-ca v0
        Validity
            Not Before: Jul 22 14:52:03 2025 GMT
            Not After : Oct 20 14:52:03 2025 GMT
        Subject: O = io.strimzi, CN = dev-scram-kafka
...
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:dev-scram-kafka-1.dev-scram-kafka-brokers.integration.svc.cluster.local,
                DNS:dev-scram-kafka-bootstrap-integration.apps.687f8d0dea2fca3eff46b2e1.am1.techzone.ibm.com,
                DNS:dev-scram-kafka-brokers.integration.svc.cluster.local,
                DNS:dev-scram-kafka-bootstrap.integration.svc,
                DNS:dev-scram-kafka-bootstrap,
                DNS:dev-scram-kafka-brokers,
                DNS:dev-scram-kafka-brokers.integration,
                DNS:dev-scram-kafka-1-integration.apps.687f8d0dea2fca3eff46b2e1.am1.techzone.ibm.com,
                DNS:dev-scram-kafka-brokers.integration.svc,
                DNS:dev-scram-kafka-1.dev-scram-kafka-brokers.integration.svc,
                DNS:dev-scram-kafka-bootstrap.integration.svc.cluster.local,
                DNS:dev-scram-kafka-bootstrap.integration
```
with various internal and external hostnames listed in the "Subject Alternative Name" section.

The equivalent for the cloud service is issued by Let's Encrypt (trusted by default):
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            06:e6:23:9d:76:2b:29:d8:81:7e:a3:7c:f0:01:47:b3:f1:b9
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = Let's Encrypt, CN = R11
        Validity
            Not Before: May 28 08:18:00 2025 GMT
            Not After : Aug 26 08:17:59 2025 GMT
        Subject: CN = eventstreams.cloud.ibm.com
...
        X509v3 extensions:
...
            X509v3 Subject Alternative Name:
                DNS:*.kafka.svc09.us-south.eventstreams.cloud.ibm.com,
                DNS:*.svc09.us-south.eventstreams.cloud.ibm.com,
                DNS:eventstreams.cloud.ibm.com,
                DNS:svc09.us-south.eventstreams.cloud.ibm.com
```
and the wildcard hostnames in the "Subject Alternative Name" section allow the multi-tenant
service to support many different customers with the same server certificate.