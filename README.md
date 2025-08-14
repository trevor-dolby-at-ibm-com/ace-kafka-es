# ace-kafka-es

This repo contains examples of configuring ACE and Kafka with IBM Event Streams in the Cloud
Pak for Integration (CP4i) and also using the free tier of IBM Cloud. The main focus is on how 
communication can be secured, and how ACE policies and credentials should be configured to 
enable secure connectivity.

See also Dale Lane's blog post at https://dalelane.co.uk/blog/?p=4573 for a detailed walkthrough
of connecting ACE to Event Streams. This repo uses the same `THIS.IS.MY.TOPIC` topic for the
flows, and the "Pre-requisites" in the blog post apply to this repo also. The flows themselves 
are a modified form of the "Using the KafkaProducer and KafkaConsumer nodes with a Kafka topic" 
tutorial in the ACE toolkit.

## Overview

The CP4i examples use an Event Streams instance based on the "Development SCRAM" sample YAML 
(see "YAML view"->"Samples" in the OpenShift UI) and many other configurations are possible.
In particular, the way the sample configures SCRAM and TLS listeners is not the only way: 
TLS can be used for external authentication, SCRAM for internal, etc. The principles in this 
repo (how components can trust each other, ACE configuration possiblities, etc) should apply 
in general, as should the examples with appropriate modifications.

For CP4i, several options exist depending on the location of the ACE runtime. This diagram is
a simplified view (Kafka has more than one network listener while only one is shown, etc) but the
basic ideas are as follows:

