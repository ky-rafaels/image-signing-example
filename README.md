# image-signing-example

## Dependencies
- docker
- kubectl
- crane
- cosign
<!-- - syft -->

## Setup 

For this example we are going to use a kubernetes discovery service as our sample app. This particular app intends to collect the image types of workload in your cluster and emit the findings as prometheus metrics.

First clone the project [here](git@github.com:ky-rafaels/kind-cluster.git) and follow instructions for setting up a 3 node kind cluster with a local docker registry we can use for our example.

### Deploy sample image 

```bash
git clone git@github.com:ky-rafaels/k8s-image-type-discovery.git
cd k8s-image-type-discovery
docker build --provenance=true --sbom=true --push --tag ttl.sh/go-discovery:1h .

# Optionally push to local registry deployed from kind-cluster deployment steps
# docker build --provenance=true --sbom=true --push --tag localhost:5000/go-discovery:v1 .
```

Create namespace and label to enable validation of signed images

```bash
kubectl create ns go-discover && kubectl label ns go-discover policy.sigstore.dev/include="true"
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

# Validating Signature Using Public Rekor

```bash
cat << EOF >> keyless-rekor-public.yaml
---
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: chainguard-images-are-signed
  annotations:
    catalog.chainguard.dev/title: Chainguard Images
    catalog.chainguard.dev/description: Enforce Chainguard images are signed
    catalog.chainguard.dev/labels: chainguard
spec:
  images:
    - glob: cgr.dev/ky-rafaels.example.com/**
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subject: https://github.com/chainguard-images/images/.github/workflows/release.yaml@refs/heads/main
      ctlog:
        url: https://rekor.sigstore.dev
EOF
```

# Validating Image Signatures with Custom Key

Create a custom local key and sign an image

```bash
❯ cosign generate-key-pair
Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub

docker build --provenance=true --sbom=true --push --tag ttl.sh/go-discovery:1h .

IMAGE=ttl.sh/go-discovery:1h@sha256:5d449cce70519f1a6f269a183c298712ac55131a14e48bbe2077f8cff02bf112

# remove signature and resign
cosign clean $IMAGE

# sign image with key
cosign sign --key cosign.key $IMAGE
```

Apply a clusterImagePolicy to sigstore controller to validate signatures using the new custom key 

```yaml
---
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: custom-key-attestation-sbom-spdxjson
spec:
  images:
  - glob: "ttl.sh/*"
  authorities:
  - name: custom-key
    key:
      data: |
        -----BEGIN PUBLIC KEY-----
        MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEDWIh6o7q+081RuGUqYVV+DyLwTNV
        1D4AbR7QMgejYT5zffY4asCkW9KYl7ZVRuic8IEUQ8PzXo3kLE++KP34/w==
        -----END PUBLIC KEY-----
    ctlog:
      url: https://rekor.sigstore.dev
    attestations:
    - name: must-have-spdxjson
      predicateType: spdxjson
      policy:
        type: cue
        data: |
          predicateType: "https://spdx.dev/Document"
```