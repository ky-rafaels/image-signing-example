# image-signing-example

## Dependencies
- docker
- kubectl
- crane
- syft
- cosign

## Build an image

For this example we are going to use a kubernetes discovery service as our sample app. This particular app intends to collect the image types of workload in your cluster and emit the findings as prometheus metrics.

First clone the project [here](git@github.com:ky-rafaels/kind-cluster.git) and follow instructions for setting up a 3 node kind cluster with a local docker registry we can use for our example.

### Clone repo and build image

```bash
git clone git@github.com:ky-rafaels/k8s-image-type-discovery.git
docker build --provenance=true --sbom=true --push --tag localhost:5000/go-discovery:v1 .
```

## Create an SBOM and Attest to Image

```bash
syft localhost:5000/go-discovery:v1 -o spdx-json > go-discovery.spdx.json

# OPTIONALLY remove any previous signatures
# cosign clean IMAGE:TAG

DIGEST=$(crane digest localhost:5000/go-discovery:v1)

cosign attest --type spdxjson \
 --predicate go-discovery.spdx.json \
 $DIGEST
```

## Create a CA bundle

Install cert-manager and trust manager using the steps [here](https://github.com/ky-rafaels/java-certs-with-containers.git) or create ca bundle manually.

If trust-manager and cert-manager are installed you can apply this bundle directly
```bash
k apply -f ca-certs/bundle.yaml
```

## Install and setup sigstore policy controller

```bash
helm repo add sigstore https://sigstore.github.io/helm-charts
helm repo update

# Create and label namespace to populate with cert bundle from trust manager
kubectl create ns cosign-system && kubectl label ns cosign-system create-certs="true"

# Validate bundle exists
kubectl get cm chainguard-dev

# Deploy sigstore policy controller
helm upgrade --install policy-controller -n cosign-system sigstore/policy-controller \
--set webhook.image.repository=cgr.dev/ky-rafaels.example.com/sigstore-policy-controller:0.12 \
--set webhook.image.version=sha256:14447007e2ebd457735f6303209424de0db6ad477e12a98c89b2bfceb1ac0026 \
--set webhook.registryCaBundle.name=chainguard.dev \
--set webhook.registryCaBundle.key=bundle.pem
```

