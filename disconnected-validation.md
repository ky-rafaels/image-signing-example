# Validating Signature Using self-managed Rekor & Fulcio instance

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

<!-- Next lets get Keycloak setup

```bash
export KEYCLOAK_ENDPOINT=$(kubectl -n keycloak get service keycloak -o jsonpath='{.status.loadBalancer.ingress[0].*}')

export KEYCLOAK_URL=http://${KEYCLOAK_ENDPOINT}:8080/auth
``` -->

<!-- Next run a script to setup the sigstore realm for your fulcio client

```bash
./scripts/token.sh
``` -->

## Setup Rekor

```bash
helm upgrade --install rekor -n rekor-system sigstore/rekor \
--create-namespace \
--version 1.6.8 \
--values ./k8s/helm/rekor-values.yaml
```

## Setup Fulcio

Deploy fulcio using helm chart

```bash
helm upgrade --install fulcio -n fulcio-system sigstore/fulcio \
--create-namespace \
--values ./k8s/helm/fulcio-values.yaml
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
EOF
```
## Sign an artifact 

Using go-discover image

```bash
cosign sign --fulcio-url http://fulcio.example.com --oidc-issuer http://keycloak.example.com/realms/sigstore --rekor-url http://rekor.example.com ttl.sh/go-discovery:1h 
```

You should then be prompted with this below and redirected to keycloak for login via browser

```bash
Generating ephemeral keys...
Retrieving signed certificate...

	The sigstore service, hosted by sigstore a Series of LF Projects, LLC, is provided pursuant to the Hosted Project Tools Terms of Use, available at https://lfprojects.org/policies/hosted-project-tools-terms-of-use/.
	Note that if your submission includes personal data associated with this signed artifact, it will be part of an immutable record.
	This may include the email address associated with the account with which you authenticate your contractual Agreement.
	This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later, and is subject to the Immutable Record notice at https://lfprojects.org/policies/hosted-project-tools-immutable-records/.

By typing 'y', you attest that (1) you are not submitting the personal data of any other person; and (2) you understand and agree to the statement and the Agreement terms at the URLs listed above.
Are you sure you would like to continue? [y/N]
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
        - issuer: "http://keycloak.example.com
          subjectRegExp: ".*@example.com" 
      ctlog:
        url: http://rekor.example.com
EOF
```