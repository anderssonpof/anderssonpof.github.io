---
layout: post
title: "Run a traefik sidecar in gcp cloud run"
date: 2024-03-28
categories: GCP GOOGLE CLOUD_RUN TRAEFIK
---

# Run a traefik sidecar in GCP cloud run

I google cloud you can run multi containers and one way to utilize this is to run a proxy in front of your web container. You could run traefik, traefik, envoy or any other product of your choice. \
Google has an example running using [traefik](https://cloud.google.com/run/docs/internet-proxy-traefik-sidecar) which I used as a basis for this.

I'm going to mount both the static and dynamic configuration files for traefik referencing secrets in Secret Manager.

First a simple static file

```yaml
api:
  dashboard: false
entryPoints:
  web:
    address: ":8080"
providers:
  # Enable the file provider to define routers / middlewares / services in file
  file:
    filename: /etc/traefik_dynamic/dynamic.yaml
    watch: true
```

Then a very basic dynamic file

```yaml
http:
  routers:
    my-router:
      rule: "PathPrefix(`/`)"
      entryPoints:
        - web
      service: cr-hello

  services:
    cr-hello:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8888" # The backend service
```

I'm going to use Google hello container as the backend for this example.

Firstly since this is a preview feature we need to enable this feature

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
# [START cloudrun_mc_hello_sidecar_step_metadata]
metadata:
  name: "traefik-multicontainer"
  labels:
    cloud.googleapis.com/location: "REGION"
  annotations:
    # Required to use Cloud Run multi-containers (preview feature)
    run.googleapis.com/launch-stage: BETA
    run.googleapis.com/description: sample tutorial service
    # Externally available
    run.googleapis.com/ingress: all
```

Then we need to define our containers that we are going to run \
Here we define the traefik container and the associated mounts and port 8080. We also define the backend hello container running on the port 8888. \
To my knowledge mounting multiple files in the same directory is not supported in cloud run so we need two mount paths.

```yaml
spec:
  template:
    metadata:
      annotations:
        # Defines container startup order within multi-container service.
        # Below requires hello container to spin up before traefik container,
        # which depends on the hello container.
        # https://cloud.google.com/run/docs/configuring/containers#container-ordering
        run.googleapis.com/container-dependencies: "{traefik: [hello]}"
    # [END cloudrun_mc_hello_sidecar_step_deps]
    # [START cloudrun_mc_hello_sidecar_step_serving]
    spec:
      containers:
        # A) Serving ingress container "traefik" listening at PORT 8080
        # Main entrypoint of multi-container service.
        # Any pings to this container will proxy over to hello container at PORT 8888.
        # https://cloud.google.com/run/docs/container-contract#port
        - image: traefik
          name: traefik
          ports:
            - name: http1
              containerPort: 8080
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
          # Referencing declared volume below,
          volumeMounts:
            - name: traefik-conf-static
              readOnly: true
              mountPath: /etc/traefik/
            - name: traefik-conf-dynamic
              readOnly: true
              mountPath: /etc/traefik_dynamic/
          startupProbe:
            timeoutSeconds: 240
            periodSeconds: 240
            failureThreshold: 1
            tcpSocket:
              port: 8080
        # [END cloudrun_mc_hello_sidecar_step_serving]
        # B) Sidecar container "hello" listening at PORT 8888,
        # which can only be accessed by serving ingress container
        # [START cloudrun_mc_hello_sidecar_step_sidecar]
        - image: us-docker.pkg.dev/cloudrun/container/hello
          name: hello
          env:
            - name: PORT
              value: "8888"
          resources:
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe:
            timeoutSeconds: 240
            periodSeconds: 240
            failureThreshold: 1
            tcpSocket:
              port: 8888
        # [END cloudrun_mc_hello_sidecar_step_sidecar]
```

After this we define the secrets. Don't forget to give the service account running your container access to the secrets.

```yaml
      # Named volume
      # [START cloudrun_mc_hello_sidecar_step_secret]
      volumes:
        - name: traefik-conf-static
          secret:
            secretName: traefik_static_config
            items:
              - key: latest
                path: traefik.yaml
        - name: traefik-conf-dynamic
          secret:
            secretName: traefik_dynamic_config
            items:
              - key: latest
                path: dynamic.yaml
      # [END cloudrun_mc_hello_sidecar_step_secret]
```

Then we deploy the configuration to gcloud

```bash
gcloud run services replace service.yaml --region europe-west1
```

After that we access the url given

![hello](/img/2024-11-21-hello.png)

If you want your cloud run to be public without authentication

```bash
gcloud run services add-iam-policy-binding treafik-multicontainer \
  --member="allUsers" \
  --role="roles/run.invoker"
```

See the full config below

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
# [START cloudrun_mc_hello_sidecar_step_metadata]
metadata:
  name: "traefik-multicontainer"
  labels:
    cloud.googleapis.com/location: "REGION"
  annotations:
    # Required to use Cloud Run multi-containers (preview feature)
    run.googleapis.com/launch-stage: BETA
    run.googleapis.com/description: sample tutorial service
    # Externally available
    run.googleapis.com/ingress: all
# [END cloudrun_mc_hello_sidecar_step_metadata]
# [START cloudrun_mc_hello_sidecar_step_deps]
spec:
  template:
    metadata:
      annotations:
        # Defines container startup order within multi-container service.
        # Below requires hello container to spin up before traefik container,
        # which depends on the hello container.
        # https://cloud.google.com/run/docs/configuring/containers#container-ordering
        run.googleapis.com/container-dependencies: "{traefik: [hello]}"
    # [END cloudrun_mc_hello_sidecar_step_deps]
    # [START cloudrun_mc_hello_sidecar_step_serving]
    spec:
      containers:
        # A) Serving ingress container "traefik" listening at PORT 8080
        # Main entrypoint of multi-container service.
        # Any pings to this container will proxy over to hello container at PORT 8888.
        # https://cloud.google.com/run/docs/container-contract#port
        - image: traefik
          name: traefik
          ports:
            - name: http1
              containerPort: 8080
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
          # Referencing declared volume below,
          volumeMounts:
            - name: traefik-conf-static
              readOnly: true
              mountPath: /etc/traefik/
            - name: traefik-conf-dynamic
              readOnly: true
              mountPath: /etc/traefik_dynamic/
          startupProbe:
            timeoutSeconds: 240
            periodSeconds: 240
            failureThreshold: 1
            tcpSocket:
              port: 8080
        # [END cloudrun_mc_hello_sidecar_step_serving]
        # B) Sidecar container "hello" listening at PORT 8888,
        # which can only be accessed by serving ingress container
        # [START cloudrun_mc_hello_sidecar_step_sidecar]
        - image: us-docker.pkg.dev/cloudrun/container/hello
          name: hello
          env:
            - name: PORT
              value: "8888"
          resources:
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe:
            timeoutSeconds: 240
            periodSeconds: 240
            failureThreshold: 1
            tcpSocket:
              port: 8888
        # [END cloudrun_mc_hello_sidecar_step_sidecar]
      # Named volume
      # [START cloudrun_mc_hello_sidecar_step_secret]
      volumes:
        - name: traefik-conf-static
          secret:
            secretName: traefik_static_config
            items:
              - key: latest
                path: traefik.yaml
        - name: traefik-conf-dynamic
          secret:
            secretName: traefik_dynamic_config
            items:
              - key: latest
                path: dynamic.yaml
      # [END cloudrun_mc_hello_sidecar_step_secret]
```
