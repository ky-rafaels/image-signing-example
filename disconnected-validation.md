# Validating Signature Using self-managed Rekor & Fulcio instance

Deploy keycloak to use as OIDC provider

```bash
helm upgrade --install keycloak -n keycloak bitnami/keycloak \
--create-namespace \
--values ./helm/keycloak-values.yaml
```

Deploy rekor using helm chart 

```bash
helm upgrade --install rekor -n rekor-system sigstore/rekor \
--create-namespace \
--values ./helm/rekor-values.yaml
```

Deploy fulcio using helm chart

```bash
helm upgrade --install fulcio -n fulcio-system sigstore/fulcio \
--create-namespace \
--values ./helm/fulcio-values.yaml
```

Apply Cluster Image Policy custom resource

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
        url: http://fulcio.fulcio-system.svc.cluster.local
        identities:
        - issuer: "http://keycloak.keycloak.svc.cluster.local
          subjectRegExp: ".*@example.com" # Adjust as needed
      ctlog:
        url: http://rekor.rekor-system.svc.cluster.local
EOF
```