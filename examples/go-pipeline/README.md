# FRSCA Golang Tekton Pipeline

This is a sample GOlang tekton pipeline.

> :warning: This pipeline is not intended to be used in production

## M1 Mac Issue

The "golang-test" task fails to run on M1 Macs due to mis-match in architecture.
Setting the parameter of "GOARCH" to nothing resolves this and allows for GO to
auto determine the architecture.

## Starting Demo

```bash
# Only if a cluster is needed.
make setup-minikube

# Use the built-in registry, or replace with your own local registry
export REGISTRY=registry.registry

# Setup FRSCA environment
make setup-frsca

# if using the built-in registry, run the proxy in the background or another window
make registry-proxy >/dev/null &

# Run a new pipeline.
make example-golang-pipeline

# Wait until it completes.
tkn pr logs --last -f

# Export the value of IMAGE_URL from the last pipeline run and the associated taskrun name:
IMAGE_URL=$(tkn pr describe --last -o jsonpath='{..taskResults}' | jq -r '.[] | select(.name | match("IMAGE_URL$")) | .value')
TASK_RUN=$(tkn pr describe --last -o json | jq -r '.status.taskRuns | keys[] as $k | {"k": $k, "v": .[$k]} | select(.v.status.taskResults[]?.name | match("IMAGE_URL$")) | .k')
if [ "${REGISTRY}" = "registry.registry" ]; then
  : "${REGISTRY_PORT:=5000}"
  IMAGE_URL="$(echo "${IMAGE_URL}" | sed 's#'${REGISTRY}'#127.0.0.1:'${REGISTRY_PORT}'#')"
fi

# Double check that the attestation and the signature were uploaded to the OCI.
crane ls "$(echo -n ${IMAGE_URL} | sed 's|:[^/]*$||')"

# Verify the image and the attestation.
cosign verify --insecure-ignore-tlog --key k8s://tekton-chains/signing-secrets "${IMAGE_URL}"
cosign verify-attestation --insecure-ignore-tlog --type slsaprovenance --key k8s://tekton-chains/signing-secrets "${IMAGE_URL}"

# Verify the signature and attestation with tkn.
# These commands do not work with the built in registry.
tkn chain signature "${TASK_RUN}"
tkn chain payload "${TASK_RUN}"

# if the registry proxy is running in the background, it can be stopped
kill %?registry-proxy
```
