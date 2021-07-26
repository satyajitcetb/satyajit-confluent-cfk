Replicator Example:

export TUTORIAL_HOME=<path to repo>/satyajit-confluent-cfk/replicator-example-notls

Create namespace:

kubectl create ns source
kubectl create ns destination

instal cfk operator :

helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes --namespace source

helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace default --set namespaced=false

Install source and destination kafka cluster:  
  
kubectl apply -f $TUTORIAL_HOME/confluent-source-notls.yaml
kubectl apply -f $TUTORIAL_HOME/confluent-destination-notls.yaml
kubectl apply -f $TUTORIAL_HOME/controlcenter-notls.yaml
kubectl apply -f $TUTORIAL_HOME/source-topic.yaml

initiate Replicator Worker to replicate from source Cluster:

kubectl -n destination exec -it replicator-0 -- bash



# Define the configuration as a file in the pod
cat <<EOF > replicator.json
 {
 "name": "replicator2",
 "config": {
     "connector.class":
     "io.confluent.connect.replicator.ReplicatorSourceConnector",
     "tasks.max": "1",
     "topic.whitelist": "topic-in-source",
     "key.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
     "value.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
     "src.kafka.bootstrap.servers": "kafka.source.svc.cluster.local:9092",
     "src.kafka.security.protocol": "PLAINTEXT",
     "dest.kafka.bootstrap.servers": "kafka.destination.svc.cluster.local:9092",
     "dest.kafka.security.protocol": "PLAINTEXT",
     "src.consumer.group.id": "replicator",
     "src.consumer.confluent.monitoring.interceptor.sasl.mechanism": "PLAIN",
     "src.consumer.interceptor.classes": "io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor",
     "src.consumer.confluent.monitoring.interceptor.bootstrap.servers": "kafka.source.svc.cluster.local:9071",
     "src.consumer.confluent.monitoring.interceptor.security.protocol": "PLAINTEXT",
     "confluent.license": "",
     "confluent.topic.replication.factor": "3"
   }
 }
EOF

# Instantiate the Replicator Connector instance through the REST interface
curl -XPOST -H "Content-Type: application/json" --data @replicator.json http://localhost:8083/connectors -kv

# Check the status of the Replicator Connector instance
curl -XGET -H "Content-Type: application/json" http://localhost:8083/connectors -kv

Produce Data in source Cluster:
kubectl create secret generic kafka-client-config-secure \
  --from-file=$TUTORIAL_HOME/kafka.properties -n source
  
  
kubectl apply -f $TUTORIAL_HOME/secure-producer-app-data.yaml

Open Control Center
kubectl port-forward controlcenter-0 9021:9021 -n destination

______________________________________

Clean Up:
kubectl delete -f $TUTORIAL_HOME/secure-producer-app-data.yaml
kubectl delete secret  kafka-client-config-secure -n source
kubectl delete -f $TUTORIAL_HOME/source-topic.yaml
kubectl delete -f $TUTORIAL_HOME/controlcenter-notls.yaml
kubectl delete -f $TUTORIAL_HOME/confluent-destination-notls.yaml
kubectl delete -f $TUTORIAL_HOME/confluent-source-notls.yaml
helm delete confluent-operator
kubectl delete ns source
kubectl delete ns destination


