# Kubernetes config files

This repo contains application configuration files for our sudoku as a service
platform.

## Application components

### API

API component provides a RESTful HTTP API to the Sudoku data store. It is
composed of a ReplicationController and a Service. The Service has an ingress on
port 80 to the outside world and that traffic is mapped into port 8080 on the
pods controlled by the ReplicationController .

### Redis master

Redis master provides a redis server with read/write capabilities. It is
composed of a ReplicationController and a Service. The Service exposes port
6379 on the internal DNS name `redis-master`. This maps onto port 6379 on the
pods controlled by the ReplicationController.

### Generator

Generator provided a Sudoku puzzle generator to submit new puzzle candidates
into the platform through API. It consists only of a ReplicationController.
New randomly generated puzzles (a solution and a board with blanks) are POSTed
to API to be persisted in Redis master.

## Usage

### Pre-reqs: 

We are currently deploying this application on the Google Cloud infrastructure.
You should have the [Google Cloud command line
tool](https://cloud.google.com/sdk/gcloud/) setup along with the [kubectrl
component](https://cloud.google.com/container-engine/docs/kubectl/).

### New Application

The following commands will setup the application in a new cluster. They are,
respectively, deploy the Redis Master replication controller, Redis Master
service, API replication controller, API service, and finally, Generator
replication controller.

```
$ kubectl create -f redis-master-controller.yaml
$ kubectl create -f redis-master-service.yaml
$ kubectl create -f api-controller.yaml
$ kubectl create -f api-service.yaml
$ kubectl create -f generator-controller.yaml
```

### Updating Generator

The easiest way to replace Generator with a new image version or config is to
update the config file, delete the existing replication controller, and create a
new one.

Update the replication controller configs like so:

```diff
  apiVersion: v1
  kind: ReplicationController
  metadata:
    name: generator
    labels:
      app: generator
  spec:
    replicas: 1
    selector:
      app: generator
    template:
      metadata:
        labels:
          app: generator
      spec:
        containers:
        - name: generator
-         image: gcr.io/total-sudoku/sudoku-generator:v4
+         image: gcr.io/total-sudoku/sudoku-generator:v5
          env:
          - name: 'API_ROOT'
            value: 'http://api'
```

Delete the existing replication controller:

```
$ kubectl delete rc generator
```

And create the new one from the updated config file:

```
$ kubectl create -f generator-controller.yaml
```

### Updating API replication controller

We use a rolling update to prevent downtime during a deployment of API. We will
need to update the API replication controller configuration and then issue the
rolling update command. Kubernetes will take care of spinning up the new pod and
then spinning down the old one when the new one is settled.

Update the repication controller configs like so:

```diff
  apiVersion: v1
  kind: ReplicationController
  metadata:
-   name: api-v5
+   name: api-v6
    labels:
-     app: api-v5
+     app: api-v6
  spec:
    replicas: 1
    selector:
-     app: api-v5
+     app: api-v6
    template:
      metadata:
        labels:
-         app: api-v5
+         app: api-v5
          role: api-frontend
      spec:
        containers:
-       - name: api-v5
+       - name: api-v6
-         image: gcr.io/total-sudoku/sudoku-api:v5
+         image: gcr.io/total-sudoku/sudoku-api:v6
          ports:
          - name: http-server
            containerPort: 8080
          env:
          - name: 'REDIS_ADDR'
            value: 'redis-master:6379'
```

Issue the rolling update command:

```
$ kubectl rolling-update api-v5 -f api-controller.yaml
```
