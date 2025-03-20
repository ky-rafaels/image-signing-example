# image-signing-example

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

DIGEST=$(docker inspect localhost:5000/go-discovery:v1 |jq -c 'first'| jq .RepoDigests | jq -c 'first' | tr -d '"')

cosign attest --type spdxjson \
 --predicate go-discovery.spdx.json \
 $DIGEST
```
