    kubectl apply -f cert-manager.yaml

    # https://github.com/rabbitmq/cluster-operator
    kubectl apply -f cluster-operator.yml
    kubectl apply -f rabbitmq.yaml

    # https://knative.dev/docs/eventing/brokers/broker-types/rabbitmq-broker/
    kubectl apply -f rabbitmq-cluster-config.yaml
    # check for  running "rabbitmq-broker-controller"
    kubectl get deployments.apps -n knative-eventing

    kubectl apply -f rabbitmq-broker-config.yaml