# Social App Cloud Infra, IAC and CICD

<p style="font-weight: bold">Feb 7, 2024 ~ Present (Nov 19), 2024</p>

## Result

This is a post on Social App Cloud Infra, IAC and CICD we use for our social app.

Below are two infra graphs made by our CTO - Jonas Chen.

##### Infra 2024
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/infra_current.png"></img>

##### Infra 2025
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/infra_2025.png"></img>

## Tech Stack

1) [GCP Autopilot K8S](#gcp-autopilot-k8s)
2) [IAC](#iac)
3) [CDN](#cdn)
4) [CICD](#cicd)
5) [In Cluster Caching](#in-cluster-caching)
6) [Rabbitmq or PubSub](#rabbitmq-or-pubsub)
7) [Serverless](#serverless)

## GCP Autopilot K8S

As a small team of devs, we backend devs often are required to handle our own infra in the early days, this often leads to unmanaged resource provisioning and removal, not to mention adjusting resource according to actual traffics.

##### growing bill (the last month is ongoing so not the highest)
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/gcp_cost_trend.png"></img>

For example

- how many nodes in the cluster? 
- how many pods per node?
- node hardware spec
- node version upgrades
- k8s cluster version upgrades
- ...

By carefully configuring these parameters, it is likely we would achieve a better cost-resource balance, however the business of our company is not at a scale to afford a headcount to handle this.

The adoption of GCP Autopilot cluster help us configure these parameters automatically and save our budget, however this would require a full migration of all our services.

Normally, iac is introduced best at start, we think service migration is the second best timing for introducing it.

Luckily, we found a terraform script for GCP Autopilot cluster.

<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/autopilot_terra.png"></img>

After some configuration and updates to the helm charts, we now have over 10 clusters ready for deployment.

## IAC

IAC, if for nothing else, provide as a doc for infra itself, so we decide to go in this path.

During the migration, we discover a whole lot of deprecated services and remove them all, here we achieve the first cost down. (although being a cheap one, this happens quite often without IAC)

Next, during the upgrade, we found our deployment scripts are failing to provision new services due to ```horizontal scaling error```, which is caused by no longer supported k8s api version used in our chart.

After more investigation, we found that although much of our service is deployed successfully to standard clusters, they never scaled correctly since a service can be provisioned without ```horizontal scaler```.

##### workload can't scale without a working scaler
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/failed_horizon.png"></img>

Without a working scaler, whatever the current pods are specified on service provisioning would be the one lasted forever.

##### Correct Scaler Example
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ...
  labels:
    app: ...
spec:
  scaleTargetRef:
    ### This should match deploy version below for scaling to work correctly ###
    apiVersion: apps/v1
    kind: Deployment
    name: apen
  minReplicas: {{ .Values.scaling.minReplicas }}
  maxReplicas: {{ .Values.scaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.scaling.cpuAverageUtilization }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.scaling.memoryAverageUtilization }}
```

##### Deploy Chart Example
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ...
  labels:
    app: ...
```

After fixing it, we save ourself ~60% of Kubernetes Engine fee. (Sounds really dumb but we really only have a handful of dev back then.)

With the introduction of IAC we now also are able to provision/destroy Dev and Staging clusters much more easily.

## CDN

The introduction of CDN mainly comes from an event we met before, which causes our networking fee to tripled for two months.

CDN not only allow faster access to media content, it also protects the storage endpoints from attackers, since its fee is much lower in the case of heavy request to the same content.

We configure it for 5 of our cloud storages and restore networking fee to its normal level, as could be seen 

##### networking fee (October~December 2023)
<img style="
  display: block;
  margin-left: auto;
  margin-right: auto;
  margin-top: 32px;
  margin-bottom: 32px;
  border-radius: 12px;
" src="./img/gcp_cost_trend.png"></img>

## CICD

For CD, with GCP ```Workload Identity Federation```, we can authorize github action to directly deploy to GCP clusters.

Since we adopt a microservice architecture, this has saved us a whole lot of time!

For CI, we use golang mockery to generate dependency mockings.

