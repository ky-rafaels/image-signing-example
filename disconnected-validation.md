# Validating Signature Using self-managed Rekor & Fulcio instance

First lets add the sigstore helm repo that we will use to install the sigstore stack

```bash
helm repo add sigstore https://sigstore.github.io/helm-charts
helm repo update
```

## Setup keycloak as OIDC provider

Deploy keycloak and import sigstore realm 

```bash
kubectl create ns keycloak && kubectl apply -f k8s/realm-cm.yaml
kubectl apply -f k8s/keycloak.yaml

# Save IP of loadbalancer svc for keycloak frontend
export KEYCLOAK_IP=$(kubectl get svc -n keycloak keycloak -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

<!-- Deploy keycloak using Bitnami helm chart with Chainguard images

```bash
helm upgrade --install keycloak -n keycloak bitnami/keycloak \
--create-namespace \
--values ./helm/bitnami-keycloak/keycloak-values.yaml
``` -->

<!-- kubectl create secret generic oidc-client-secret --from-literal=clientSecret=iHTF2wUHCLtXflgnGXwD4ZI3tAkEIHnM -n fulcio-system -->


## Setup Sigstore Stack 

First deploy the sigstore stack using the scaffold helm chart

```bash
helm upgrade --install sigstore sigstore/scaffold \
--values k8s/helm/scaffold-values.yaml \
--create-namespace \
-n sigstore 
 ```

Ensure that all components have been deployed

```bash
kubectl rollout status deployment scaffold-tuf-tuf -n tuf-system
```

Initialize cosign

```bash
cosign initialize --root http://tuf.example.com/root.json --mirror http://tuf.example.com
```

## Deploy Ingress and Setup DNS resolution

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

# Find ExternalIP of Ingress 
INGRESS=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

cat <<EOF | sudo tee -a /etc/hosts
${KEYCLOAK_IP} keycloak.example.com
${INGRESS} rekor.example.com
${INGRESS} fulcio.example.com
${INGRESS} tuf.example.com
EOF
```

### Install Docker Mac Net Connect for connecting to exposed services in your KiND cluster

Repo can be found [here](https://github.com/chipmk/docker-mac-net-connect)
```bash
brew install chipmk/tap/docker-mac-net-connect

sudo brew services start chipmk/tap/docker-mac-net-connect

# Run in debug mode 
sudo docker-mac-net-connect 
```

## Sign an artifact 

Using go-discover image

```bash
cosign sign --fulcio-url http://fulcio.example.com \
--oidc-issuer http://keycloak.example.com/realms/sigstore \
--rekor-url http://rekor.example.com \
--oidc-client-secret-file=oidc-client-secret.txt \
--allow-insecure-registry \
ttl.sh/go-discovery:1h
```

If successful you should see the following output

```bash
Generating ephemeral keys...
Retrieving signed certificate...

	The sigstore service, hosted by sigstore a Series of LF Projects, LLC, is provided pursuant to the Hosted Project Tools Terms of Use, available at https://lfprojects.org/policies/hosted-project-tools-terms-of-use/.
	Note that if your submission includes personal data associated with this signed artifact, it will be part of an immutable record.
	This may include the email address associated with the account with which you authenticate your contractual Agreement.
	This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later, and is subject to the Immutable Record notice at https://lfprojects.org/policies/hosted-project-tools-immutable-records/.

By typing 'y', you attest that (1) you are not submitting the personal data of any other person; and (2) you understand and agree to the statement and the Agreement terms at the URLs listed above.
Are you sure you would like to continue? [y/N] y
Your browser will now be opened to:
http://keycloak.example.com/realms/sigstore/protocol/openid-connect/auth?access_type=online&client_id=sigstore&code_challenge=h-fV6KBAd6BBYBFlkSinXjpX9Uh2i9w63yYcfcfEFcY&code_challenge_method=S256&nonce=2vHKMBAQavJ5NCtq2LINMqLkx8o&redirect_uri=http%3A%2F%2Flocalhost%3A58276%2Fauth%2Fcallback&response_type=code&scope=openid+email&state=2vHKM7X3dF4BUKDcDE3IFhfYV6d
Successfully verified SCT...
WARNING: Image reference ttl.sh/go-discovery:1h uses a tag, not a digest, to identify the image to sign.
    This can lead you to sign a different image than the intended one. Please use a
    digest (example.com/ubuntu@sha256:abc123...) rather than tag
    (example.com/ubuntu:latest) for the input to cosign. The ability to refer to
    images by tag will be removed in a future release.

tlog entry created with index: 0
Pushing signature to: ttl.sh/go-discovery
```

## Apply Cluster Image Policy custom resource

```bash
cat << EOF > custom-key-validation.yaml
---
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: disconnected-key-validation
spec:
  images:
    - glob: cgr.dev/ky-rafaels.example.com/**
  authorities:
    - keyless:
        url: http://fulcio.example.com
        identities:
        - issuer: "http://keycloak.example.com/realms/sigstore"
          subjectRegExp: ".*@chainguard.dev" 
      ctlog:
        url: http://rekor.example.com
EOF
```