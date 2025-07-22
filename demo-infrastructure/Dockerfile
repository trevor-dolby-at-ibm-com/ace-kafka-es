FROM cp.icr.io/cp/appc/ace:13.0.4.0-r1

# See https://www.ibm.com/docs/en/app-connect/13.0.x?topic=cacerid-building-sample-supported-app-connect-enterprise-image-using-docker
# for other base image versions.

# docker build -t ace-kafka-es:13.0.4.0-r1 -f demo-infrastructure/Dockerfile .

# Build projects
ADD KafkaConsumerApplication KafkaProducerApplication /tmp/source-projects/
RUN LICENSE=accept . /opt/ibm/ace-13/server/bin/mqsiprofile && \
    ibmint deploy --input-path /tmp/source-projects --output-work-directory /home/aceuser/ace-server

# Set default policy project and pick up credentials
ADD demo-infrastructure/read-creds.sh  /home/aceuser/ace-server/read-creds.sh
ADD demo-infrastructure/server.conf.yaml /home/aceuser/ace-server/overrides

# Fix for OpenShift random userids and/or Kaniko build issues
RUN chmod -R 777 /home/aceuser/ace-server /tmp /var/mqsi 2>/dev/null || /bin/true 
