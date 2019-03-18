# Micro on Kubernetes [![License](https://img.shields.io/:license-apache-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![GoDoc](https://godoc.org/github.com/micro/kubernetes/go/micro?status.svg)](https://godoc.org/github.com/micro/kubernetes) [![Travis CI](https://api.travis-ci.org/micro/kubernetes.svg?branch=master)](https://travis-ci.org/micro/kubernetes) [![Go Report Card](https://goreportcard.com/badge/micro/kubernetes)](https://goreportcard.com/report/github.com/micro/kubernetes)

Micro on Kubernetes is a kubernetes native micro service deployment.

## Overview

Micro is a blueprint for microservice development. Kubernetes is a container orchestration system.
Together they provide the foundations for a microservice platform. Micro on Kubernetes
provides a kubernetes native runtime to help build micro services.

## Features

- No external dependencies
- K8s native services
- Service mesh integration
- gRPC communication protocol
- Pre-initialised micro images
- Healthchecking sidecar

## Getting Started

  - [Overview](#overview)
  - [Features](#features)
  - [Getting Started](#getting-started)
  - [Installing Micro](#installing-micro)
  - [Writing a Service](#writing-a-service)
  - [Deploying a Service](#deploying-a-service)
    - [Create a Deployment](#create-a-deployment)
    - [Create a Service](#create-a-service)
  - [Writing a Web Service](#writing-a-web-service)
  - [Healthchecking](#healthchecking)
    - [Install](#install)
    - [Run](#run)
    - [K8s Deployment](#k8s-deployment)
  - [Load Balancing](#load-balancing)
    - [Usage](#usage)
    - [Deployment](#deployment)
    - [Service](#service)
  - [Using Service Mesh](#using-service-mesh)
  - [Using Config Map](#using-config-map)
    - [Example](#example)
  - [Contribute](#contribute)
    - [TODO](#todo)

## Installing Micro




```
go get github.com/micro/kubernetes/cmd/micro
```

or

```
docker pull microhq/micro:kubernetes
```

For go-micro

```
import "github.com/micro/kubernetes/go/micro"
```

## Writing a Service

Write a service as you would any other [go-micro](https://github.com/micro/go-micro) service.

```go
import (
	"github.com/micro/go-micro"
	k8s "github.com/micro/kubernetes/go/micro"
)

func main() {
	service := k8s.NewService(
		micro.Name("greeter"),
	)
	service.Init()
	service.Run()
}
```

## Deploying a Service

Here's an example k8s deployment for a micro service

### Create a Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: greeter-srv
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
            "/greeter-srv",
            "--server_address=0.0.0.0:8080",
            "--broker_address=0.0.0.0:10001"
          ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
```

Deploy with kubectl

```
kubectl create -f greeter.yaml
```

### Create a Service

```
apiVersion: v1
kind: Service
metadata:
  name: greeter
  labels:
    app: greeter
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    app: greeter
```

Deploy with kubectl

```
kubectl create -f greeter-svc.yaml
```

## Writing a Web Service

Write a web service as you would any other [go-web](https://github.com/micro/go-web) service.

```go
import (
	"net/http"

	"github.com/micro/go-web"
	k8s "github.com/micro/kubernetes/go/web"
)

func main() {
	service := k8s.NewService(
		web.Name("greeter"),
	)

	service.HandleFunc("/greeter", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`hello world`))
	})

	service.Init()
	service.Run()
}
```

## Healthchecking

### With Sidecar
 The healthchecking sidecar exposes `/health` as a http endpoint and calls the rpc endpoint `Debug.Health` on a service.
Every go-micro service has a built in Debug.Health endpoint.

#### Install
```
go get github.com/micro/kubernetes/cmd/health
```

or

```
docker pull microhq/health:kubernetes
```

#### Run

Run e.g healthcheck greeter service with address localhost:9091

```
health --server_name=greeter --server_address=localhost:9091
```

Call the healthchecker on localhost:8080

```
curl http://localhost:8080/health
```

#### K8s Deployment

Add the healthchecking sidecar to a kubernetes deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: greeter-srv
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
            "/greeter-srv",
            "--server_address=0.0.0.0:8080",
            "--broker_address=0.0.0.0:10001"
          ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
        - name: health
          command: [
            "/health",
            "--health_address=0.0.0.0:8081",
            "--server_name=greeter",
            "--server_address=0.0.0.0:8080"
          ]
          image: microhq/health:kubernetes
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 3
            periodSeconds: 3
```
### With Probe
Health Probe utility allows you to query health of go-micro services. Meant to be used for health checking micro services in [Kubernetes](https://kubernetes.io/), using the [exec probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-command).

#### Using as base image
We provide `microhq/probe:kubernetes` to use it as base image to avoid extra necessary steps 
```dockerfile
FROM microhq/probe:kubernetes
ADD greeter-srv /greeter-srv
ENTRYPOINT [ "/greeter-srv" ]
```

#### K8s Deployment
In your Kubernetes Pod specification manifest, specify a `livenessProbe` and/or `readinessProbe` for the container:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
            "/greeter-srv",
            "--server_address=0.0.0.0:8080",
            "--broker_address=0.0.0.0:10001"
          ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
          livenessProbe:
            exec:
              initialDelaySeconds: 5
              periodSeconds: 3
              command: [
                "/probe",
                "--server_name=greeter",
                "--server_address=0.0.0.0:8080"
              ]
```


## Load Balancing

Micro includes client side load balancing by default but kubernetes also provides Service load balancing strategies.
In **micro on kubernetes** we offload load balancing to k8s by using the [static selector](https://github.com/micro/go-plugins/tree/master/selector/static) and k8s services.

Rather than doing address resolution, the static selector returns the service name plus a fixed port e.g greeter returns greeter:8080.
Read about the [static selector](https://github.com/micro/go-plugins/tree/master/selector/static).

This approach handles both initial connection load balancing and health checks since Kubernetes services stop routing traffic to unhealthy services, but if you want to use long lived connections such as the ones in gRPC protocol, a service-mesh like [Conduit](https://conduit.io/), [Istio](https://istio.io) and [Linkerd](https://linkerd.io/) should be preferred to handle service discovery, routing and failure.

Note: The static selector is enabled by default.

### Usage

To manually set the static selector when running your service specify the flag or env var

Note: This is already enabled by default

```
MICRO_SELECTOR=static ./service
```

or

```
./service --selector=static
```

### Deployment

An example deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: greeter-srv
    spec:
      containers:
        - name: greeter
          command: [
            "/greeter-srv",
            "--server_address=0.0.0.0:8080",
            "--broker_address=0.0.0.0:10001"
          ]
          image: microhq/greeter-srv:kubernetes
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
```

Deploy with kubectl

```
kubectl create -f deployment-static-selector.yaml
```

### Service

The static selector offloads load balancing to k8s services. So ensure you create a k8s Service for each micro service.

Here's a sample service

```
apiVersion: v1
kind: Service
metadata:
  name: greeter
  labels:
    app: greeter
spec:
  ports:
  - port: 8080
    protocol: TCP
  selector:
    app: greeter
```

Deploy with kubectl

```
kubectl create -f service.yaml
```

Calling micro service "greeter" from your service will route to the k8s service greeter:8080.

## Using Service Mesh

Service mesh acts as a transparent L7 proxy for offloading distributed systems concerns to an external source.

See [linkerd2](https://linkerd.io/) for usage.

## Using Config Map

[Go Config](https://github.com/micro/go-config) is a simple way to manage dynamic configuration. We've provided a pre-initialised version
which reads from environment variables and the k8s config map.

It uses the `default` namespace and expects a configmap with name `micro` to be present.

### Example

Create a configmap

```
// we recommend to setup your variables from multiples files example:
$ kubectl create configmap micro --namespace default --from-file=./testdata

// verify if were set correctly with
$ kubectl get configmap micro --namespace default
{
    "apiVersion": "v1",
    "data": {
        "config": "host=0.0.0.0\nport=1337",
        "mongodb": "host=127.0.0.1\nport=27017\nuser=user\npassword=password",
        "redis": "url=redis://127.0.0.1:6379/db01"
    },
    "kind": "ConfigMap",
    "metadata": {
        ...
        "name": "micro",
        "namespace": "default",
        ...
    }
}
```

Import and use the config

```go
import "github.com/micro/kubernetes/go/config"

cfg := config.NewConfig()

// the example above "mongodb": "host=127.0.0.1\nport=27017\nuser=user\npassword=password" will be accessible as:
conf.Get("mongodb", "host") // 127.0.0.1
conf.Get("mongodb", "port") // 27017
```

## Contribute

We're looking for contributions from the community to help guide the development of Micro on Kubernetes

### TODO

- Fix k8s namespace/service name issue
- Add example multi-service application
- Add k8s CRD for micro apps


I have been using the probe for almost a year without any issue.

I believe it deserves to be the main health checking


[img]https://www.google.com/url?sa=i&source=images&cd=&cad=rja&uact=8&ved=2ahUKEwjuh7CPg4zhAhVYknAKHVXRAyoQjRx6BAgBEAU&url=https%3A%2F%2Fwww.certapet.com%2Fcat-fails-funny-cat-pics%2F&psig=AOvVaw1XcezJgyQI3so6TFe6bcxy&ust=1553009972075233[/img]