# Notes on container configuration

Starting from the repo root directory:
```
docker build -t ace-kafka-es:13.0.4.0-r1 -f demo-infrastructure/Dockerfile .
docker tag ace-kafka-es:13.0.4.0-r1 tdolby/experimental:ace-kafka-es-13.0.4.0-r1
docker push tdolby/experimental:ace-kafka-es-13.0.4.0-r1
```
Note that the tdolby/experimental images are not supported and should not be relied
upon to stay available.

## Config maps and secrets

After logging into the cluster and switching to the correct namespace, the config maps
can be created as follows:

```
kubectl create configmap scram-external --from-file=EventStreams.policyxml=SCRAMExternal/EventStreams.policyxml --from-file=policy.descriptor=SCRAMExternal/policy.descriptor
kubectl create configmap tls-internal --from-file=EventStreams.policyxml=TLSInternal/EventStreams.policyxml --from-file=policy.descriptor=TLSInternal/policy.descriptor
kubectl create configmap ibm-cloud-lite-plan --from-file=EventStreams.policyxml=IBMCloudLitePlan/EventStreams.policyxml --from-file=policy.descriptor=IBMCloudLitePlan/policy.descriptor
kubectl create configmap external-via-service --from-file=EventStreams.policyxml=ExternalViaService/EventStreams.policyxml --from-file=policy.descriptor=ExternalViaService/policy.descriptor
```

The secrets would be as follows (note that on Windows the single quotes should be removed!):
```
kubectl create secret generic truststore-file --from-file=es-cert.p12=/path/to/es-cert.p12
kubectl create secret generic truststore-pass --from-literal=name='truststorePass' --from-literal=type='truststore' --from-literal=password='TRUSTSTORE PASSWORD'
kubectl create secret generic keystore-file --from-file=user.p12=/path/to/user.p12
kubectl create secret generic keystore-pass --from-literal=name='keystorePass' --from-literal=type='keystore' --from-literal=password='KEYSTORE PASSWORD'
kubectl create secret generic scram-kafka-credentials --from-literal=name='kafka-credentials' --from-literal=type='kafka' --from-literal=username='SCRAM USERNAME' --from-literal=password='SCRAM PASSWORD'
kubectl create secret generic ibm-cloud-kafka-credentials --from-literal=name='kafka-credentials' --from-literal=type='kafka' --from-literal=username='token' --from-literal=password='CLOUD TOKEN PASSWORD'
```

Deploying the containers:
```
kubectl apply -f demo-infrastructure/ace-kafka-es-external-via-service-deployment.yaml
kubectl apply -f demo-infrastructure/ace-kafka-es-ibm-cloud-lite-plan-deployment.yaml
kubectl apply -f demo-infrastructure/ace-kafka-es-scram-external-deployment.yaml
kubectl apply -f demo-infrastructure/ace-kafka-es-tls-internal-deployment.yaml
```

Utilities:
```
kubectl delete secret truststore-file 
kubectl delete secret truststore-pass 
kubectl delete secret keystore-file 
kubectl delete secret keystore-pass 
kubectl delete secret scram-kafka-credentials 
kubectl delete secret ibm-cloud-kafka-credentials 

kubectl delete configmap scram-external
kubectl delete configmap tls-internal
kubectl delete configmap ibm-cloud-lite-plan
kubectl delete configmap external-via-service
```
