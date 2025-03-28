# Validating Signature Using self-managed Rekor & Fulcio instance

## Setup keycloak as OIDC provider

Deploy keycloak using Bitnami helm chart with Chainguard images

```bash
helm upgrade --install keycloak -n keycloak bitnami/keycloak \
--create-namespace \
--values ./helm/keycloak-values.yaml
```

Next lets get Keycloak setup

```bash
export KEYCLOAK_ENDPOINT=$(kubectl -n keycloak get service keycloak -o jsonpath='{.status.loadBalancer.ingress[0].*}')

export KEYCLOAK_URL=http://${KEYCLOAK_ENDPOINT}:8080/auth
```

Next run a script to setup the sigstore realm for your fulcio client

```bash
./scripts/token.sh
```

## Setup Rekor

```bash
helm upgrade --install rekor -n rekor-system sigstore/rekor \
--create-namespace \
--values ./helm/rekor-values.yaml
```

## Setup Fulcio

Deploy fulcio using helm chart

```bash
helm upgrade --install fulcio -n fulcio-system sigstore/fulcio \
--create-namespace \
--set server.args.oidcClientSecret=
--values ./helm/fulcio-values.yaml
```

## Deploy Ingress and Setup DNS resolution

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

# Find ExternalIP of Ingress 
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
172.18.102.1  # returned

cat <<EOF | sudo tee -a /etc/hosts
172.18.102.1 keycloak.ky-rafaels.example.com
172.18.102.1 rekor.ky-rafaels.example.com
172.18.102.1 fulcio.ky-rafaels.example.com
EOF
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
        url: http://fulcio.ky-rafaels.example.com
        identities:
        - issuer: "http://keycloak.ky-rafaels.example.com
          subjectRegExp: ".*@example.com" # Adjust as needed
      ctlog:
        url: http://rekor.rekor-system.svc.cluster.local
EOF
```