![ace-es-light](demo-infrastructure/images/ace-and-es-cp4i.png#gh-light-mode-only)![ace-es-dark](demo-infrastructure/images/ace-and-es-cp4i-dark.png#gh-dark-mode-only)
The primary options are

1. ACE outside the cluster connecting to the external listener for Event Streams, using the OpenShift ingress Routes. This is the [SCRAM External](#scram-external) section below.
2. ACE inside the cluster connecting to the internal listener for Event Streams, using the OpenShift internal services. This is the [TLS Internal](#tls-internal) section below.

The other options have drawbacks and may not behave as desired:

- Option 3 above shows an internal ACE attempting to use the external service without going through
  the ingress Routes, but this approach actually falls back to using the ingress in reality despite
  the apparent configuration and is generally less secure due to certificate hostname issues. See 
  below for further details, and option 2 is preferred over this one.
- Attempting to port-forward from the ACE system into the cluster will not work as expected due to
  the same issues presented by option 3 and also because of the way Kafka works: the client connects
  to the bootstrap server and is then told to connect to other Kafka servers, and those later connections
  will be made to hostnames that will in general not be forwarded. It is possible to use an [Event Gateway](#event-gateway) to work around this, but otherwise option 1 is preferred over port forwarding.

For the Lite plan on IBM Cloud, the (simplified) picture is as follows:

![ace-es-cloud-light](demo-infrastructure/images/ace-and-es-cloud.png#gh-light-mode-only)![ace-es-cloud-dark](demo-infrastructure/images/ace-and-es-cloud-dark.png#gh-dark-mode-only)
This resembles option 1 in the CP4i picture but uses different security protocols, and is described
further in the [IBM Cloud Lite Plan](#ibm-cloud-lite-plan) section below.

## Security options

Various authentication options exist for Event Streams (authorization is a different subject), but at
a high level they revolve around two questions:

- How can the client trust that the server is what it claims to be?
- How can the server be sure that the client is allowed to connect and perform work?

Different connect options in the diagrams above have different answers but all address the questions:

|Option|Why can the server be trusted?|Why can the client be trusted?|Client artifacts needed|
|---|---|---|---|
|[SCRAM External](#scram-external)|The server presents a TLS key for the correct hostname issued by a CA in the truststore provided by an ES administrator|The client sends SCRAM credentials issued by Event Streams|es-cert.p12 and SCRAM credentials|
|[TLS Internal](#tls-internal)|The server presents a TLS key for the correct hostname issued by a CA in the truststore provided by an ES administrator|The client presents a TLS key (mTLS) issued by Event Streams or provided by an ES administrator|es-cert.p12 and user.p12|
|[IBM Cloud Lite Plan](#ibm-cloud-lite-plan)|The server presents a TLS key for the correct hostname issued by a globally-trusted CA (Let's Encrypt at the time of writing)|The client sends a cloud token provided by Event Streams|Cloud token|

The other options mentioned above (option 3 and port forwarding) answer the "Why can the server be trusted?"
question with various forms of "it can't be trusted" because it will present a TLS key with the wrong hostname.
Although some of the options can be made to work at a technical level by switching off hostname validation, this
is not a secure way to communicate. 

## How to tell if the flows are working

The expected server output is of the form
```
2025-07-23 13:19:31.908970: BIP1990I: Integration server 'ace-kafka-es-work-dir' starting initialization; version '13.0.4.0' (64-bit)
2025-07-23 13:19:31.943840: BIP9905I: Initializing resource managers.
2025-07-23 13:19:31.946120: BIP9985I: Using Java version 17. The following integration server components are unavailable with this Java version: FlowSecurityProviders/TFIM, GlobalCacheBackends/WXS, JavaNodes/CORBA, JavaNodes/WS-Security, JavaNodes/WSRR.
2025-07-23 13:19:34.574856: BIP10112I: The resources from 'imbopentelemetry.lil' have not been loaded because the runtime component 'OpenTelemetry' has not been enabled. Reason: 'Integration Server Configuration'. Further detail: 'server.conf.yaml'.
2025-07-23 13:19:37.975620: BIP9906I: Reading deployed resources.
2025-07-23 13:19:37.986376: BIP9907I: Initializing deployed resources.
2025-07-23 13:19:37.990436: BIP2155I: About to 'Initialize' the deployed resource 'KafkaConsumerApplication' of type 'Application'.
2025-07-23 13:19:37.998052: BIP2155I: About to 'Initialize' the deployed resource 'KafkaProducerApplication' of type 'Application'.
2025-07-23 13:19:38.305652: BIP9332I: PolicyProject 'IBMCloudLitePlan' has been reloaded successfully.
2025-07-23 13:19:38.308508: BIP2155I: About to 'Start' the deployed resource 'KafkaConsumerApplication' of type 'Application'.
2025-07-23 13:19:44.529176: BIP2269I: Deployed resource 'KafkaConsumerFlow' (uuid='KafkaConsumerFlow',type='MessageFlow') started successfully.
2025-07-23 13:19:44.533240: BIP9332I: Application 'KafkaConsumerApplication' has been reloaded successfully.
2025-07-23 13:19:44.535664: BIP2155I: About to 'Start' the deployed resource 'KafkaProducerApplication' of type 'Application'.
2025-07-23 13:19:44.661348: BIP3132I: The HTTP Listener has started listening on port '7800' for 'http' connections.
2025-07-23 13:19:44.664644: BIP1996I: Listening on HTTP URL '/KafkaTutorial/httpin'.
Started native listener for HTTP input node on port 7800 for URL /KafkaTutorial/httpin
2025-07-23 13:19:44.669128: BIP2269I: Deployed resource 'KafkaProducerFlow' (uuid='KafkaProducerFlow',type='MessageFlow') started successfully.
2025-07-23 13:19:44.673100: BIP9332I: Application 'KafkaProducerApplication' has been reloaded successfully.
2025-07-23 13:19:45.122392: BIP2866I: IBM App Connect Enterprise administration security is inactive.
2025-07-23 13:19:45.147192: BIP3132I: The HTTP Listener has started listening on port '7600' for 'RestAdmin https' connections.
2025-07-23 13:19:45.153280: BIP1991I: Integration server has finished initialization.
```
with no errors visible. The KafkaConsumerFlow will attempt to subscribe to the topic on startup
so any errors in connection parameters should appear without any need for user action.

To publish a message, curl can be used as follows:
```
curl -X POST --data  {"test":true} http://localhost:7800/KafkaTutorial/httpin
```

## SCRAM External

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

The [SCRAMExternal/EventStreams.policyxml](/SCRAMExternal/EventStreams.policyxml) file shows the 
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

## TLS Internal

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
[TLSInternal/EventStreams.policyxml](/TLSInternal/EventStreams.policyxml) file shows the 
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

## Event Gateway

As noted above, a normal Event Gateway will advertise external route addresses because those are
what is set in the KAFKA_ADVERTISED_LISTENER parameter (usually including many addresses of the form 
`rt-10-tools.apps.688037023bdaabb2db30f3a0.ocp.techzone.ibm.com:443`) but it is possible to funnel 
all connections across a single port for bootstrap as well as eventing traffic:
![ace-and-es-gateway-light](demo-infrastructure/images/ace-and-es-gateway.png#gh-light-mode-only)![ace-and-es-gateway-dark](demo-infrastructure/images/ace-and-es-gateway-dark.png#gh-dark-mode-only)

This approach is a combination of the standard Docker approach described at https://ibm.github.io/event-automation/eem/installing/install-docker-egw/ 
and the "Kubernetes Deployment" option in the Event Endpoint Management UI "Add Gateway" wizard. The
initial steps follow the Docker pattern (creating certificates, etc) but the gateway itself should
be deployed as a Kubernetes deployment with the settings modified as follows
```
            - name: GATEWAY_PORT
              value: '9092'
            - name: KAFKA_ADVERTISED_LISTENER
              value: 'localhost:9092,localhost:9093,localhost:9094'
```
to cause the gateway to advertise localhost addresses to Kafka clients connecting to the bootstrap
server. A Kubernetes service is needed to allow the gateway to be found by the port forwarding 
(see [example](/demo-infrastructure/port-forward-gw-group-pf-1-svc.yaml)) without needing to use a
(changing) pod name.

Once this gateway is running and topics can be published to it, then the port forwarding can be run
locally with the following commands (adjust as needed for namespaces and service names):
```
oc --namespace integration port-forward --address 0.0.0.0 svc/port-forward-gw-group-pf-1-svc 9092:9092
oc --namespace integration port-forward --address 0.0.0.0 svc/port-forward-gw-group-pf-1-svc 9093:9092
oc --namespace integration port-forward --address 0.0.0.0 svc/port-forward-gw-group-pf-1-svc 9094:9092
```

ACE connection policies are similar to the [SCRAM External](#scram-external) scenario but in this case
the connections are made to the local end of the port forwarding tunnel. As can be seen in the
[EventGatewayPortForward/EventStreams.policyxml](/EventGatewayPortForward/EventStreams.policyxml)
file, the key elements of the policy that tie the solution together are as follows:
```
    <bootstrapServers>localhost:9092</bootstrapServers>
    <securityProtocol>SASL_SSL</securityProtocol>
    <saslMechanism>PLAIN</saslMechanism>
    <sslProtocol>TLSv1.3</sslProtocol>
    <securityIdentity>kafka-credentials</securityIdentity>
    <saslConfig>org.apache.kafka.common.security.plain.PlainLoginModule required;</saslConfig>
    <sslTruststoreLocation>c:\tmp\pf-egw-cert.p12</sslTruststoreLocation>
    <sslTruststoreType>PKCS12</sslTruststoreType>
    <sslTruststoreSecurityIdentity>changeit</sslTruststoreSecurityIdentity>
    <sslEnableCertificateHostnameChecking>true</sslEnableCertificateHostnameChecking>
```
The authentication in this case is using SASL_SSL PLAIN (similar to the [IBM Cloud Lite Plan](#ibm-cloud-lite-plan))
and the credentials come from the Event Endpoint Management UI's catalog of topics, where the "Subscribe"
button for a particular topic and gateway option ("THIS.IS.MY.TOPIC.ALIAS" and "port forwading", for example)
provides credentials and connection information. The truststore is created from the CA and TLS certifcates
(but not the private key) created during the creation of the Event Gateway using commands such as 
```
keytool -importcert -keystore c:\tmp\pf-egw-cert.p12 -storepass changeit -storetype PKCS12 -file "c:\Users\temp\Downloads\port-forward-cert-chain.pem"
```
The equivalent table entry for this case is
|Option|Why can the server be trusted?|Why can the client be trusted?|Client artifacts needed|
|---|---|---|---|
|[Event Gateway](#event-gateway)|The gateway presents a TLS key for the correct hostname (`localhost`, in this case) issued by a CA in the truststore provided by the gateway administrator|The client sends SCRAM credentials issued by Event Endpoint Management's catalog|pf-egw-cert.p12 and SCRAM credentials|