##### part of our workflow
```
github-production:
	gcloud auth configure-docker
	docker push ${IMAGE}
	docker push asia.gcr.io/${PROJECT_ID}/${SERVICE_NAME}:latest
	gcloud components install kubectl
	gcloud container clusters get-credentials apen-dev --region ${REGION} --project ${PROJECT_ID}
	kubectl rollout restart deployment/${DEPLOY_NAME}
	kubectl rollout status deployment/${DEPLOY_NAME}

mocks:
	./mockery --all --inpackage

unit-test:
	go test -v -cover -short ./...

integration-test:
	go test -v -cover -run Integration ./...

# Add iam roles to allow managing k8s cluster
auth-github-to-gcp:
	gcloud iam workload-identity-pools create "github" \
	--project="${PROJECT_ID}" \
	--location="global" \
	--display-name="GitHub Actions Pool"
	gcloud iam workload-identity-pools providers create-oidc ...
	gcloud iam service-accounts add-iam-policy-binding ...

full-test:
	go test -v -cover ./...
```

Since integration tests are hard to maintain, we leave it out for now and consider this better handled with QA team.

## In Cluster Caching

GCP provides caching on the project level, which could be accessed across clusters, it is however, quite expensive.

The main drawback of using in cluster caching as far as we have concerned have been the lack of monitoring dashboards provided by GCP.

So we adopt a mixed solution, for most services we direct them to use in cluster caching, for performance critical services we direct them to use GCP caching solution for better monitoring.

This save us ~75% caching fee.

## Rabbitmq or PubSub

The comparison of RabbitMQ and GCP PubSub is quite similar to caching, where the former is much cheaper but lack monitoring dashboards, we also endup a mixed solution at the end.

This also saves us ~60% fee for message queue cost.

## Serverless

Some of the services don't always need to be alive, like video compression or broadcasting messages.

Instead of containerizing those service and make them a workload in Kubernetes Engine, we could instead deploy them as serverless services using services like ```GCP cloud run```.

##### Example Dockerfile for Serverless
```
# Get started with a build env with Rust nightly
FROM rustlang/rust:nightly-bullseye as builder

# If you’re using stable, use this instead
# FROM rust:1.74-bullseye as builder

# Install cargo-binstall, which makes it easier to install other
# cargo extensions like cargo-leptos
RUN wget https://github.com/cargo-bins/cargo-binstall/releases/latest/download/cargo-binstall-x86_64-unknown-linux-musl.tgz
RUN tar -xvf cargo-binstall-x86_64-unknown-linux-musl.tgz
RUN cp cargo-binstall /usr/local/cargo/bin

# Install cargo-leptos
# RUN cargo binstall cargo-leptos -y
RUN cargo install --locked cargo-leptos

# Add the WASM target
RUN rustup target add wasm32-unknown-unknown

# Make an /app dir, which everything will eventually live in
RUN mkdir -p /app
WORKDIR /app
COPY . .

# Build the app
RUN cargo leptos build --release -vv

FROM debian:bookworm-slim as runtime
WORKDIR /app
RUN apt-get update -y \
  && apt-get install -y --no-install-recommends openssl ca-certificates \
  && apt-get autoremove -y \
  && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/*

# -- NB: update binary name from "admin" to match your app name in Cargo.toml --
# Copy the server binary to the /app directory
COPY --from=builder /app/target/release/admin /app/

# /target/site contains our JS/WASM/CSS, etc.
COPY --from=builder /app/site /app/site

# Copy Cargo.toml if it’s needed at runtime
COPY --from=builder /app/Cargo.toml /app/

# Set any required env variables and
ENV RUST_LOG="info"
ENV LEPTOS_SITE_ADDR="0.0.0.0:8080"
ENV LEPTOS_SITE_ROOT="site"
ENV DATABASE_URL = "postgres://postgres:Fuchiisawesome!%40%23@127.0.0.1/ads"
EXPOSE 8080

# -- NB: update binary name from "admin" to match your app name in Cargo.toml --
# Run the server
CMD ["/app/admin"]
```

##### Example Deploy Workflow (Manual Revision)
```
image:
	docker build -t ${TAG} .
	docker tag ${TAG} ${IMAGE}
	docker tag ${TAG} asia.gcr.io/${PROJECT_ID}/${SERVICE_NAME}:latest
	docker push asia.gcr.io/${PROJECT_ID}/${SERVICE_NAME}:latest
```

After moving them out of clusters, we save another 10% on Kubernetes Engine fees for ourselve.
