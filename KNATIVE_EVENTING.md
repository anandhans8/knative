# What is Knative Eventing?
Knative Eventing is a system that is designed to address a common need for cloud native development and provides composable primitives to enable late-binding event sources and event consumers.

### Setting up Knative Eventing Resources
Run the following command to create a namespace called dhl-event
``` sh
kubectl create namespace dhl-event
```

Add dhl-event namespace to "knative-eventing-injection" label to that namespace
``` sh
kubectl label namespace dhl-event knative-eventing-injection=enabled
```

### Validating that the Broker is running
Run the following command to verify that the Broker is in a healthy state
``` sh
kubectl --namespace dhl-event get Broker default
```

### Creating event consumers
Your event consumers receive the events sent by event producers
Here you will create two event consumers, hello-display and goodbye-display, to demonstrate how you can configure your event producers to selectively target a specific consumer.

To deploy the hello-display consumer to your cluster, run the following command:
``` sh
kubectl --namespace dhl-event apply --filename - << END
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: hello-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          # Source code: https://github.com/knative/eventing-contrib/blob/release-0.6/cmd/event_display/main.go
          image: gcr.io/knative-releases/github.com/knative/eventing-sources/cmd/event_display@sha256:37ace92b63fc516ad4c8331b6b3b2d84e4ab2d8ba898e387c0b6f68f0e3081c4

---

# Service pointing at the previous Deployment. This will be the target for event
# consumption.
  kind: Service
  apiVersion: v1
  metadata:
    name: hello-display
  spec:
    selector:
      app: hello-display
    ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
END
```

To deploy the goodbye-display consumer to your cluster, run the following command:
``` sh
kubectl --namespace dhl-event apply --filename - << END
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goodbye-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: goodbye-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          # Source code: https://github.com/knative/eventing-contrib/blob/release-0.6/cmd/event_display/main.go
          image: gcr.io/knative-releases/github.com/knative/eventing-sources/cmd/event_display@sha256:37ace92b63fc516ad4c8331b6b3b2d84e4ab2d8ba898e387c0b6f68f0e3081c4

---

# Service pointing at the previous Deployment. This will be the target for event
# consumption.
kind: Service
apiVersion: v1
metadata:
  name: goodbye-display
spec:
  selector:
    app: goodbye-display
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
END
```

Verify that your event consumers are working by running the following command:
``` sh
kubectl --namespace dhl-event get deployments hello-display goodbye-display
```

### Creating Triggers
A Trigger defines the events that you want each of your event consumers to receive. Your Broker uses triggers to forward events to the right consumers. Each trigger can specify a filter to select relevant events based on the Cloud Event context attributes.

To create the first Trigger, run the following command:
``` sh
kubectl --namespace dhl-event apply --filename - << END
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: hello-display
spec:
  filter:
    attributes:
      type: greeting
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: hello-display
END
```

To add the second Trigger, run the following command:
``` sh
kubectl --namespace dhl-event apply --filename - << END
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: goodbye-display
spec:
  filter:
    attributes:
      source: sendoff
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: goodbye-display
END
```

Verify that the triggers are working correctly by running the following command:
``` sh
kubectl --namespace dhl-event get triggers
```

### Creating event producers
You can only access the Broker from within your Eventing cluster, you must create a Pod within that cluster to act as your event producer.

To create the Pod, run the following command:
``` sh
kubectl --namespace dhl-event apply --filename - << END
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curl
  name: curl
spec:
  containers:
    # This could be any image that we can SSH into and has curl.
  - image: radial/busyboxplus:curl
    imagePullPolicy: IfNotPresent
    name: curl
    resources: {}
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
END
```

### Sending Events to the Broker
You can create an event by sending an HTTP request to the Broker. SSH into the Pod by running the following command:
``` sh 
kubectl --namespace event-example attach curl -it
```

To make the first request, which creates an event that has the type greeting, run the following in the SSH terminal
``` sh
curl -v "http://default-broker.dhl-event.svc.cluster.local" \
  -X POST \
  -H "Ce-Id: say-hello" \
  -H "Ce-Specversion: 0.3" \
  -H "Ce-Type: greeting" \
  -H "Ce-Source: not-sendoff" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Hello Knative!"}'
```

To make the second request, which creates an event that has the source sendoff, run the following in the SSH terminal:
``` sh
curl -v "http://default-broker.dhl-event.svc.cluster.local" \
  -X POST \
  -H "Ce-Id: say-goodbye" \
  -H "Ce-Specversion: 0.3" \
  -H "Ce-Type: not-greeting" \
  -H "Ce-Source: sendoff" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Goodbye Knative!"}'
```

To make the third request, which creates an event that has the type greeting and thesource sendoff, run the following in the SSH terminal:
``` sh
curl -v "http://default-broker.dhl-event.svc.cluster.local" \
  -X POST \
  -H "Ce-Id: say-hello-goodbye" \
  -H "Ce-Specversion: 0.3" \
  -H "Ce-Type: greeting" \
  -H "Ce-Source: sendoff" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Hello Knative! Goodbye Knative!"}'
```

### Verifying events were received
Look at the logs for the hello-display event consumer by running the following command:
``` sh
kubectl --namespace dhl-event logs -l app=hello-display --tail=100
```

Look at the logs for the goodbye-display event consumer by running the following command:
``` sh
kubectl --namespace dhl-event logs -l app=goodbye-display --tail=100
```
