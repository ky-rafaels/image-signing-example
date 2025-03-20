# image-signing-example


## Build new image

```bash
docker build --provenance=true --sbom=true --push --tag IMAGE .
```

## Create a Vex document for packages not affected

```bash
❯ vexctl create \
∙ --author="kyle.rafaels@caci.com" \
∙ --product="pkg:io.quarkus.http:quarkus-http-core@5.3.3" \
∙ --vuln="CVE-2024-12397" \
∙ --status="not_affected" \
∙ --justification="vulnerable_code_not_in_execute_path" \
∙ --file="CVE-2024-12397.vex.json"
 > VEX document written to CVE-2024-12397.vex.json
 ```

## Create an SBOM and Attest to Image

```bash
syft IMAGE:TAG -o spdx-json > image.spdx.json

cosign clean IMAGE:TAG

DIGEST=$(docker inspect IMAGE:TAG |jq -c 'first'| jq .RepoDigests | jq -c 'first' | tr -d '"')

cosign attest --type spdxjson \
 --predicate example-image.spdx.json \
 $DIGEST
```
