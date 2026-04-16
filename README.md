# OpenClaw On OpenChoreo with Agent Sandbox

This is a sample config for running [OpenClaw](https://openclaw.ai/) agents on [OpenChoreo](https://openchoreo.dev) with [Agent Sandbox](https://agent-sandbox.sigs.k8s.io/).

This guide assumes you have OpenChoreo running locally with the default configuration as outlined in https://openchoreo.dev/docs/getting-started/try-it-out/on-k3d-locally/ - otherwise you will need to modify the resources in this repository to match your tooling - it's also configured to use an API key from https://platform.openai.com (OpenClaw supports other model providers, but you will need to modify the configuration in this repo).

## 1. Install Agent-Sandbox CRDs and controllers on the data plane cluster(s)

```shell
# https://github.com/kubernetes-sigs/agent-sandbox/releases
export VERSION="v0.3.10"
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/$VERSION/manifest.yaml
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/$VERSION/extensions.yaml
```

Reference: https://agent-sandbox.sigs.k8s.io/docs/overview/#installation

## 2. Add the secrets to the secret store (OpenBao) and create the OpenChoreo SecretReferences

```shell
export OPENCLAW_GATEWAY_TOKEN="$(openssl rand -hex 32)"
echo "OpenClaw Gateway Token: ${OPENCLAW_GATEWAY_TOKEN}" # You'll need this to sign in later

kubectl exec -n openbao openbao-0 -- sh -c "
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv put secret/openai-api-key value='<YOUR_OPENAI_API_KEY>'
  bao kv put secret/openclaw-gateway-token value='${OPENCLAW_GATEWAY_TOKEN}'
"
```

```sh
cat <<'EOF' | kubectl apply -f -
apiVersion: openchoreo.dev/v1alpha1
kind: SecretReference
metadata:
  name: openai-api-key
  namespace: default
spec:
  refreshInterval: 1h
  template:
    type: Opaque
  data:
    - secretKey: value
      remoteRef:
        key: openai-api-key
        property: value
---
apiVersion: openchoreo.dev/v1alpha1
kind: SecretReference
metadata:
  name: openclaw-gateway-token
  namespace: default
spec:
  refreshInterval: 1h
  template:
    type: Opaque
  data:
    - secretKey: value
      remoteRef:
        key: openclaw-gateway-token
        property: value
EOF
```

## 2. Apply OpenChoreo Resources

> Alternatively, you can use `occ apply -f <filename>`

This will create a Project called "openclaw-agents", a custom ClusterComponentType "openclaw-sandbox-agent" and a new Component to try it out.

```sh
kubectl create -f project.yaml
kubectl create -f openclaw-sandbox-cct/openchoreo/cluster-component-type-openclaw-agent-sandbox.yaml
kubectl create -f openclaw-sandbox-cct/openchoreo/component-openclaw-agent-sandbox.yaml
kubectl create -f openclaw-sandbox-cct/workload.yaml
```

## 4. Log in to create a pairing request to OpenClaw

Open the external gateway URL shown on the OpenChoreo Portal (via the Component > Deploy tab > View Endpoint URLs) and log in using the `OPENCLAW_GATEWAY_TOKEN` we created earlier.

```sh
echo $OPENCLAW_GATEWAY_TOKEN
```

You'll get an error saying "Pairing required" when you click "Connect" - this is expected.

## 5. Approve Browser pairing request for OpenClaw

Get the pairing request ID from the OpenClaw pod created by the agent-sandbox by running:

```sh
NS=$(kubectl get pod -A -l openchoreo.dev/component=openclaw-sandbox-agent -o jsonpath='{.items[0].metadata.namespace}')
POD=$(kubectl get pod -n "$NS" -l openchoreo.dev/component=openclaw-sandbox-agent -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n "$NS" "$POD" -- node /app/dist/index.js devices list
```

You'll see a table with pairing requests - copy the request ID (this is a UUID) and run the following command:

```sh
kubectl exec -n "$NS" "$POD" -- node /app/dist/index.js devices approve <requestId>
```

## 6. Log into OpenClaw

Once the pairing is approved, you can log into OpenClaw Control UI and start using it.


---

> **Note:** There's another directory in this repository called `/openclaw-cct` which provides a custom component type for OpenClaw using default K8s primitives like with a standard deployment and PVC - without the abstractions provided by agent-sandbox. You technically do not need a custom component type (CCT) for this, even the default `deployment/web-app` CCT should suffice.

**If you enjoyed using OpenChoreo, consider giving the project a star: https://github.com/openchoreo/openchoreo**
