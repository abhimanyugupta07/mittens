---
id: running
title: How to Run
---

The simplest way to run Mittens is as a cmd application. It receives a number of command line arguments (see [flags](https://expediagroup.github.io/mittens/docs/about/getting-started#flags)).
You can also run it as a linked Docker container or even as a sidecar in Kubernetes.

## Run as a cmd application

You can run the binary executable as follows:
        
    ./mittens -target-readiness-http-path=/health -target-grpc-port=6565 -max-duration-seconds=60 -concurrency=3 -http-request=get:/hotel/potatoes -grpc-requests=service/method:"{\"foo\":\"bar\", \"bar\":\"foo\"}"

## Run as a linked Docker container

    version: "2"

    services:
    
      foo:
        image: lorem/ipsum:1.0
        ports:
          - "8080:8080"
    
      mittens:
        image: expediagroup/mittens:latest
        links:
          - app
        command: "-target-readiness-http-path=/health -target-grpc-port=6565 -max-duration-seconds=60 -concurrency=3 -http-requests=get:/hotel/potatoes -grpc-requests=service/method:{\"foo\":\"bar\", \"bar\":\"foo\"}"

_Note_: If you use Docker for Mac you might need to set the target host (`target-http-host`, `target-grpc-host`) to `docker.for.mac.localhost`, or `docker.for.mac.host.internal`, or `host.docker.internal` (depending on your version of Docker) so that your container can resolve localhost.

## Run as a sidecar on Kubernetes

```yaml
# for versions before 1.9.0 use apps/v1beta1
# for versions before 1.6.0 use extensions/v1beta1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: foo
  template:
    metadata:
      labels:
        app: foo
    spec:
      containers:
      # primary container goes here
      # - name: foo
      #   image: lorem/ipsum:1.0
      # sidecar follows
      - name: mittens
        image: mittens:latest
        resources:
          limits:
            memory: 50Mi
            cpu: 50m
          requests:
            memory: 50Mi
            cpu: 50m
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 30
        livenessProbe: 
          httpGet:
            path: /live
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 30
        args:
        - "-concurrency=3"
        - "-max-duration-seconds=60"
        - "-target-readiness-http-path=/health"
        - "-target-grpc-port=6565"
        - "-http-requests=get:/health"
        - "-http-requests=post:/hotel/aubergines:{\"foo\":\"bar\"}"
        - "-grpc-requests=service/method:{\"foo\":\"bar\",\"bar\":\"foo\"}"
```

### gRPC health checks on Kubernetes

Kubernetes does not natively support gRPC health checks.

This leaves you with a couple of options which are documented [here](https://kubernetes.io/blog/2018/10/01/health-checking-grpc-servers-on-kubernetes/).

## Note about warm-up duration

`-max-duration-seconds` includes the time needed for your application to start.
Let's say that your application takes 30 seconds to start (ie, for _/ready_ to start returning 200).
What happens is that after these initial 30 seconds, mittens will start but it will only run for 60 seconds. This is because we already spent 30 seconds waiting for the app to start.

If the application is not ready after 90 seconds, we skip the warmup routine.
