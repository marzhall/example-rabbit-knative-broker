# Example Rabbit + Knative broker setup

This repo intends to have a minimal set of steps to run in order to create a
working example service that revieves and processes events from knative using
a broker that's provided by rabbitmq.

It uses parts of both the [knative documentation](git@github.com:knative/docs.git)
and the [rabbitmq documentation](https://github.com/rabbitmq/cluster-operator).

A set of steps is layed out that combines the examples to effect a service
that listens for knative messages, and a broker used by knative that is
based on a rabbitmq instance created as part of the cluster.

## Installation

These steps are intended to be as minimal as possible. If you have any trouble
with them, please create an issue, or ping me directly.

### Presumptions:

It's assumed that you already have a kubernetes cluster that has knative
installed. If you do not, and you're doing local development, I'd recommend
looking at the `kn` [tool](https://github.com/knative-sandbox/kn-plugin-quickstart),
which will allow you to create a cluster with either kind or minikube, and will
automatically install knative for you.

### Rabbit Infrastructure:

We'll be installing a certificate manager to handle authentication between rabbitmq
and brokers, then a rabbitmq kubernetes operator, and finally checking to see
if everything is installed correctly. 

The instructions and their yaml files come from the linked repositories.

    # https://github.com/rabbitmq/cluster-operator
    kubectl apply -f rabbit/cluster-operator.yml
    kubectl apply -f rabbit/rabbitmq.yaml

    # https://knative.dev/docs/eventing/brokers/broker-types/rabbitmq-broker/
    kubectl apply -f rabbit/rabbitmq-cluster-config.yaml
    # check for  running "rabbitmq-broker-controller"
    kubectl get deployments.apps -n knative-eventing

### Basic eventing service using the rabbitmq infrastructure

This is the application that we'll use to handle events on a broker via
knative. It will create a broker, and a configuration for that broker
that will direct it to use the rabbitmq instance we've created.

#### Build our event-handling container and load it into our cluster

    cd helloworld-go
    docker build -t dev.local/kevents:latest .
    # The following code uploads the container into a kind cluster.
    # Those using minikube will have an equivalent command, but I'm
    # unsure of what it is.
    kind load docker-image dev.local/kevents:latest --name knative --namespace knative-samples

#### Spin up our rabbitmq and event-handling k8s infrastructure

    # assuming you're still inside the 'helloworld-go' directory 
    kubectl apply -f sample-app.yaml trigger.yaml

#### Test that event handling occurring

    # Get our broker URL
    kubectl get broker --namespace knative-samples | grep -v NAME | awk '{ printf $2 }'
    
    # Run a curl instance we can use to send a message to the broker
    kubectl -n knative-samples run curl --image=radial/busyboxplus:curl -it

    # Inside the container we've started, send a message to our rabbit broker
    # using curl, and using the broker URL we got in the first command
    curl -v "<THE URL FROM THE FIRST COMMAND>" \
    -X POST \
    -H "Ce-Id: 536808d3-88be-4077-9d7a-a3f162705f79" \
    -H "Ce-Specversion: 1.0" \
    -H "Ce-Type: dev.knative.samples.helloworld" \
    -H "Ce-Source: dev.knative.samples/helloworldsource" \
    -H "Content-Type: application/json" \
    -d '{"msg":"Hello World from the curl pod."}'
    
    # Leave the container
    exit

    # Print the logs of the event-handline container to see if it received an event:
    kubectl --namespace knative-samples logs -l app=helloworld-go --tail=50

    # You should see something similar to:
    Event received.
    Validation: valid
    Context Attributes,
     specversion: 1.0
     type: dev.knative.samples.helloworld
     source: dev.knative.samples/helloworldsource
     id: 536808d3-88be-4077-9d7a-a3f162705f79
     time: 2019-10-04T22:35:26.05871736Z
     datacontenttype: application/json
    Extensions,
     knativearrivaltime: 2019-10-04T22:35:26Z
     knativehistory: default-kn2-trigger-kn-channel.knative-samples.svc.cluster.local
     traceparent: 00-971d4644229653483d38c46e92a959c7-92c66312e4bb39be-00
    Data,
      {"msg":"Hello World from the curl pod."}

    Hello World Message "Hello World from the curl pod."
    Responded with event
    Validation: valid
    Context Attributes,
      specversion: 1.0
      type: dev.knative.samples.hifromknative
      source: knative/eventing/samples/hello-world
      id: 37458d77-01f5-411e-a243-a459bbf79682
      datacontenttype: application/json
    Data,
      {"msg":"Hi from Knative!"}

And that should be it! From there, you should be able to create services that hit the
endpoint at the URL you hit with CURL, and use the yaml and hellow-world container
as a base for services that handle events. 